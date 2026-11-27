---
name: hive-etl-sql-template
description: >-
  创建、改写或审查 Hive 数仓目标表 DDL、insert overwrite 分区写入或其日调度成品脚本（如 DIM/DWD/DWS/ADS 层）时使用；
  即使只修改建表段或只修改写入段也触发。只要涉及 SELECT 查询体，必须同时加载并遵循同环境的 simple-direct-sql 基础技能；
  单纯写通用 SELECT 取数查询请只使用 simple-direct-sql。
---

# Hive 数仓加工 SQL 规范与校验流程

> **路由与依赖规则**
> - 本技能专职覆盖 **Hive 数仓单文件交付结构、DDL 建模与物理存储、调度宏、全量字段中文注释、Hive 写入列对位及引擎校验**，不重复定义通用查询规则。
> - 只要任务包含 `select` 查询逻辑，**必须同时加载当前环境中的 `simple-direct-sql` 技能**并严格执行其全部规则（若仅编写或修改纯建表 DDL，无需额外加载查询规则）。
> - **明确覆盖项（以本技能为准）**：
>   1. **交付形式**：数仓落表任务按每张目标表独立 `.sql` 成品模板交付，覆盖 `simple-direct-sql` 默认输出单个查询代码块的约定；
>   2. **注释粒度**：建表与内外层所有 `select` 投影列均须带简洁准确的中文注释，覆盖 `simple-direct-sql` 仅对非显而易见口径写注释的约定。

## 一、Hive 单文件成品模板与建表规范

1. **三段式单文件结构**（每张目标表独立一个 `.sql` 文件）：
   - 顶部两行注释态清表语句：
     - `-- alter table <db>.<table> set tblproperties ('external.table.purge'='true');`
     - `-- drop table <db>.<table>;`
   - 建表语句：`create external table if not exists <db>.<table> (...)`（注意：`if not exists` 不会更新已存在的旧表结构，改表时须核对线上实际表结构或配合清表重建流程）。
   - 写入语句：`insert overwrite table <db>.<table> partition (...) select ...;`（含动态分区写入时，在 `insert overwrite` 前声明 `set hive.exec.dynamic.partition = true;` 与 `set hive.exec.dynamic.partition.mode = nonstrict;`）。
2. **全量中文注释与按位置写入对位（权威定义）**：
   - 建表语句的表注释、每个普通列注释、每个分区列注释均须为**简洁准确的中文注释**。
   - `insert overwrite` 的外层 `select` 及**所有内部子查询的 `select`** 中，每个输出列都必须显式写 `as <字段名>`，并在行尾补充 `-- <中文注释>`（同层建议纵向对齐以利阅读）。
   - **Hive 按位置写入对位规则**：
     - 外层 `select` 先按 DDL 定义顺序依次输出**全部普通列**；
     - **静态分区列**只在 `partition (dt = '...')` 中赋值，**严禁出现在外层 `select` 投影中**；
     - **动态分区列**按 `partition (...)` 中列出的顺序依次排在外层 `select` 最末尾，且同样须带显式 `as <分区列名>` 与 `-- <中文注释>`；
     - **混合分区**按 DDL `partitioned by` 的分区列顺序书写 `partition (...)`，静态分区在前、动态分区在后，且各输出列与目标列类型无损兼容。
3. **建表与输出列排列顺序**：
   - 严格遵循**维度在前，指标在后，同类字段归组相邻**。
   - 推荐分组顺序：`时间/日期维度` → `业务实体维度（ID/名称/属性/空间坐标等）` → `行政区划维度（按省→市→区县层级，编码与名称成对相邻）` → `事实指标/度量（按次数/人数/流量/金额/比例等同类归组）`。
4. **调度日期宏**：
   - 统一使用 `'$[yyyy-MM-dd-1]'`（注意 `MM` 为大写月份格式字面量，不可小写化；不使用 `${biz_date}` 等其他占位符）。
   - 自然日时间戳过滤结合左闭右开写法：`data_time >= '$[yyyy-MM-dd-1] 00:00:00' and data_time < date_add('$[yyyy-MM-dd-1]', 1)`。
5. **Hive 数据类型与显式转换边界**：
   - 整数类型统一用 `bigint`，不使用 `int`；涉及精确对账的货币/金额或高精度比例时，按源表与目标合同选用 `decimal(p, s)`，不盲目用浮点 `double` 替代精确金额。
   - 不写无意义的 `cast(...)`；当**关联键精度保护**（如 19 位长整数发生 `string = bigint` 比较会隐式转 `double` 丢精度）、**货币/小数精度控制**、**`union all` 分支类型对齐**或**目标表列类型合同**要求时，显式转为无损兼容类型。
