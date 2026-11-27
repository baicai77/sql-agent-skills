---
name: simple-direct-sql
description: >-
  SQL 查询（SELECT）的基础通用规范。只要任务涉及编写、生成、补全、改写、简化或审查 SELECT 查询
  （包括取数、统计、报表、指标口径、排查结果异常，以及嵌入在脚本、代码或数仓 ETL 中的查询逻辑），均必须使用此技能。
  不限方言（MySQL、PostgreSQL、Hive、Spark SQL、Presto/Trino、ClickHouse 等），所有上层 SQL/数仓加工技能均以此为基础。
  不单独覆盖建表 DDL、数据库运维权限、ORM 代码及纯索引/执行计划调优。
---

# Simple Direct SQL（通用 SQL 查询基础规范）

目标：用最短的阅读路径写出**结果正确**的 SQL。优先级：**正确 > 简洁 > 其他**。

判断标准只有一条：SQL 里的每张表、每个 `join`、每个条件、每个函数，都必须对应一条**明确的业务需求**或**已核实的数据事实**；对应不上的就删掉。

## 编写步骤

1. **定粒度与数据合同**：先确定结果一行代表什么（每用户 / 每天每渠道 / 每订单），明确各输入表的主键唯一性、可空性与版本语义，由此决定 `group by` 和 `join` 方式。
2. **选最少的表**：字段在哪张表就只查哪张表。只有在需要它的字段、需要保留它的全集（如"包括没下单的用户"）或需要用它过滤时，才关联维表。
3. **按「规则 1」组织查询层级与风格**：单表一步可完成时直接单层查；多表关联或分阶段加工时按规则 1 将单表处理内聚到内部子查询，统一小写与 `t1/m1` 别名。
4. **按「规则 2～4」精简结构**：去无必要 CTE、去无依据防御性代码、保持列与计算最小化。
5. **自检**：按「准确性清单」逐项核对语义与边界后再输出。

## 规则 1：编码风格、表别名与子查询内聚（权威定义）

- **关键字与标准函数小写**：
  - 所有 SQL 关键字、数据类型与标准内置函数（`select`、`from`、`left join`、`inner join`、`full outer join`、`on`、`where`、`group by`、`having`、`order by`、`union all`、`as`、`and`、`or`、`case when … then … else … end`、`sum`、`count`、`coalesce` 等）一律小写，不要大写。
  - **不改写大小写的对象**：字符串字面量（如 `'PAID'`）、日期格式与调度宏（如 `yyyy-MM-dd` 中的 `MM`）、区分大小写的库表/列标识符，以及大小写敏感方言的特有函数名（如 ClickHouse 的 `countIf`、`sumIf`、`argMax`）。
- **表别名固定规范**：
  - 同一可见作用域内按输入实例依次编号：指标表（事实表、业务明细表、指标聚合子查询，含事实表自关联）命名为 `t1`、`t2`、`t3`……；维度表（含同一维表按出发地/目的地等不同角色多次关联）命名为 `m1`、`m2`、`m3`……。
  - 某输入内部因分阶段处理需再嵌套过渡子查询（如无 `qualify` 方言先算 `rn` 再筛 `rn = 1`）时，内部过渡别名继承所属输入编号，命名为 `t1_0`、`t2_0`、`m1_0`、`m2_0`，对外参与关联的别名仍为 `t1/t2`、`m1/m2`。
  - `exists / not exists` 等相关子查询必须使用独立别名（如顺延编号或 `s1`），严禁遮蔽外层别名。
