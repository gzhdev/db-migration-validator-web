# mapping_template.xlsx 填写说明

> 下载地址：映射配置生成器页面 → **下载 Excel 模板**

模板包含 4 个 Sheet，其中前 3 个为可编辑的数据 Sheet，第 4 个为说明：

| Sheet | 说明 |
|-------|------|
| 单表示例 | 单张源表映射到目标表 |
| 多表JOIN示例 | 多张源表 JOIN 后映射到目标表 |
| 多源插入示例 | 多个 INSERT 条件各自映射到同一目标表 |
| 填写说明 | 列定义速查（只读） |

上传时，所有数据 Sheet 会按顺序合并导入（"填写说明" 自动跳过）。

---

## 一、列说明

### 表级列（每个映射条目的第一行填写）

| 列名 | 必填 | 说明 |
|------|------|------|
| `source_table` | 单表必填 | 源表名，与 `join_tables` 二选一 |
| `target_table` | 是 | 目标表名 |
| `source_schema` | 否 | 源表 Schema（PostgreSQL 等需要） |
| `target_schema` | 否 | 目标表 Schema |
| `description` | 否 | 本条映射的说明文字 |
| `source_filter` | 否 | 源表 WHERE 条件，如 `deleted_at IS NULL` |
| `target_filter` | 否 | 目标表 WHERE 条件；多源插入同一目标表时**必填** |
| `sample_size` | 否 | 抽样行数，`0` 表示全量（默认） |
| `batch_size` | 否 | 批量大小，默认 `1000` |
| `join_tables` | 多表必填 | 多表 JOIN 配置，填 JSON 数组（见第三节） |
| `table_filters` | 否 | 多表模式各源表过滤，填 JSON 对象（见第三节） |

### 字段级列（每行一个字段，可连续填写多行）

| 列名 | 必填 | 说明 |
|------|------|------|
| `target_field` | 是 | 目标字段名 |
| `source_field` | 条件 | 源字段名；无转换时必填，多表用 `alias.field` 格式 |
| `is_primary_key` | 否 | `true` / `false`，默认 false |
| `transform_type` | 否 | 转换类型（见第四节） |
| `transform_fields` | 否 | 转换涉及的源字段，多个用 `\|` 分隔，如 `first_name\|last_name` |
| `transform_params` | 否 | 转换参数，JSON 格式，如 `{"separator": " "}` |
| `compare_rule` | 否 | 比较规则，默认 `exact`（见第五节） |
| `tolerance` | 否 | 数值容差，`compare_rule=numeric_tolerance` 时填写 |
| `nullable` | 否 | `false` 表示不允许 NULL，默认 `true` |
| `min_value` | 否 | 最小值校验 |
| `max_value` | 否 | 最大值校验 |
| `allowed_values` | 否 | 允许值列表，多个用 `\|` 分隔，如 `active\|inactive` |
| `pattern` | 否 | 正则表达式校验 |
| `value_check_expr` | 否 | 自定义 Spark SQL 布尔表达式校验 |

---

## 二、单表模式

### 规则

- `source_table` 和 `target_table` 在**每个映射条目的第一行**填写
- 后续字段行的 `source_table` 和 `target_table` 留空，表示继续属于上一个条目
- 遇到新的 `source_table` 或 `target_table` 时，开始新的映射条目

### 示例：users → user_profile

| source_table | target_table | source_filter | target_filter | target_field | source_field | is_primary_key | transform_type | transform_fields | transform_params | compare_rule | tolerance | nullable |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| users | user_profile | deleted_at IS NULL | is_deleted = false | user_id | id | true | | | | | | |
| | | | | full_name | | | concat | first_name\|last_name | {"separator": " "} | | | |
| | | | | email_address | email | | | | | ignore_case | | |
| | | | | phone | phone_number | | | | | ignore_whitespace | | |
| | | | | balance | account_balance | | | | | numeric_tolerance | 0.01 | |
| | | | | status_code | | | case | status | {"cases": [{"when": "active", "then": "1"}, {"when": "inactive", "then": "0"}], "else": "0"} | | | |
| | | | | remark | remark | | | | | skip | | |
| | | | | age | age | | | | | | | false |

