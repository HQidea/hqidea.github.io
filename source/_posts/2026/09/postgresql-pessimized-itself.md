---
title: 什么？PostgreSQL 把自己给负优化了
date: 2026-09-09 17:26:40
tags:
---

这是一篇在AI时代手写的古法博客。

## 背景

起因是某客户环境在用户批量导入2w条数据后，离线任务出现了堆积。

该环境近期没有变更，一般分析来看，要么命中了代码里的某个bug，要么infra出现了问题。

结果用ai分析之后，发现既不是代码中的bug也不是infra故障，而是pg执行计划把自己给负优化了🤡（好吧，多少也算一点infra的问题。。。）

离线任务的执行是通过后台worker轮询数据库，抢占待执行的任务，抢占成功的worker就可以执行相应任务，这也算是一种简单的分布式任务执行方式。

## 问题分析

抢占任务的sql大致如下：

```
WITH task_to_run AS (
    SELECT t."id"
    FROM "task" t
    JOIN "workflow" wf ON wf.id = t.wf_id
    WHERE
        -- wheres
        AND t."status" = 'pending'
    ORDER BY wf."created_at" ASC, t."next_run_at" ASC
    LIMIT 1
    FOR UPDATE OF t SKIP LOCKED
)
UPDATE "task"
SET
    -- sets
    "status" = 'running',
FROM task_to_run
WHERE
    "task"."id" = task_to_run."id"
```

其中`workflow`表167w行数据，`task`表175w行数据。一般情况下，处于`pending`状态的task不会很多，所以pg会先查询`task`表，取出所有`pending`状态的task，针对每一个task，使用`wf_id`去`workflow`表中查询具体数据，取到`created_at`字段，从小到大排序，最终返回结果。

但是当用户批量导入2w条数据后，每条数据会创建2个workflow，每个workflow至少有一个task，因此这时候表里面有大概4w个`pending`状态的task。因为表中数据变化很大，pg触发了一次autoanalyze，autoanalyze算了一笔账，如果还是按之前的方式先查询`task`表中的数据，要查询4w次才能取到结果数据。

换个角度，4w条`pending`状态的task对应4w条workflow，假设这些数据是均匀分布的，而且正好sql只取`created_at`最小的`pending`数据，如果沿着`created_at`索引先查询`workflow`表，大概只需要取40多条数据，就能找到结果。

40和4w对比，pg选择了40。看起来pg选择了代价更小的执行方式，但是实际执行的时候却花费了比查询4w条数据更大的代价，这条sql的执行时间从0.2s上升到了17s。原因就是4w条`pending`状态的task并不是均匀分布的，为了找到`created_at`最小的`pending`数据，pg至少读了165w条数据，而不是推测的40多条。

问题找到后，解决办法就相对明确，让pg先读`task`表，再读`workflow`表。

## 解决办法

办法一：删掉`workflow`表的`created_at`索引，让pg选不了错误的执行计划。

办法二：设置`enable_incremental_sort = off`

这条sql中有2个排序键，wf."created_at" 和 t."next_run_at"。负优化的执行计划用了`created_at`索引，但是这个索引只能保证第一个键有序，第二个键是`task`表上的，执行计划就得补一个排序，补排序有2种方式：

+ 普通排序：无视第一个排序键的顺序，把所有满足条件的数据全部取出来，重新排序

+ 增量排序（Incremental Sort）：在保留第一个排序键顺序的情况下取数据，判断排序键是否变了，变了之后把前一个排序键下的结果进行排序

关闭增量排序的目的就是让按`workflow`表`created_at`索引取数据的代价变大，让执行计划回退到先读`task`表。

办法三：把join改成子查询，
```
SELECT t."id"
FROM "task" t
WHERE
    -- wheres
    AND t."status" = 'pending'
    AND EXISTS (
        SELECT 1 FROM "workflow" wf
        WHERE wf.id = t.wf_id
    )
ORDER BY (SELECT wf."created_at" FROM "workflow" wf WHERE wf.id = t.wf_id) ASC, t."next_run_at" ASC
LIMIT 1
FOR UPDATE OF t SKIP LOCKED
```
缺点是针对`task`表中的每条候选数据，需要查2次`workflow`表，整体执行速度会慢1倍。

办法四：LATERAL JOIN

```
SELECT t."id"
FROM "task" t
CROSS JOIN LATERAL (
    SELECT wf."created_at"
    FROM "workflow" wf WHERE wf.id = t.wf_id
    LIMIT 1
) wf
WHERE
    -- wheres
    AND t."status" = 'pending'
ORDER BY wf."created_at" ASC, t."next_run_at" ASC
LIMIT 1
FOR UPDATE OF t SKIP LOCKED
```

LATERAL JOIN允许子查询使用FROM中其他表的字段，上面的sql中必须增加`LIMIT 1`，不然pg会把它改写成普通join。

## 后记

问题解决后，我们再来补个知识：

### 执行计划成本估算

成本是一个估算分数，pg对每一种候选做法算一个分数，选择分数低的做法执行。每个步骤有2个分数，起步成本和总成本：

+ 起步成本：交出第一行结果的分数

+ 总成本：交出全部结果的分数

#### 先读`task`表，再读`workflow`表的成本：

1. hash join cost=70119.69..191579.49
2. limit     cost=191775.79..191775.80
    ```
    // https://github.com/postgres/postgres/blob/REL_17_9/src/backend/optimizer/util/pathnode.c#L3921

    *total_cost = *startup_cost +
        (input_total_cost - input_startup_cost) * count_rows / input_rows;
    ```

#### 先读`workflow`表，再读`task`表的成本：

1. 嵌套循环 cost=0.85..1286401.50
2. 增量排序 cost=33.63..1288168.16
    ```
    // https://github.com/postgres/postgres/blob/REL_17_9/src/backend/optimizer/path/costsize.c#L2067

    group_input_run_cost = input_run_cost / input_groups;  // 这里假设了数据是均匀分布的

    startup_cost = group_startup_cost + input_startup_cost + group_input_run_cost;
    ```
3. limit   cost=33.63..66.45