- **单表直查 vs 内部子查询内聚**：
  - **单表一步可完成时**：无多表关联，且无需窗口结果过滤或二次聚合时，直接一层 `select … from tbl t1 where … group by … having …` 完成，禁止套纯透传外壳。
  - **多表关联或单表分阶段加工时**：单表候选集的分区限制（如 `dt` 控制）、行过滤、字段转换及必要的去重、预聚合在该输入自己的内部子查询 `(select … from … where …) t1 / m1` 中完成，不要散落在外部；若某关联输入没有任何单表过滤、分区限制或字段加工（如无分区静态小维表直连），可直接关联原表 `join dim_x m1`，不必硬套空投影壳。
  - **谓词位置边界（保证语义等价）**：决定“单表候选行是否存在”的过滤下推至该表内部子查询；决定“两表能否匹配”而非“输入行是否存在”的谓词（如跨表有效期区间 `t1.biz_date >= m1.start_date and t1.biz_date < m1.end_date`，或“保留全部左表行、仅当左表满足某条件时才匹配右表”的条件）必须保留在外层 `on`，不得因下推导致需保留的行丢失。
  - **外层职责**：外层负责表间关联（`join … on …`）、跨表计算、**最终聚合（`group by / having`）**、反连接未命中过滤（`where m1.pk is null`）、其他依赖关联结果的跨表过滤及最终列输出。

## 规则 2：默认不用 CTE

- 默认不用 `with t as (...)` 拆步骤，也不要把单表简单查询机械拆成多层嵌套。
- 单层无法直接表达时，按「规则 1」用 `from / join` 中的内部子查询收敛单表或分阶段逻辑。
- 单表查询优先用方言原生能力减少嵌套：
  - 条件聚合：`sum(case when … then … end)`，PG/Trino 的 `count(*) filter (where …)`，ClickHouse 的 `countIf/sumIf`；
  - 取极值对应的值：Presto/Trino/Spark 的 `max_by`，ClickHouse 的 `argMax`；
  - 每组取首行 / Top N：支持 `qualify` 的方言单表直接用 `qualify`，ClickHouse 用 `limit n by`；PostgreSQL 每组取 **1 行**可用 `distinct on` 并指定确定性排序，严格取前 **N 行**用 `row_number()` 后筛 `rn <= n`（要求保留并列时另行按 `rank / dense_rank` 确定口径）。
- **例外**（仅以下情况可用 CTE）：
  - 用户明确要求使用 CTE，或团队规范要求；
  - 需要递归（`with recursive`）；
  - 同一中间结果在一条语句中被引用多次，且无法用窗口函数或条件聚合消除。

## 规则 3：不写防御性 SQL

一切清洗、去重与兜底都必须以**业务需求**或**已查实的数据事实**为依据；未说明或未证实存在脏数据时，不添加：

- 多余的空值/空串兜底：`coalesce / ifnull / nvl`、`nullif(col, '')`；
- 多余的除零保护：`nullif(x, 0)` 或 `case when x = 0 …`；
- 多余的格式与非空清洗：`is not null`、`!= ''`、`trim()`、`lower()`、`cast()`、`rlike`；
- 恒真条件：`where 1 = 1`；
- **防守性 `join` 与盲目去重**：关联或去重前先确认主键唯一性与关联命中率——维表关联键本身已唯一时不在条件中堆叠冗余层级/排除条件；单次关联已命中时不做二次兜底 `left join`；上游无真实重复时不盲加 `row_number()` 去重。

常见的纯冗余写法：

- `where t1.status = 'paid' and t1.status is not null`：等值条件本身已排除 `null`；
- `inner join` 的关联键再加 `is not null`：等值关联本身不会匹配 `null`。

**注意：以下场景直接决定结果语义正确性，不属于冗余防御，必须保留：**

| 场景 | 正确写法 |
| :-- | :-- |
| 需求要求无匹配数据显示为 0（`left join` 后 `sum`） | `coalesce(sum(x), 0)`；`count(col)` 本身返回 0，不用包 |
| `full outer join` 输出两侧合并后的实体主键/公共维度 | `coalesce(t1.entity_id, t2.entity_id)` |
| 整数相除求比率（PG、Presto/Trino、SQL Server 会截断成整数） | `sum(a) * 1.0 / sum(b)` 或显式 `cast` |
| 分母按业务可能为 0，且目标方言除零会报错（如 PG） | `nullif(分母, 0)`；分母恒 ≥ 1（如分组内 `count(*)`）时不加 |
| 反连接排除子集且右表列可能含 `null` | 用 `not exists`（注意左键本身可为 `null` 时按业务确认是否保留空键行） |

