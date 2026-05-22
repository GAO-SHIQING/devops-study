# SQL / 数据库基础（PostgreSQL）零基础学习路线

---

## 一、前置条件

- 已掌握 Python 基础（能写脚本、理解数据结构）
- 理解基本的命令行操作
- 有 Docker 基础（会用 Docker 启动 PostgreSQL 容器，免去本地安装）

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 关系模型与    │    │ 查询进阶      │    │ 设计与优化    │    │ Python + 实战 │
│ CRUD          │    │               │    │               │    │               │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：关系模型与 CRUD

**目标：理解关系数据库的核心概念，能用 SQL 建表、增删改查**

#### Day 1：数据库是什么

| 主题 | 内容 |
|------|------|
| 为什么需要数据库 | 文件存数据的痛点：并发冲突、查询困难、一致性无保障 |
| DBMS | 数据库管理系统：PostgreSQL、MySQL、SQLite、Oracle |
| 关系型 vs NoSQL | 结构化数据 + SQL vs 文档/键值/图（MongoDB、Redis） |
| PostgreSQL 特点 | ACID、MVCC、扩展性强、SQL 标准兼容度高 |

**练习：** 用 Docker 启动 PostgreSQL 实例：

```bash
docker run -d --name pg-learn \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=learn \
  -p 5432:5432 \
  postgres:16-alpine

docker exec -it pg-learn psql -U postgres -d learn
```

#### Day 2：表、行、列 — 创建你的第一张表

| 主题 | 说明 |
|------|------|
| 表 (Table) | 关系的物理表示，二维结构 |
| 行 (Row) | 一条记录 |
| 列 (Column) | 一个字段 / 属性 |
| Schema | 命名空间，组织表的逻辑分组 |

**练习：**

```sql
CREATE SCHEMA shop;

CREATE TABLE shop.products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    price       DECIMAL(10,2),
    created_at  TIMESTAMP DEFAULT now()
);

-- 查看表结构
\d shop.products
```

#### Day 3：数据类型

| 类别 | 类型 |
|------|------|
| 整数 | `SMALLINT` / `INTEGER` / `BIGINT` / `SERIAL` (自增) |
| 浮点 | `DECIMAL(precision, scale)` (精确) / `REAL` / `DOUBLE PRECISION` (近似) |
| 文本 | `CHAR(n)` (定长) / `VARCHAR(n)` (变长) / `TEXT` (无限制) |
| 布尔 | `BOOLEAN` (`true` / `false` / `NULL`) |
| 日期时间 | `DATE` / `TIME` / `TIMESTAMP` / `TIMESTAMPTZ` (带时区) |
| JSON | `JSON` (存原文) / `JSONB` (二进制，可索引 — PG 特色) |
| 数组 | `INTEGER[]` / `TEXT[]` — PG 特有 |

**练习：** 创建一张 `users` 表，使用至少 8 种不同类型。插入 5 行数据，体会每种类型的写入方式。

#### Day 4：约束

| 约束 | 说明 |
|------|------|
| `NOT NULL` | 列不允许为 NULL |
| `UNIQUE` | 列值必须唯一（允许一个 NULL） |
| `PRIMARY KEY` | = NOT NULL + UNIQUE，每表一个 |
| `FOREIGN KEY` | 参照完整性：子表的值必须引用父表存在的主键 |
| `CHECK` | 自定义条件，如 `CHECK (price > 0)` |
| `DEFAULT` | 插入时不指定则使用默认值 |

**练习：** 设计 `orders` 和 `order_items` 两张表，建立外键关系：

```
orders (id, user_id → users.id, total, status, created_at)
order_items (id, order_id → orders.id, product_id → products.id, quantity, unit_price)
```

#### Day 5：INSERT / SELECT 基础

| 操作 | 语法 |
|------|------|
| INSERT 单行 | `INSERT INTO t (c1, c2) VALUES (v1, v2);` |
| INSERT 多行 | `INSERT INTO t (c1, c2) VALUES (v1, v2), (v3, v4);` |
| SELECT 基础 | `SELECT c1, c2 FROM t;` |
| 列别名 | `SELECT c1 AS alias FROM t;` |
| 去重 | `SELECT DISTINCT category FROM products;` |
| 表达式 | `SELECT name, price * 1.1 AS tax_included FROM products;` |