6. **分层存储格式与表属性**：
   - DIM 层：`stored as orc`，`tblproperties ('orc.compress' = 'SNAPPY', 'serialization.format' = '1')`。
   - DWD / DWS / ADS 层（默认，若目标库同层既有表另有统一存储规范则对齐既有表）：`stored as parquet`，`tblproperties ('parquet.compression' = 'snappy', 'serialization.format' = '1')`。

## 二、Hive 语法与执行引擎避坑

1. **Hive 无 `qualify` 下的版本去重落地**（仅在核实存在重复或多版本时使用）：
   - **严格遵循 `simple-direct-sql` 的版本去重先后语义**：「实体最新状态满足条件」须先在完整候选版本集上按排序键 + 稳定决胜键取 `rn = 1`，再过滤可变状态与事件时间（且窗口内仅用于分组/排序而不被外层读取的列无需额外投影）；「满足条件的最新记录」才先筛后排。
   - 单表去重：内部子查询算 `rn`，外层 `from (...) t1 where t1.rn = 1`。
   - 多表关联去重：需去重的表必须在自身子查询内先完成 `rn = 1` 过滤（内部过渡别名用 `t1_0` / `m1_0`）再作为 `t1` / `m1` 参与关联，不把单表的 `rn = 1` 留到最外层 `where`。
   - `union all` 去重：仅当最终去重键在各分支间严格互斥时才下推至分支内去重；否则须合并候选集后再统一取 `rn = 1`。
2. **MapJoin 内存不足排查与处理**：
   - 仅在通过 `explain` 执行计划与任务日志确认是 MapJoin 小表哈希表构建侧内存不足后，再对该作业评估设置 `set hive.auto.convert.join = false;` 改走 Reduce Join（维表行数与 `MapredLocalTask exit status: 3` 仅作为排查线索，不能单独等同于 OOM）。
3. **规避 Hive 笛卡尔积检查失败**：
   - 关联维表时，外层 `on` 必须包含左表列与右表列的关联条件（如先在 `t1` 内算出 `substr(data_time, 1, 10) as data_date`，外层写 `on t1.data_date = m1.date_key`），严禁在外层 `on` 中只写常量条件（如 `on m1.date_key = '$[yyyy-MM-dd-1]'`）。
4. **上游 ODS 分区延迟与增量/快照核查**：
   - 写分区裁剪前先核查上游 ODS 是全量快照、日增量还是 CDC 变更流，以及是否存在 T-2 延迟，防止静态分区裁剪漏掉有效版本或跑出 0 行。

## 三、交付前校验清单

1. **静态合规校验**：逐项核对「一·2」的位置对位（静态/动态分区列位置、类型兼容、中文注释完整性），并对照 `simple-direct-sql` 准确性清单确认无语义隐患、无残留异构日期宏、括号配对无误。
2. **Hive 实测验证**（有可用 `dbx` Hive 连接时，以最近有效分区执行只读 `select` 验证，严禁直接修改线上库表）：
   - **主键非空与唯一性**：单列主键验证 `pk is null` 行数为 0 且无重复；复合主键逐列检查非空，并按原始键列组合 `group by k1, k2, … having count(*) > 1` 检查重复（不要用字符串拼接代替复合键，以免 `'1'+'23'` 与 `'12'+'3'` 发生碰撞误判）；
   - **无扇出与度量守恒**：对比关联前后主表行数，核对支付金额、退款金额等核心可加指标在关联与汇总前后是否守恒（单看 `count(*) = count(distinct pk)` 不能单独证明无漏数或汇总表内部无扇出）；
   - **关联命中与未匹配核查**：统计各维表未匹配行数与未匹配键分布，确认符合业务预期；
   - **数据质量与完整性**：核查可空字符串字段的空串 `''` 占比、时间范围 `min/max` 及时间片完整性（如 5 分钟粒度满 288 片）；若上游新建 DIM 尚未落表，可将已验证的 DIM `select` 内联代入验证下游逻辑。

## 四、标准成品模板示例

### 1. 单表直抽静态分区模板（一步可完成，不套子查询）