## 规则 4：输出最小化

- 只 `select` 下游或最终结果真正读取的列，不写 `select *`（窗口子查询中仅用于 `partition by / order by` 而外层不读取的列无需投影输出）。
- **列结构尽可能窄且尽可能全**：同一信息不重复占列，不保留同值/同义冗余列（如同义编码双存、同值不同类型双存、可由已有列直接拼出的冗余列），不额外增加无业务要求的加工元数据列。
- 不为"以防重复"加 `distinct`；统计去重实体数时按需用 `count(distinct …)`。
- 不写重复过滤条件，不写**不影响结果语义**的排序（如无 `limit` 子查询内的 `order by`、无需求的最终 `order by`；而 `order by … limit n`、PG `distinct on`、窗口函数内的 `order by` 决定结果，必须保留）。
- 常规查询中注释仅用于非显而易见的业务口径，一行说清（若上层落表模板规范另有全量字段注释要求，从其规定）。

## 准确性清单

- **粒度与可加性**：`select` 非聚合列与 `group by` 一致；关联前核对各输入粒度、关联键唯一性与指标可加性（如分组 `count(distinct user_id)` 不可跨组直接相加）。
- **防扇出**：主表关联一对多明细后再 `sum` 主表字段会重复累加，须先把多的一侧在内部子查询聚合到关联键粒度。
- **版本去重与筛选先后**：对存在多版本更新的数据去重时，先明确**版本统计截点**（截至业务日当天的截点状态，还是包含后续日期的事后追溯变更），再明确业务要的是**「实体最新状态满足条件」**还是**「满足条件的最新记录」**——前者必须先在完整候选版本上排序取 `rn = 1`，再筛可变状态与事件时间（防止已取消/冲正的新版本被提前筛掉而误取旧支付版本）；后者才先筛后排。若时间列可能并列，须补充**同一窗口内可区分真实先后**的稳定决胜键（如版本号、CDC 位点序号，勿用窗口分组键本身充当排序键）；分区裁剪不得排除截点内可能胜出的版本，增量/快照语义不明时先向用户确认。
- **关联类型与谓词位置**：按「规则 1」放置单表过滤与 `on` 匹配条件；反连接 `left join … where m1.pk is null` 必须检测匹配成功时保证非空的右侧键列，不可检测可空属性列。
- **集合合并（`union all` / `full outer join`）**：`union all` 各分支须按位置对齐列义与类型，仅当**最终去重键**在分支间严格互斥时，分支内去重才能替代合并后全局去重（不同 `dt` 分区不自动代表业务主键互斥）；`full outer join` 两侧须先在各自子查询达到目标粒度，外层用 `coalesce(t1.key, t2.key)` 输出统一实体键。
- **where 与 having**：聚合前过滤用 `where`，聚合结果过滤用 `having`。
- **时间边界与可裁剪性**：时间戳范围用左闭右开 `ts >= '2024-01-01 00:00:00' and ts < '2024-02-01 00:00:00'`；不在过滤列上套函数（如 `date(ts)`、`substr(ts, 1, 10)`），分区表须带分区键裁剪。
- **计数与并列语义**：区分 `count(*)`、`count(col)`（不计 `null`）和 `count(distinct col)`；Top N 按需求选 `row_number`（严格 N 条）或 `rank / dense_rank`（保留并列）。

## 输出约定

- 默认输出一个 SQL 代码块（若上层技能规定按独立 `.sql` 文件模板交付，以上层技能为准）。未指定方言时用标准 SQL；用到方言特有语法时一句话说明。
- 表结构、主键唯一性、版本语义或业务口径不明确时，若不同理解会导致结果明显不同，**先问用户**；轻微常规前提在 SQL 后用 1～3 条简短"假设"说明。
- **改写**：严格保证在已知数据合同下结果等价；若简化会改变任何边界输出（如空值、空白字符、孤儿键行为），必须明确指出差异。
- **审查**：先列正确性与口径风险，再列可简化项，最后给出改写后的 SQL。

## 示例

### 1. 在明确数据合同下等价简化（去 CTE、去冗余防御、单表直查）