说明：
- `full_name`：拼接 first_name 和 last_name，中间加空格
- `email_address`：忽略大小写比较
- `balance`：允许 0.01 以内的数值误差
- `status_code`：枚举转换，active→1，inactive→0，其余→0
- `remark`：跳过比较
- `age`：`nullable=false`，目标值不允许为 NULL

---

## 三、多表 JOIN 模式

### `join_tables` 格式（JSON 数组）

```json
[
  {"table_name": "orders",   "alias": "o", "join_type": "primary"},
  {"table_name": "users",    "alias": "u", "join_type": "left",  "join_condition": "o.user_id = u.id"},
  {"table_name": "products", "alias": "p", "join_type": "inner", "join_condition": "o.product_id = p.id"}
]
```

| 字段 | 说明 |
|------|------|
| `table_name` | 源表名 |
| `alias` | 表别名，字段引用时使用，如 `o.id` |
| `join_type` | 第一个表填 `primary`；其余填 `inner` / `left` / `right` / `full` / `cross` |
| `join_condition` | 第一个表不填；其余表必填 JOIN 条件 |
| `schema` | 可选，源表 Schema |

### `table_filters` 格式（JSON 对象）

```json
{"o": "o.deleted_at IS NULL", "u": "u.status = 'active'"}
```

key 为表别名，value 为该表的 WHERE 条件。

### 示例：orders JOIN users → order_detail

| join_tables | table_filters | target_table | target_field | source_field | is_primary_key | transform_type | transform_fields | transform_params | compare_rule | tolerance | nullable |
|---|---|---|---|---|---|---|---|---|---|---|---|
| [{"table_name":"orders","alias":"o","join_type":"primary"},{"table_name":"users","alias":"u","join_type":"left","join_condition":"o.user_id = u.id"}] | {"o":"o.deleted_at IS NULL"} | order_detail | order_id | o.id | true | | | | | | |
| | | | customer_name | u.name | | | | | | | true |
| | | | order_no | | | concat | o.prefix\|o.seq | {"separator": "-"} | | | |
| | | | total_amount | | | math | o.price\|o.qty | {"expression": "o.price * o.qty"} | numeric_tolerance | 0.001 | |
| | | | created_at | o.create_time | | | | | | | |

说明：
- `customer_name`：来自 LEFT JOIN 的 users 表，用户可能不存在，`nullable=true`
- `order_no`：拼接前缀和序号
- `total_amount`：数学表达式计算，允许 0.001 误差

---

## 四、多源插入同一目标表

适用场景：存储过程中有多个 INSERT 语句，按不同条件从不同源表插入同一张目标表。

### 规则

- 在 Excel 中写**多个映射条目**，每个条目对应一个 INSERT 语句
- 每个条目的 `target_table` 相同
- **每个条目必须填写 `target_filter`**，且各条目的 `target_filter` 须满足：
  - **互斥**：同一行目标数据只能被一个条目匹配
  - **完整覆盖**：目标表所有行都能被某个条目匹配到

> 如果 `target_filter` 有遗漏，那部分目标数据不会被任何条目校验，**不会报错，静默跳过**。

### 示例：3 个 INSERT → target 表

假设存储过程逻辑如下：

```sql
-- INSERT 1: A 类数据，直接从 source_a 取
INSERT INTO target SELECT id, name, value FROM source_a WHERE type = 'A';

-- INSERT 2: B 类数据，需要 JOIN 补充信息
INSERT INTO target
SELECT a.id, b.display_name, a.price * b.factor
FROM source_a a LEFT JOIN source_b b ON a.id = b.ref_id
WHERE a.type = 'B';

-- INSERT 3: 其余数据，从 source_c 取
INSERT INTO target SELECT id, title, score FROM source_c WHERE type != 'A' AND type != 'B';
```