**练习：** 向 `products` 表插入 20 条商品数据，练习 SELECT 的各种变体

#### Day 6：WHERE / ORDER BY / LIMIT

| 子句 | 说明 |
|------|------|
| `WHERE` | 行过滤：`=` / `!=` / `>` / `<` / `BETWEEN` / `IN` / `LIKE` / `IS NULL` |
| `AND` / `OR` | 组合条件，`()` 控制优先级 |
| `LIKE` | 模糊匹配：`%` (任意字符) / `_` (单个字符) |
| `ORDER BY` | `ASC` (默认) / `DESC`，支持多列 |
| `LIMIT` / `OFFSET` | 分页：`LIMIT 10 OFFSET 20` |

**练习：**
```sql
-- 价格在 50-100 之间的商品，按价格降序，取前5
SELECT name, price FROM shop.products
WHERE price BETWEEN 50 AND 100
ORDER BY price DESC
LIMIT 5;

-- 名称包含 "Pro" 的商品
SELECT * FROM shop.products WHERE name LIKE '%Pro%';
```

#### Day 7：UPDATE / DELETE / 第一周综合

| 操作 | 语法 | 注意 |
|------|------|------|
| UPDATE | `UPDATE t SET c1=v1 WHERE condition;` | **永远先写 WHERE！** |
| DELETE | `DELETE FROM t WHERE condition;` | **永远先写 WHERE！** |

**综合练习：** 为一个小书店设计数据库并操作：

```
表结构：
├── authors (id, name, bio)
├── books (id, title, author_id→authors.id, price, stock, published_date)
├── customers (id, name, email, phone)
└── sales (id, book_id→books.id, customer_id→customers.id, quantity, sale_date)

要求：
1. 创建上述表，设置正确的约束和外键
2. 插入至少 5 位作者、10 本书、5 位顾客、15 条销售记录
3. 更新某本书的价格（注意 WHERE）
4. 删除库存为 0 的书籍
5. 查询销售额最高的 3 本书
```

---

### 第2周：查询进阶

**目标：掌握 JOIN、聚合、子查询、窗口函数**

#### Day 8：INNER JOIN

| 主题 | 说明 |
|------|------|
| 为什么需要 JOIN | 数据分散在多表中，查询时需要关联 |
| INNER JOIN 语法 | `FROM a INNER JOIN b ON a.id = b.a_id` |
| 只返回匹配行 | 两表都有关联值的行才返回 |
| 多表 JOIN | 连续 JOIN，写清楚每个 ON 条件 |

**练习：**
```sql
-- 查询每本书的书名和作者名
SELECT b.title, a.name AS author
FROM books b
INNER JOIN authors a ON b.author_id = a.id
ORDER BY b.title;

-- 查询每笔销售的顾客名和书名
SELECT c.name, b.title, s.quantity, s.sale_date
FROM sales s
JOIN books b ON s.book_id = b.id
JOIN customers c ON s.customer_id = c.id;
```

#### Day 9：LEFT / RIGHT / FULL OUTER JOIN

| JOIN 类型 | 说明 |
|-----------|------|
| LEFT JOIN | 保留左表所有行，右表无匹配 → NULL |
| RIGHT JOIN | 保留右表所有行（通常改写为 LEFT JOIN） |
| FULL JOIN | 保留两表所有行 |
| 自 JOIN | 表自己 JOIN 自己，用别名区分 |

**练习：**
```sql
-- 找出还没有任何销售记录的书
SELECT b.title
FROM books b
LEFT JOIN sales s ON b.id = s.book_id
WHERE s.id IS NULL;

-- 找出所有买了书的顾客和没买过书的顾客
SELECT c.name, COUNT(s.id) AS purchase_count
FROM customers c
LEFT JOIN sales s ON c.id = s.customer_id
GROUP BY c.name;
```

#### Day 10：GROUP BY 与聚合函数

