# MySQL 面试题

## 1.1 NoSQL 和 SQL 有什么区别？

SQL 数据库是关系型数据库，如 MySQL、PostgreSQL、Oracle 和 SQL Server。数据通常以二维表存储，具有固定的数据结构，可以通过外键和 `JOIN` 表达数据之间的关系，适合事务要求高、数据关系复杂的场景。

NoSQL 是非关系型数据库的统称，如 Redis 和 MongoDB。它可以使用键值、文档、列族或图等模型存储数据，数据结构通常更灵活，适合高并发、海量数据和分布式扩展场景。

两者的主要区别如下：

- **数据模型**：SQL 以关系表为主；NoSQL 支持键值、文档等多种模型。
- **事务与一致性**：SQL 通常强调 ACID 和强一致性；许多 NoSQL 系统更注重可用性与扩展性，可能采用 BASE 和最终一致性，但这不是绝对的，部分 NoSQL 也支持事务和强一致性。
- **扩展方式**：SQL 通常优先纵向扩展，分库分表后需要处理跨库查询和分布式事务；NoSQL 通常更容易通过分片进行水平扩展。
- **适用场景**：金融交易、订单等强事务场景通常优先选择 SQL；缓存、会话、日志和高并发读写等场景可以考虑 NoSQL。

实际项目中二者并非只能选择一个，常见做法是使用 SQL 保存核心业务数据，同时使用 Redis 等 NoSQL 数据库承担缓存或高并发访问。

## 1.2 数据库三大范式是什么？

数据库范式是设计关系型数据库表结构的一组规范，主要目的是减少数据冗余和更新异常。常见的三大范式如下：

### 第一范式（1NF）

要求每一列都是不可再分的原子值，不能在一个字段中保存一组数据。

例如，不要把多个电话号码写在同一个 `phone` 字段中，而应拆成独立记录或独立字段，使每个单元格只保存一个值。

### 第二范式（2NF）

在满足 1NF 的基础上，要求所有非主键列都完全依赖于整个主键，不能只依赖联合主键的一部分。

例如订单明细表使用 `(order_id, product_id)` 作为联合主键时，`quantity` 依赖整个联合主键；但 `order_time` 只依赖 `order_id`，应拆到订单表中，避免重复保存。

### 第三范式（3NF）

在满足 2NF 的基础上，要求非主键列不能依赖其他非主键列，也就是消除传递依赖。

例如学生表中，`student_id -> teacher_name -> teacher_phone`。教师电话应放到教师表中，学生表只保存 `teacher_id`，避免教师信息重复。

### 面试中的简便记忆

- 1NF：字段不可再分。
- 2NF：非主键列完全依赖整个主键。
- 3NF：非主键列不能依赖其他非主键列。

范式可以减少冗余并避免插入、更新、删除异常，但范式越高，表拆分通常越多。实际项目会根据查询性能和业务需要，在规范化与适度反规范化之间做取舍。

## 1.3 MySQL 怎么进行联表查询？

联表查询使用 `JOIN` 根据两个表之间的关联字段组合数据，常见类型如下。假设员工表 `employees.department_id` 关联部门表 `departments.id`：

### 1. 内连接（`INNER JOIN`）

只返回两个表中能够匹配的记录：

```sql
SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
INNER JOIN departments AS d
  ON e.department_id = d.id;
```

### 2. 左外连接（`LEFT JOIN`）

返回左表的全部记录。右表没有匹配时，右表字段为 `NULL`：

```sql
SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
LEFT JOIN departments AS d
  ON e.department_id = d.id;
```

这可以查出所有员工，包括尚未分配部门的员工。

### 3. 右外连接（`RIGHT JOIN`）

返回右表的全部记录。左表没有匹配时，左表字段为 `NULL`：

```sql
SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
RIGHT JOIN departments AS d
  ON e.department_id = d.id;
```

这可以查出所有部门，包括暂时没有员工的部门。实际开发中通常改写表的顺序后使用 `LEFT JOIN`，可读性更好。

### 4. 全外连接（`FULL JOIN`）

全外连接会返回两张表的所有记录，包括双方无法匹配的记录。MySQL 不直接支持 `FULL JOIN`，可以使用 `LEFT JOIN` 和 `RIGHT JOIN` 组合实现：

```sql
SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
LEFT JOIN departments AS d
  ON e.department_id = d.id

UNION

SELECT e.name AS employee_name, d.name AS department_name
FROM employees AS e
RIGHT JOIN departments AS d
  ON e.department_id = d.id;
```

