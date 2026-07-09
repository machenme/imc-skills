# Schema Queries — 多数据库 Schema 抽取 SQL

## When

需要从 MySQL / PostgreSQL / SQLite 数据库中提取表结构、字段、外键信息。

## Goal

获取完整的 Schema 元数据（表名 + 注释 + 字段列表 + 主键 + 外键），供 DDL 和 DBML 生成使用。

---

## 1. 查询所有用户表

**MySQL**：
```sql
SELECT TABLE_NAME, TABLE_COMMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = '{database}' AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME;
```

**PostgreSQL**：
```sql
SELECT tablename FROM pg_catalog.pg_tables
WHERE schemaname = 'public' ORDER BY tablename;
```

**SQLite**：
```sql
SELECT name FROM sqlite_master
WHERE type = 'table' AND name NOT LIKE 'sqlite_%'
ORDER BY name;
```

---

## 2. 查询单表字段元数据

> 若表数量 > 50，分批查询，每批 50 张。

**MySQL**：
```sql
SELECT COLUMN_NAME, ORDINAL_POSITION, DATA_TYPE, IS_NULLABLE,
       COLUMN_DEFAULT, COLUMN_KEY, EXTRA, COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = '{database}' AND TABLE_NAME = '{table}'
ORDER BY ORDINAL_POSITION;
```

**PostgreSQL**：
```sql
SELECT a.column_name, a.ordinal_position, a.data_type, a.is_nullable,
       a.column_default, a.is_identity,
       (SELECT tc.constraint_type FROM information_schema.table_constraints tc
        JOIN information_schema.key_column_usage kcu
        ON tc.constraint_name = kcu.constraint_name
        WHERE tc.table_name = '{table}' AND kcu.column_name = a.column_name
        AND tc.constraint_type = 'PRIMARY KEY') IS NOT NULL AS is_pk,
       pgd.description AS column_comment
FROM information_schema.columns a
LEFT JOIN pg_catalog.pg_description pgd
  ON pgd.objsubid = a.ordinal_position AND pgd.objoid = '{table}'::regclass
WHERE a.table_name = '{table}'
ORDER BY a.ordinal_position;
```

**SQLite**：
```sql
PRAGMA table_info('{table}');
```

---

## 3. 查询外键关系

**MySQL**：
```sql
SELECT CONSTRAINT_NAME, COLUMN_NAME,
       REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = '{database}' AND TABLE_NAME = '{table}'
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

**PostgreSQL**：
```sql
SELECT tc.constraint_name, kcu.column_name,
       ccu.table_name AS foreign_table_name,
       ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY' AND tc.table_name = '{table}';
```

**SQLite**：
```sql
PRAGMA foreign_key_list('{table}');
```

---

## 4. 无外键推断（Fallback）

当查询到的外键总数为 0 时启动：

```
遍历所有字段，匹配命名模式 /^(.+_)?(\w+)_id$/
  user_id → 查找名为 users 或 user 的表
  order_id → 查找名为 orders 或 order 的表

规则：
  - 优先匹配复数形式（users），回退单数形式（user）
  - 编辑距离最小的表名胜出
  - 推断的关系标记 is_inferred = true
```