| 函数 | 说明 |
|------|------|
| `COUNT()` | `COUNT(*)` / `COUNT(column)` / `COUNT(DISTINCT col)` |
| `SUM()` / `AVG()` | 求和 / 平均值 |
| `MIN()` / `MAX()` | 最小 / 最大值 |
| `GROUP BY` | 按指定列分组聚合 |
| `HAVING` | 聚合后的过滤（WHERE 不行） |

**练习：**
```sql
-- 每位作者写了多少本书（按数量降序）
SELECT a.name, COUNT(b.id) AS book_count
FROM authors a
LEFT JOIN books b ON a.id = b.author_id
GROUP BY a.name
ORDER BY book_count DESC;

-- 总销售额超过 500 的顾客
SELECT c.name, SUM(s.quantity * b.price) AS total_spent
FROM customers c
JOIN sales s ON c.id = s.customer_id
JOIN books b ON s.book_id = b.id
GROUP BY c.name
HAVING SUM(s.quantity * b.price) > 500;
```

#### Day 11：子查询

| 类型 | 说明 | 示例 |
|------|------|------|
| 标量子查询 | 返回单值 | `WHERE price > (SELECT AVG(price) FROM books)` |
| IN / NOT IN | 检查是否在结果集中 | `WHERE id IN (SELECT book_id FROM sales)` |
| EXISTS / NOT EXISTS | 检查子查询是否返回行（通常比 IN 快） | `WHERE EXISTS (SELECT 1 FROM sales WHERE ...)` |
| 派生表 | 子查询在 FROM 中，作为临时表 | `FROM (SELECT ...) AS sub` |

**练习：**
```sql
-- 价格高于平均价的书籍
SELECT title, price FROM books
WHERE price > (SELECT AVG(price) FROM books);

-- 至少被销售过的书籍（用 EXISTS）
SELECT title FROM books b
WHERE EXISTS (SELECT 1 FROM sales s WHERE s.book_id = b.id);
```

#### Day 12：CTE（公用表表达式）

| 主题 | 说明 |
|------|------|
| 语法 | `WITH name AS (query) SELECT ... FROM name` |
| 链式 CTE | 多个 CTE 用逗号分隔，后面的可以引用前面的 |
| 递归 CTE | `WITH RECURSIVE` —— 处理树/图结构 |
| CTE vs 子查询 | CTE 可读性更好，可复用 |

**练习：**
```sql
-- 用 CTE 找出销量最高的书籍
WITH book_sales AS (
    SELECT book_id, SUM(quantity) AS total_sold
    FROM sales
    GROUP BY book_id
)
SELECT b.title, bs.total_sold
FROM books b
JOIN book_sales bs ON b.id = bs.book_id
ORDER BY bs.total_sold DESC
LIMIT 5;

-- 递归 CTE：生成 1 到 100 的数字序列
WITH RECURSIVE numbers(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 100
)
SELECT n FROM numbers;
```

#### Day 13：窗口函数

| 函数 | 说明 |
|------|------|
| `ROW_NUMBER()` | 行号（1, 2, 3...） |
| `RANK()` | 排名（有间隔：1, 1, 3...） |
| `DENSE_RANK()` | 排名（无间隔：1, 1, 2...） |
| `LAG()` / `LEAD()` | 前一行 / 后一行的值 |
| `SUM() OVER (...)` | 累计求和 / 移动平均 |
| `PARTITION BY` | 窗口分区 |
| `ORDER BY` | 窗口内排序 |

**练习：**
```sql
-- 每本书的销售额排名
SELECT b.title,
       SUM(s.quantity * b.price) AS revenue,
       RANK() OVER (ORDER BY SUM(s.quantity * b.price) DESC) AS rank
FROM books b
JOIN sales s ON b.id = s.book_id
GROUP BY b.id;

-- 按月累计销售数量
SELECT DATE_TRUNC('month', sale_date) AS month,
       SUM(quantity) AS monthly_qty,
       SUM(SUM(quantity)) OVER (ORDER BY DATE_TRUNC('month', sale_date)) AS cumulative
FROM sales
GROUP BY DATE_TRUNC('month', sale_date);
```

#### Day 14：第二周综合练习

基于第一周的书店数据库，完成以下复杂查询：