```sql
-- alter table dwd.dwd_example_info_da set tblproperties ('external.table.purge'='true');
-- drop table dwd.dwd_example_info_da;
create external table if not exists dwd.dwd_example_info_da
(
    id          string comment '主键id',
    name        string comment '名称',
    create_time string comment '创建时间'
) comment '示例明细表'
    partitioned by (
        dt string comment '数据日期 格式yyyy-MM-dd'
        )
    stored as parquet
    tblproperties (
        'parquet.compression' = 'snappy',
        'serialization.format' = '1'
        );

insert overwrite table dwd.dwd_example_info_da
    partition (dt = '$[yyyy-MM-dd-1]')
select t1.id          as id,         -- 主键id
       t1.name        as name,       -- 名称
       t1.create_time as create_time -- 创建时间
from ods.ods_example_da t1
where t1.dt = '$[yyyy-MM-dd-1]'
;
```

### 2. 多表关联静态分区模板（单表筛选、dt 控制与转换内聚于子查询）

已知合同：`ods.ods_order_da` 当日 `order_id` 唯一，`create_time` 为 `'yyyy-MM-dd HH:mm:ss'` 格式；`dim.dim_region_da` 当日 `region_code` 唯一。

```sql
-- alter table dwd.dwd_order_detail_da set tblproperties ('external.table.purge'='true');
-- drop table dwd.dwd_order_detail_da;
create external table if not exists dwd.dwd_order_detail_da
(
    order_date  string        comment '订单日期',
    order_id    string        comment '订单id',
    region_code string        comment '区划编码',
    region_name string        comment '区划名称',
    item_cnt    bigint        comment '商品件数',
    pay_amount  decimal(18,2) comment '实付金额'
) comment '订单明细日表'
    partitioned by (
        dt string comment '数据日期 格式yyyy-MM-dd'
        )
    stored as parquet
    tblproperties (
        'parquet.compression' = 'snappy',
        'serialization.format' = '1'
        );

insert overwrite table dwd.dwd_order_detail_da
    partition (dt = '$[yyyy-MM-dd-1]')
select t1.order_date  as order_date,  -- 订单日期
       t1.order_id    as order_id,    -- 订单id
       t1.region_code as region_code, -- 区划编码
       m1.region_name as region_name, -- 区划名称
       t1.item_cnt    as item_cnt,    -- 商品件数
       t1.pay_amount  as pay_amount   -- 实付金额
from (
    select substr(create_time, 1, 10) as order_date,  -- 订单日期
           order_id                   as order_id,    -- 订单id
           region_code                as region_code, -- 区划编码
           item_cnt                   as item_cnt,    -- 商品件数
           pay_amount                 as pay_amount   -- 实付金额
    from ods.ods_order_da
    where dt = '$[yyyy-MM-dd-1]'
      and order_status = 'paid'
) t1
left join (
    select region_code as region_code, -- 区划编码
           region_name as region_name  -- 区划名称
    from dim.dim_region_da
    where dt = '$[yyyy-MM-dd-1]'
) m1
    on t1.region_code = m1.region_code
;
```

### 3. 动态分区写入对位写法（动态分区列置于外层投影末尾）

适用前提：**仅当输入数据完整覆盖每个待重写目标分区全量数据**（避免局部增量 `insert overwrite` 冲掉目标日存量明细），且**业务明确要求按最新调度日维表口径重刷历史分区**时使用（若要求还原历史当日维度，须关联对应历史快照维表）。

```sql
set hive.exec.dynamic.partition = true;
set hive.exec.dynamic.partition.mode = nonstrict;

insert overwrite table dwd.dwd_order_detail_da
    partition (dt)
select t1.order_date  as order_date,  -- 订单日期
       t1.order_id    as order_id,    -- 订单id
       t1.region_code as region_code, -- 区划编码
       m1.region_name as region_name, -- 区划名称
       t1.item_cnt    as item_cnt,    -- 商品件数
       t1.pay_amount  as pay_amount,  -- 实付金额
       t1.order_date  as dt           -- 数据日期 格式yyyy-MM-dd
from (
    select substr(create_time, 1, 10) as order_date,  -- 订单日期
           order_id                   as order_id,    -- 订单id
           region_code                as region_code, -- 区划编码
           item_cnt                   as item_cnt,    -- 商品件数
           pay_amount                 as pay_amount   -- 实付金额
    from ods.ods_order_da
    where dt >= '2026-11-01'
      and dt <= '$[yyyy-MM-dd-1]'
      and order_status = 'paid'
) t1
left join (
    select region_code as region_code, -- 区划编码
           region_name as region_name  -- 区划名称
    from dim.dim_region_da
    where dt = '$[yyyy-MM-dd-1]'
) m1
    on t1.region_code = m1.region_code
;
```