其中 `UNION` 会去除完全重复的行；如果使用 `UNION ALL`，则需要额外过滤已匹配的记录，避免重复结果。

### 简单记忆

- `INNER JOIN`：只要匹配的。
- `LEFT JOIN`：左表全部保留。
- `RIGHT JOIN`：右表全部保留。
- `FULL JOIN`：两边全部保留，MySQL 需组合实现。

## 1.4 MySQL 如何避免重复插入数据？

### 1. 使用 `UNIQUE` 唯一约束

在需要保证唯一的列上建立唯一索引或唯一约束，由数据库从根本上防止重复数据：

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255)
);
```

当插入相同的 `email` 时，MySQL 会返回重复键错误。这是保证数据唯一性的基础方案。

### 2. `INSERT ... ON DUPLICATE KEY UPDATE`

如果遇到重复键，不报错，而是更新已有记录，适合“插入或更新”（upsert）：

```sql
INSERT INTO users (email, name)
VALUES ('example@example.com', 'John Doe')
ON DUPLICATE KEY UPDATE name = 'John Doe';
```

### 3. `INSERT IGNORE`

遇到重复键时忽略本次插入，其他可插入的数据仍可继续处理：

```sql
INSERT IGNORE INTO users (email, name)
VALUES ('example@example.com', 'John Doe');
```

需要注意，`INSERT IGNORE` 可能同时忽略部分数据校验错误，不应在不了解影响的情况下滥用。

### 如何选择？

- 必须保证唯一：使用 `UNIQUE` 约束。
- 重复时需要更新：使用 `ON DUPLICATE KEY UPDATE`。
- 重复时直接跳过：使用 `INSERT IGNORE`。

## 1.5 `CHAR` 和 `VARCHAR` 有什么区别？

- **`CHAR`**：固定长度字符串。定义长度后，每个值按固定长度存储，不足部分通常用空格补齐，适合身份证号、固定长度编码、状态码等长度稳定的数据。
- **`VARCHAR`**：可变长度字符串。定义的是最大长度，实际占用空间取决于字符串实际长度，并额外保存长度信息，适合姓名、地址、备注等长度不固定的数据。

例如：

```sql
code CHAR(6),        -- 固定长度编码
nickname VARCHAR(50) -- 最多 50 个字符的昵称
```

简单来说，`CHAR` 读取长度固定、适合短小且长度稳定的值；`VARCHAR` 更节省可变文本的存储空间。选择类型时还要结合字符集，因为 `VARCHAR(n)` 的实际最大字节数会受到字符集影响。

## 1.6 `VARCHAR` 后面的数字代表字节还是字符？

在 MySQL 中，`VARCHAR(n)` 的 `n` 表示**最多可以存储的字符数**，不是字节数。

```sql
nickname VARCHAR(10)
```

这表示最多存储 10 个字符。实际占用的字节数取决于字符集：

- ASCII 字符通常每个占 1 个字节，10 个字符约占 10 个字节。
- `utf8mb4` 字符每个最多占 4 个字节，10 个字符最多可能占 40 个字节，另外还需要保存长度信息。

因此，`VARCHAR(10)` 既可以存 10 个英文字母，也可以存 10 个中文字符；但中文通常占用更多字节。设计表结构时还要注意 MySQL 单行最大字节数等限制。

## 1.7 `INT(1)` 和 `INT(10)` 在 MySQL 中有什么区别？

`INT(1)` 和 `INT(10)` 的括号数字不是取值范围，也不会改变 `INT` 的存储大小。普通 `INT` 始终占用 4 个字节，取值范围也相同：

```sql
CREATE TABLE test_int (
    num1 INT(1),
    num2 INT(10)
);
```

历史上，这个数字被称为“显示宽度”，配合 `ZEROFILL` 时可以在显示时补前导零：

```sql
CREATE TABLE test_int (
    num1 INT(1) ZEROFILL,
    num2 INT(10) ZEROFILL
);

INSERT INTO test_int VALUES (1, 1);
-- 旧版本客户端可能显示：1、0000000001
```

它不会限制可以存入的数字，也不会把数字截断。MySQL 8.0.17 起，整数类型的显示宽度已被弃用，`ZEROFILL` 也不建议用于新设计，因此不要用 `INT(1)` 表示只能存 0 或 1；需要限制取值时，应使用 `TINYINT`、`CHECK` 约束或业务逻辑。