```
1. 用 CTE + 窗口函数找出每月销售额排名前 3 的书籍
2. 计算每位顾客的购买频率（两次购买之间的平均天数）
3. 找出"同时买了书 A 和书 B"的顾客组合
4. 用递归 CTE 展示书籍分类的层级关系（如果加上 category 表）
5. 生成一份月度销售报告：总销售额、增长率、Top 5 畅销书
```

---

### 第3周：设计与优化

**目标：理解索引、规范化、事务、能看懂查询计划**

#### Day 15：索引原理

| 主题 | 说明 |
|------|------|
| 没有索引时 | 全表扫描（Seq Scan），O(n) |
| B-Tree 索引 | 平衡多路搜索树，O(log n)，适合等值和范围查询 |
| 索引开销 | 写操作（INSERT/UPDATE/DELETE）需要维护索引 |
| 哪些列需要索引 | WHERE / JOIN / ORDER BY 经常出现的列 |

**练习：**
```sql
-- 建索引前后对比
EXPLAIN ANALYZE SELECT * FROM books WHERE author_id = 3;
-- ↑ 观察 Planning Time / Execution Time 和扫描方式

CREATE INDEX idx_books_author ON books(author_id);

EXPLAIN ANALYZE SELECT * FROM books WHERE author_id = 3;
-- ↑ 再对比，应该从 Seq Scan 变成 Index Scan
```

#### Day 16：索引类型与策略

| 索引类型 | 适用场景 |
|----------|----------|
| B-Tree（默认） | 等值、范围、排序 |
| Hash | 仅等值查询（PG 10+ 才有 WAL 日志） |
| GIN | 全文搜索、数组、JSONB |
| GiST | 几何数据、全文搜索 |
| 复合索引 | `CREATE INDEX ON t (a, b)` —— 注意列顺序（"最左前缀"） |
| 部分索引 | `WHERE active = true` —— 只索引活跃记录 |
| 覆盖索引 | `INCLUDE (col)` —— 避免回表 |

**练习：** 为你数据库中的核心查询创建合适的索引，用 `EXPLAIN` 验证是否生效

#### Day 17：范式化

| 范式 | 规则 | 示例 |
|------|------|------|
| 1NF | 列不可再分，无重复组 | 不要在一列存 `"张三,李四"` |
| 2NF | 满足 1NF，非主键列完全依赖主键 | 复合主键时不部分依赖 |
| 3NF | 满足 2NF，无传递依赖 | 派生字段不放表中：不要同时存 `birth_date` 和 `age` |

**练习：** 把下面的 CSV 风格数据拆成满足 3NF 的表结构：

```
订单号, 日期, 客户名, 客户电话, 商品1, 数量1, 单价1, 商品2, 数量2, 单价2
```

#### Day 18：反范式化 — 什么时候该"违反"范式

| 场景 | 说明 |
|------|------|
| 缓存字段 | 统计值（如 `order_count`）冗余在表中，避免每次 COUNT |
| 历史快照 | `order_items.unit_price` 存成交价，而不是 JOIN 查最新价 |
| 报表表 | 预聚合宽表，牺牲空间换查询速度 |

**练习：** 在你的书店模型中有意做一处反范式化（如在 `books` 表增加 `sales_count` 列），写一个触发器在插入销售时自动更新它

#### Day 19：事务与隔离级别

| 概念 | 说明 |
|------|------|
| ACID | Atomicity / Consistency / Isolation / Durability |
| `BEGIN` / `COMMIT` / `ROLLBACK` | 事务控制 |
| 脏读 / 不可重复读 / 幻读 | 并发的三种问题 |
| 隔离级别 | READ UNCOMMITTED / READ COMMITTED(默认) / REPEATABLE READ / SERIALIZABLE |
| MVCC | PostgreSQL 用多版本并发控制实现隔离 |

**练习：** 开两个 psql 终端，模拟以下场景：

```
Terminal A: BEGIN; UPDATE books SET stock = stock - 1 WHERE id = 1;
Terminal B: BEGIN; SELECT stock FROM books WHERE id = 1;
-- B 看到什么？（取决于隔离级别）
Terminal A: COMMIT;
Terminal B: SELECT stock FROM books WHERE id = 1;
-- B 现在看到什么？
```