已知数据合同：统计 `2024-01-01` 已支付订单每用户实付总额；`orders.user_id` 非空且均存在于 `users` 表，`status` 枚举值已规范为小写无空格，已支付订单 `amount` 非空。

```sql
-- Bad：在同一数据合同下滥用 CTE、大写关键字、多余防御与无必要关联
WITH base_orders AS (
    SELECT o.user_id, COALESCE(o.amount, 0) AS amount
    FROM orders o
    WHERE 1 = 1
      AND o.dt = '2024-01-01'
      AND o.user_id IS NOT NULL
      AND TRIM(LOWER(o.status)) = 'paid'
),
user_agg AS (
    SELECT user_id, SUM(amount) AS total_amount
    FROM base_orders
    GROUP BY user_id
)
SELECT DISTINCT u.user_id, COALESCE(a.total_amount, 0) AS total_amount
FROM users u
INNER JOIN user_agg a ON u.user_id = a.user_id;

-- Good：单表一步可完成，不套子查询；指标表用 t1，关键字全部小写，合同内结果完全等价
select
    t1.user_id,
    sum(t1.amount) as total_amount
from orders t1
where t1.dt = '2024-01-01'
  and t1.status = 'paid'
group by t1.user_id;
```

### 2. 多表关联：保留当日全部用户、无订单显示 0

已知数据合同：`dim_users` 在当日分区内 `user_id` 唯一；`dwd_orders` 中每笔订单 `order_id` 非空。

```sql
-- 维表 m1、指标表 t1 在各自内部子查询完成 dt 控制与筛选，外层负责关联与最终聚合
select
    m1.user_id,
    count(t1.order_id)          as order_cnt,    -- 未匹配时为 null，count 返回 0
    coalesce(sum(t1.amount), 0) as total_amount  -- 需求要求无订单显示 0
from (
    select user_id
    from dim_users
    where dt = '2024-01-01'
) m1
left join (
    select order_id, user_id, amount
    from dwd_orders
    where dt = '2024-01-01'
      and status = 'paid'
) t1
    on m1.user_id = t1.user_id
group by m1.user_id;
```

### 3. 防 join 扇出（一对多明细先在内部子查询聚合到关联键粒度）

已知口径：统计 `2024-01-01` 当日下单用户的订单总额，以及这些订单在当日发生的退款额（`orders` 按 `order_id` 唯一，一笔订单在 `refunds` 中可能有多笔退款）。

```sql
-- t1 完成当日订单分区裁剪，t2 完成当日退款按 order_id 预聚合消除扇出，外层关联并按用户汇总
select
    t1.user_id,
    sum(t1.amount)                      as order_amount,
    coalesce(sum(t2.refund_amount), 0)  as refund_amount
from (
    select order_id, user_id, amount
    from orders
    where dt = '2024-01-01'
) t1
left join (
    select order_id, sum(amount) as refund_amount
    from refunds
    where dt = '2024-01-01'
    group by order_id
) t2
    on t1.order_id = t2.order_id
group by t1.user_id;
```

### 4. 指定分区内每个用户最近一笔订单（分阶段窗口过滤与稳定决胜键）

已知数据合同：取 `2024-01-01` 分区内每用户 `created_at` 最新的一笔订单；同一时刻并列时按 `order_id` 倒序决胜。

```sql
-- 通用写法：单表需分阶段过滤窗口结果，在 t1 内完成 dt 控制与排序编号
select
    t1.user_id,
    t1.order_id,
    t1.amount,
    t1.created_at
from (
    select
        user_id,
        order_id,
        amount,
        created_at,
        row_number() over (partition by user_id order by created_at desc, order_id desc) as rn
    from orders
    where dt = '2024-01-01'
) t1
where t1.rn = 1;

-- 支持 qualify 的方言：单表一步可完成，无需嵌套
select
    t1.user_id,
    t1.order_id,
    t1.amount,
    t1.created_at
from orders t1
where t1.dt = '2024-01-01'
qualify row_number() over (partition by t1.user_id order by t1.created_at desc, t1.order_id desc) = 1;
```