对应 Excel 填写（3 个条目，`target_filter` 互斥覆盖）：

**条目 1**

| source_table | target_table | source_filter | target_filter | target_field | source_field | is_primary_key |
|---|---|---|---|---|---|---|
| source_a | target | type = 'A' | type = 'A' | id | id | true |
| | | | | name | name | |
| | | | | value | value | |

**条目 2**

| join_tables | target_table | table_filters | target_filter | target_field | source_field | is_primary_key | transform_type | transform_fields | transform_params |
|---|---|---|---|---|---|---|---|---|---|
| [{"table_name":"source_a","alias":"a","join_type":"primary"},{"table_name":"source_b","alias":"b","join_type":"left","join_condition":"a.id = b.ref_id"}] | target | {"a":"a.type = 'B'"} | type = 'B' | id | a.id | true | | | |
| | | | | name | b.display_name | | | | |
| | | | | value | | | math | a.price\|b.factor | {"expression": "a.price * b.factor"} |

**条目 3**

| source_table | target_table | source_filter | target_filter | target_field | source_field | is_primary_key |
|---|---|---|---|---|---|---|
| source_c | target | type != 'A' AND type != 'B' | type != 'A' AND type != 'B' | id | id | true |
| | | | | name | title | |
| | | | | value | score | |

### 互斥覆盖检验方法

在数据库中执行以下 SQL 验证 `target_filter` 设计是否正确：

```sql
-- 检查重叠（应返回 0 行）
SELECT COUNT(*) FROM target
WHERE (type = 'A')
  AND (type = 'B');          -- 每两个条件之间做 AND 检查

-- 检查覆盖完整性（应与目标表总行数一致）
SELECT COUNT(*) FROM target
WHERE type = 'A'
   OR type = 'B'
   OR (type != 'A' AND type != 'B');
```

---

## 五、转换类型参考

| `transform_type` | `transform_fields` | `transform_params` | 等价 SQL |
|---|---|---|---|
| `concat` | `f1\|f2` | `{"separator": "-"}` | `CONCAT(f1, '-', f2)` |
| `upper` | `name` | | `UPPER(name)` |
| `lower` | `name` | | `LOWER(name)` |
| `trim` | `name` | | `TRIM(name)` |
| `substring` | `code` | `{"start": 1, "length": 3}` | `SUBSTR(code, 1, 3)` |
| `replace` | `desc` | `{"pattern": "foo", "replacement": "bar"}` | `REPLACE(desc, 'foo', 'bar')` |
| `constant` | （留空） | `{"value": "Y"}` | `'Y'` |
| `constant` | （留空） | `{"value": null}` | `NULL` |
| `coalesce` | `f1\|f2` | `{"default": "N/A"}` | `COALESCE(f1, f2, 'N/A')` |
| `case` | `status` | `{"cases": [{"when": "active", "then": "1"}, {"when": "inactive", "then": "0"}, {"when": "pending", "then": "2"}], "else": "-1"}` | `CASE WHEN status='active' THEN '1' WHEN status='inactive' THEN '0' WHEN status='pending' THEN '2' ELSE '-1' END` |
| `case`（复合条件） | `A\|B` | `{"cases": [{"when_expr": "A=0 AND B=1", "then": "0"}, {"when_expr": "A=1 AND B=0", "then": "2"}], "else": "1"}` | `CASE WHEN A=0 AND B=1 THEN '0' WHEN A=1 AND B=0 THEN '2' ELSE '1' END` |
| `date_format` | `create_time` | `{"format": "%Y-%m-%d"}` | `DATE_FORMAT(create_time, '%Y-%m-%d')` |
| `json_extract` | `metadata` | `{"pattern": "$.user.id"}` | `GET_JSON_OBJECT(metadata, '$.user.id')` |
| `cast` | `amount` | `{"to": "int"}` | `CAST(amount AS INT)` |
| `math` | `price\|qty` | `{"expression": "price * qty"}` | `price * qty` |