#### Day 20：EXPLAIN 查询计划

| 主题 | 说明 |
|------|------|
| `EXPLAIN` | 显示执行计划，不实际执行 |
| `EXPLAIN ANALYZE` | 实际执行并显示真实耗时 |
| 关键指标 | Seq Scan / Index Scan / Nested Loop / Hash Join / Merge Join |
| `cost` | `cost=0.00..12.50` = 启动成本..总成本（单位：磁盘页） |
| `rows` | 估算返回行数（不准确时考虑 `ANALYZE tbl`） |
| `buffers` | `EXPLAIN (ANALYZE, BUFFERS)` 查看缓存命中率 |

**练习：** 找一个慢查询，用 EXPLAIN ANALYZE 分析，尝试加索引、改写 SQL、或者调整表结构来优化

#### Day 21：第三周综合练习

回到你的书店数据，完成以下优化任务：

```
1. 分析每张表的大小和行数
2. 为所有外键列和外键关联的查询列建立索引
3. 用 EXPLAIN ANALYZE 分析 5 个高频查询
4. 对至少一个查询做优化（加索引 / 改写 SQL）
5. 评估需要做反范式化的场景，实现一处
6. 验证不同隔离级别下的并发行为
```

---

### 第4周：Python + 实战

**目标：用 Python 操作 PostgreSQL，掌握 ORM 和 Migration，完成综合项目**

#### Day 22：psycopg2 — Python 直连 PostgreSQL

| 主题 | 说明 |
|------|------|
| 安装 | `pip install psycopg2-binary` |
| Connection | `psycopg2.connect(host, port, dbname, user, password)` |
| Cursor | `conn.cursor()` — 执行 SQL 和获取结果 |
| 参数化查询 | `cur.execute("SELECT * FROM t WHERE id=%s", (val,))` —— **防 SQL 注入** |
| 上下文管理器 | `with conn, conn.cursor() as cur:` |

**练习：**
```python
import psycopg2

conn = psycopg2.connect(
    host="localhost", port=5432,
    dbname="learn", user="postgres", password="secret"
)

def get_books_by_author(author_name: str):
    with conn, conn.cursor() as cur:
        cur.execute("""
            SELECT b.title, b.price
            FROM books b
            JOIN authors a ON b.author_id = a.id
            WHERE a.name = %s
        """, (author_name,))
        return cur.fetchall()

print(get_books_by_author("George Orwell"))
```

#### Day 23：SQLAlchemy Core

| 主题 | 说明 |
|------|------|
| Engine | `create_engine("postgresql+psycopg2://...", echo=True)` |
| MetaData / Table | 反射式定义表结构 |
| 表达式构建 | `select([t.c.name]).where(t.c.price > 50)` |
| 事务 | `with engine.begin() as conn:` |

**练习：** 用 SQLAlchemy Core 重写 Day 22 的查询，对比与原生 SQL 的差异

#### Day 24：SQLAlchemy ORM

| 主题 | 说明 |
|------|------|
| Declarative Base | `class Book(Base): ...` 用类映射表 |
| Session | `sessionmaker` / `scoped_session` |
| 关系 | `relationship("Author", back_populates="books")` |
| 级联 | `cascade="all, delete-orphan"` |

**练习：** 用 ORM 模型定义书店的全部表，实现增删改查：

```python
from sqlalchemy import create_engine, Column, Integer, String, Float, ForeignKey
from sqlalchemy.orm import DeclarativeBase, relationship, Session

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id = Column(Integer, primary_key=True)
    name = Column(String(200), nullable=False)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id = Column(Integer, primary_key=True)
    title = Column(String(300), nullable=False)
    price = Column(Float)
    author_id = Column(Integer, ForeignKey("authors.id"))
    author = relationship("Author", back_populates="books")

# 使用
engine = create_engine("postgresql+psycopg2://postgres:secret@localhost:5432/learn")
Base.metadata.create_all(engine)

with Session(engine) as session:
    # 新增
    author = Author(name="J.K. Rowling")
    session.add(author)
    session.commit()

    # 查询
    books = session.query(Book).filter(Book.price > 50).all()
    for b in books:
        print(b.title, b.author.name)
```

#### Day 25：Alembic — 数据库迁移

| 主题 | 说明 |
|------|------|
| 为什么需要 Migration | 版本控制数据库 schema，团队协作 |
| `alembic init` | 初始化迁移环境 |
| `alembic revision --autogenerate` | 根据模型变更自动生成迁移脚本 |
| `alembic upgrade head` | 执行迁移到最新版本 |
| `alembic downgrade -1` | 回滚一步 |

**练习：** 给书店模型添加 `publisher` 表，用 Alembic 生成迁移并执行：

```bash
pip install alembic
alembic init alembic
# 编辑 alembic/env.py 配置数据库连接
# 编辑 alembic.ini 设置 sqlalchemy.url
alembic revision --autogenerate -m "add publisher table"
alembic upgrade head
```

#### Day 26：备份与恢复

| 工具 | 用途 |
|------|------|
| `pg_dump` | 导出单库：`pg_dump -U postgres learn > backup.sql` |
| `pg_dumpall` | 导出全部数据库（含角色） |
| `pg_restore` | 从自定义格式恢复 |
| `psql < backup.sql` | 从 SQL 文件恢复 |
| `COPY` | 快速的 CSV 导入导出 |
| WAL 归档 | 时间点恢复（PITR） |

**练习：** 用 Docker Compose 编排一个带自动备份的 PostgreSQL：

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
  backup:
    image: postgres:16-alpine
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        while true; do
          PGPASSWORD=secret pg_dump -h db -U postgres learn > /backups/backup_$(date +%Y%m%d_%H%M%S).sql
          sleep 3600
        done
    volumes:
      - ./backups:/backups

volumes:
  pgdata:
```

#### Day 27-28：第四周综合项目 — 电商数据库设计

设计并实现一个简易商城数据库：

```
需求：
├── 商品管理：多分类、品牌、SKU（规格变体）、库存
├── 用户系统：注册、地址、会员等级
├── 购物车：加购、修改数量、过期清理
├── 订单系统：下单、支付状态、发货
└── 评价系统：评分、评论、图片

要求：
1. 设计 ER 图，标注所有表和关系
2. 编写建表 SQL（至少 10 张表），包括约束和索引
3. 用 SQLAlchemy ORM 建立 Python 模型
4. 用 Alembic 管理数据库版本
5. 编写业务查询：
   - 用户下单流程（事务：锁库存 → 创建订单 → 清空购物车）
   - 商品搜索（多条件 + 分页 + 排序）
   - 用户购买历史与个性化推荐（买了 X 的人也买了 Y）
   - 销售统计仪表盘（日/周/月销售额、热销排行、转化率）
6. 用 Python 脚本生成 10 万条测试数据，验证查询性能
```

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能设计表结构，写出基本的 CRUD SQL
Week 2 结束：✓ 能写 JOIN、子查询、CTE、窗口函数的复杂查询
Week 3 结束：✓ 能设计索引、分析执行计划、理解事务和范式
Week 4 结束：✓ 能用 Python + SQLAlchemy + Alembic 管理数据库，完成电商项目
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 官方文档 | [postgresql.org/docs](https://www.postgresql.org/docs/current/) —— 最权威的 SQL 参考 |
| 在线练习 | [pgexercises.com](https://pgexercises.com/) —— PostgreSQL 交互式练习 |
| 免费课程 | [SQL for Data Analysis (Mode)](https://mode.com/sql-tutorial) —— 面向查询的教程 |
| 书籍 | 《SQL 必知必会》(Sams Teach Yourself SQL in 10 Minutes) —— 经典入门 |
| 书籍 | 《高性能 PostgreSQL》(PG 原理与优化进阶) |
| 工具 | DBeaver / pgAdmin —— GUI 管理工具 |
| 工具 | [dbdiagram.io](https://dbdiagram.io/) —— 在线 ER 图绘制 |
| 浏览器工具 | [sqliteonline.com](https://sqliteonline.com/) —— 在线 SQL 练习（支持 PostgreSQL 语法） |