### case 多分支详解

`cases` 数组中每个对象是一个 `WHEN` 分支，按顺序匹配，第一个匹配的分支生效，`else` 对应 `ELSE`（可不填，不填时未匹配的返回 NULL）。

**等值多分支**（所有分支比较同一个字段）

`transform_fields` 填源字段名，每个分支用 `when` 指定匹配值：

```json
transform_fields: status
transform_params:
{
  "cases": [
    {"when": "active",   "then": "1"},
    {"when": "inactive", "then": "0"},
    {"when": "pending",  "then": "2"},
    {"when": "deleted",  "then": "9"}
  ],
  "else": "-1"
}
```

等价 SQL：
```sql
CASE status
  WHEN 'active'   THEN '1'
  WHEN 'inactive' THEN '0'
  WHEN 'pending'  THEN '2'
  WHEN 'deleted'  THEN '9'
  ELSE '-1'
END
```

**复合条件多分支**（每个分支条件涉及不同字段或函数）

`transform_fields` 填所有分支涉及的字段（用 `|` 分隔），每个分支用 `when_expr` 写完整的 SQL 条件：

```json
transform_fields: type|category
transform_params:
{
  "cases": [
    {"when_expr": "type = 'A'",                       "then": "1"},
    {"when_expr": "type = 'B' AND category = 'X'",    "then": "2"},
    {"when_expr": "type = 'B' AND category != 'X'",   "then": "3"},
    {"when_expr": "type IS NULL OR category IS NULL",  "then": "0"}
  ],
  "else": "-1"
}
```

**混合写法**（等值分支和复合条件分支可以出现在同一个 cases 数组中）

```json
transform_fields: status|region
transform_params:
{
  "cases": [
    {"when": "VIP",                                      "then": "gold"},
    {"when_expr": "status = 'normal' AND region = 'CN'", "then": "silver"},
    {"when_expr": "status = 'normal' AND region != 'CN'","then": "bronze"}
  ],
  "else": "none"
}
```

**在 Excel 单元格中填写时**，`transform_params` 整列填一个 JSON 字符串，直接粘贴即可，Excel 不需要对内部双引号做额外处理。

---

## 六、比较规则参考

| `compare_rule` | 说明 | 适用场景 |
|---|---|---|
| `exact` | 精确比较（默认） | 字符串、ID 等 |
| `ignore_case` | 忽略大小写 | email、枚举值 |
| `ignore_whitespace` | 忽略前后空白 | 手动录入的文本 |
| `numeric_tolerance` | 数值容差，需填 `tolerance` | 浮点数、金额 |
| `skip` | 跳过比较 | 不关心的字段、自动生成字段 |

---

## 七、常见问题

**Q：主键由多个字段组成怎么配置？**

将多个字段的 `is_primary_key` 都设为 `true`，工具会用这些字段的组合作为 JOIN 键。

**Q：目标表有自动生成的字段（如 created_at），不想比较怎么办？**

`compare_rule` 填 `skip`。

**Q：源表字段是整数，目标表是字符串，比较会失败吗？**

`transform_type` 填 `cast`，`transform_params` 填 `{"to": "string"}`，将源字段转成字符串后再比较。

**Q：Excel 里的 JSON 里有双引号，和单元格冲突怎么办？**

在 Excel 单元格中直接输入 JSON 文本，Excel 会自动处理引号。如果使用公式栏输入，双引号需要成对转义（`""`）。建议直接在单元格内粘贴，不要通过公式生成。

**Q：多源插入时，校验报告里数据总量对不上怎么排查？**

逐一检查每个条目的 `target_filter` 是否覆盖了目标表中该 INSERT 产生的全部行。在目标库执行：
```sql
SELECT COUNT(*) FROM target WHERE <target_filter>;  -- 各条目分别统计
SELECT COUNT(*) FROM target;                         -- 目标表总行数
```
两者之和应等于目标表总行数。
