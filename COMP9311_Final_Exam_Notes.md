## 0. 题型总览与答题策略

### 0.1 高频模块

从样题看，9311 期末的稳定结构大致是：

| 模块 | 常见题型 | 核心能力 |
|---|---|---|
| 概念题 / True-False / short answer | SQL、ER 属性、ACID、index、buffer、normal form 判断 | 用短句准确说明，不写无关内容 |
| ER modelling | 根据文字需求画 ER 图，并说明 assumptions | 抓实体、属性、关系、cardinality、participation、弱实体、多值/复合/派生属性 |
| ER → relational schema | 把 ER 图或文字模型转成 relations | PK、FK、junction table、弱实体 key、多值属性拆表 |
| Relational Algebra / SQL | all / none / only / most / same / not / recommend / available | set difference、division、grouping、self-join、anti-join、date overlap |
| Functional Dependencies | closure、entailment、candidate keys、minimal cover | 机械算法，步骤要完整 |
| Normalisation / decomposition | highest NF、3NF、BCNF、lossless、dependency preserving | 用 key、prime attributes、FD projection、chase/binary test |
| Transactions | conflict serializability、wait-for graph、deadlock、2PL | 画图，不靠直觉 |
| Recovery | log analysis、redo/undo、checkpoint | committed / active / before checkpoint 分类 |
| Buffer / storage / index | LRU/MRU/FIFO、hit rate、hash vs range、ISAM | 手动模拟，说明原因 |

> 建议复习顺序：先看 **FD/范式/分解** 和 **关系代数/SQL**，再看 **ER 建模**，最后看 **并发、恢复、buffer、概念题**。

### 0.2 考试写法原则

这门课的题通常不是要求长篇解释，而是要求**结论 + 可验证的推导**。比较好的答案结构是：

1. 先定义你要用的中间对象，例如 closure、集合、关系代数中间表。
2. 再给出计算步骤。
3. 最后给出结论，并用一句话说明为什么。

不要只写“显然”。例如：

不够好的写法：

> `BD` is a key because it determines many attributes.

更安全的写法：

\[
(BDGJ)^+ = \{A,B,C,D,E,G,H,I,J\}
\]

且 `BDGJ` 的任一真子集都不能推出全部属性，因此 `BDGJ` 是 candidate key。


---

## Q1. 概念题与短答题

### 1.1 True/False 常见判断

| 命题类型 | 正确判断 | 说明模板 |
|---|---:|---|
| PostgreSQL 不允许 self-defined data type | False | PostgreSQL 支持 user-defined types，例如 composite type、domain、enum 等。 |
| `AS` 会改变数据库 | False | `AS` 只是给输出列或表起 alias，不修改 stored data 或 schema。 |
| BCNF 一定也是 2NF | True | BCNF 比 3NF 更强，而 3NF 在通常定义下排除了 2NF 中的 partial dependency 问题。 |
| primary key 必须只有一个 attribute | False | primary key 可以是 composite key。 |
| lossless and dependency-preserving 3NF decomposition 不一定存在 | False | 对任意关系，都存在 lossless 且 dependency-preserving 的 3NF synthesis。 |
| hash index 适合 range search | False | hash index 适合 equality search；range search 通常用 ordered index，如 B+ tree。 |
| ISAM 是 dynamic structure | False | ISAM 主要是 static indexed sequential structure；动态插入通常依赖 overflow pages。 |
| optimistic concurrency control 适合高冲突环境 | False | optimistic control 适合冲突少的场景；冲突多时 validation failure 多，重启成本高。 |

答 True/False 时，如果题目要求 “true or false only”，就不要解释。若要求 explain，则写一到两句话，不要扩展到无关知识。

### 1.2 Simple attribute vs composite attribute

**Simple attribute** 不能再被有意义地拆分。例如 `age`、`salary`、`email`。  
**Composite attribute** 可以拆成更小的 component。例如 `address = (street, suburb, postcode)`，`dimensions = (height, width, length)`。

为什么需要 composite attribute：

- 它保留现实语义，ER 图更清楚。
- 查询时可以只使用一部分，例如只按 `postcode` 检索。
- 转关系模式时通常会把 composite attribute 拆成 component attributes，而不是存成一个不可分字符串。

考试表达可以写：

> A simple attribute is atomic in the conceptual model, while a composite attribute consists of smaller meaningful components. Composite attributes are useful when the database needs to access or constrain individual components separately.

### 1.3 Multivalued attribute vs repeated normal attribute

如果题目说：

- “a lab can be held in multiple weeks”
- “a painting has multiple artists”
- “a restaurant can have multiple phone numbers”
- “a book can have multiple authors”

这通常不是在同一个 relation 里放 `week1, week2, week3` 或 `author1, author2`。更合适是：

\[
LabWeek(\underline{labID, week})
\]

\[
PaintingArtist(\underline{paintingID, artistName})
\]

这样符合 1NF，也方便查询和扩展。

### 1.4 Derived attribute

题目中的这些通常是 derived attribute：

- total number of students enrolled in a course
- number of days elapsed since issue date
- total number of books in an order
- number of followers / followees
- number of likes

画 ER 图时可以标成 derived attribute；转关系模式时有两种写法：

1. 不存，运行查询时计算。
2. 存为冗余字段，但需要说明由其他关系维护。

考试更安全的 assumption：

> I treat this as a derived attribute, since it can be computed from the corresponding relationship instances.

### 1.5 NATURAL JOIN vs Cartesian Product

题目常问：

```sql
SELECT A
FROM R1 NATURAL JOIN R2;

SELECT A
FROM R1, R2;
```

关键点：

`NATURAL JOIN` 会自动在两个表中**同名属性**上做等值连接，并且同名列只保留一份。  
`FROM R1, R2` 是 Cartesian product，没有 join condition。

它们不一定总是不同：

- 如果 `R1` 和 `R2` 没有共同属性名，`NATURAL JOIN` 等价于 Cartesian product。
- 如果有共同属性名，`NATURAL JOIN` 会筛掉共同属性值不相等的 tuple，通常不同。
- 如果共同属性上的值恰好全部匹配，也可能结果 tuple 数量与 cross product 不同或相同，需要看数据。

答案模板：

> They do not always produce different results. If `R1` and `R2` share no common attribute names, the natural join degenerates into a Cartesian product. If they share common attributes, the natural join additionally requires equality on those attributes, so the result may differ from `FROM R1, R2`.

### 1.6 `ORDER BY DESC` vs `MAX`

题目：

```sql
SELECT A
FROM R
ORDER BY A DESC;
```

和

```sql
SELECT MAX(A)
FROM R;
```

要找最大值，最合适是 `MAX(A)`。

原因：

- `ORDER BY A DESC` 返回的是所有 `A` 值，只是排序后最大值在最前。
- 如果没有 `LIMIT 1`，它不是“只返回最大值”的查询。
- `MAX(A)` 明确返回单个 aggregate value。
- 若要返回拥有最大 `A` 的完整 tuple，则需要子查询或 `ORDER BY ... LIMIT 1`，不能只写 `MAX(A)`。

考试短答：

> The second query is more appropriate for finding the maximum value of `A`, because it directly returns the aggregate maximum. The first query returns all values of `A` in descending order, unless an additional `LIMIT 1` is used.

### 1.7 Atomicity in ACID

Atomicity 的意思是 transaction 内部操作要么全部生效，要么全部不生效。

例子：

> 转账：从账户 A 扣 100，同时给账户 B 加 100。Atomicity 要求两个更新要么一起 commit，要么一起 rollback。

反例：

> 系统在 A 已经扣款但 B 尚未加款时 crash，恢复后只保留扣款结果。这违反 atomicity。

不要把 atomicity 写成 isolation。Isolation 是并发事务之间互不干扰；atomicity 是单个事务内部不可拆分。

---

## Q2. ER Modelling 与 Relational Schema Translation

### 2.1 ER 题的整体流程

做 ER 题时，不要一边读一边乱画。建议按下面顺序：

1. 圈出名词：通常是 entity type，例如 `Student`、`Course`、`Lab`、`Gallery`、`Painting`、`Staff`。
2. 找唯一标识：`uniquely identified by ...` 通常是 primary key。
3. 找属性：普通属性、复合属性、多值属性、派生属性分开标。
4. 找动词和约束：`enrol in`、`belong to`、`look after`、`managed by`、`contains` 通常是 relationship。
5. 标 cardinality 和 participation：zero or more、exactly one、at least one、one or more。
6. 找 weak entity：局部唯一编号，例如 document number only unique within painting。
7. 写 assumptions：尤其是题目没有明确的地方，例如 lab enrolment 是否必须对应同一 course enrolment。

### 2.2 Cardinality 文字翻译表

| 题目文字 | ER 含义 |
|---|---|
| A can have zero or more B | A 到 B 是 0..* |
| A must have at least one B | A 到 B 是 1..*，A total participation |
| Each B belongs to exactly one A | B 到 A 是 1..1，B total participation |
| A is not required to have B | A 到 B 是 0..* |
| Each order must contain one or more dishes | Order 对 Contains 是 total participation，且 1..* |
| A staff must look after one or more galleries | Staff 对 LookAfter 是 total participation，且 1..* |
| A gallery must be looked after by at least one staff | Gallery 对 LookAfter 是 total participation，且 1..* |

### 2.3 Attribute 类型处理

| 文字线索 | ER 表示 | Relational schema 处理 |
|---|---|---|
| location consists of building and room | composite attribute | `building`, `room` 两列 |
| dimensions consists of height, width, length | composite attribute | `height`, `width`, `length` 三列 |
| lab can be held in multiple weeks | multivalued attribute | 新表 `LabWeek(labID, week)` |
| restaurant has multiple phone numbers | multivalued attribute | 新表 `RestaurantPhone(rID, phone)` |
| number of days elapsed | derived attribute | 通常不存，或标注 derived |
| total number of students | derived aggregate | 从 `Enrol` 关系计算 |

### 2.4 Weak entity

弱实体的典型线索：

> document number is only unique between documents belonging to the same painting.

这说明 `SupportingDocument` 不是全局由 `docNo` 唯一标识，而是由 owner entity 的 key 加 partial key 唯一标识：

\[
SupportingDocument(\underline{paintingID, docNo}, rating, condition, reportYear)
\]

其中 `paintingID` 同时是 foreign key，引用：

\[
Painting(\underline{paintingID}, title, rarity, ...)
\]

### 2.5 ER → Relational Schema 规则

#### 规则 1：Strong entity

每个 strong entity 变一个 relation：

\[
Student(\underline{zID}, name, contactNo, email, currentDegree)
\]

\[
Course(\underline{courseID}, courseName, lecturerName, totalCapacity)
\]

如果有 composite attribute，把 component 放入 relation：

\[
Lab(\underline{labID}, time, building, room, courseID)
\]

#### 规则 2：Multivalued attribute

多值属性单独建表：

\[
LabWeek(\underline{labID, week})
\]

\[
PaintingArtist(\underline{paintingID, artistName})
\]

\[
RestaurantPhone(\underline{restaurantID, phoneNo})
\]

#### 规则 3：1:N relationship

把 `1` 端的 PK 放到 `N` 端作为 FK。

例如每个 `Lab` 属于 exactly one `Course`：

\[
Lab(\underline{labID}, time, building, room, courseID)
\]

其中 `courseID` 是 FK → `Course(courseID)`。  
如果 Lab 必须属于 Course，则 `courseID` not null。

#### 规则 4：M:N relationship

单独建 junction relation，PK 通常是两个 entity keys 的组合。

例如学生选课：

\[
CourseEnrolment(\underline{zID, courseID})
\]

学生选 lab：

\[
LabEnrolment(\underline{zID, labID})
\]

tutor 在 lab on duty：

\[
LabDuty(\underline{tutorID, labID})
\]

#### 规则 5：Relationship 有自己的 attribute

如果 relationship 有属性，例如 order contains dish and records quantity：

\[
OrderItem(\underline{orderID, dishID}, quantity)
\]

如果 quantity 是 order 的总 dish 数，通常是 derived；如果是每种 dish 的数量，则属于 relationship。

#### 规则 6：Weak entity

弱实体 relation 包含 owner key + partial key：

\[
SupportingDocument(\underline{paintingID, docNo}, rating, condition, reportYear)
\]

#### 规则 7：Recursive relationship

例如 user follows user：

\[
Follow(\underline{followerID, followeeID})
\]

两个属性都 FK → `User(userID)`，但 role name 不同。

#### 规则 8：Either user or group receives message

若题目说 message is received by another user or a group，有两种 reasonable design：

方案 A：两个 nullable FK，并加 constraint：

\[
Message(\underline{msgID}, senderID, receiverUserID, receiverGroupID, content, timestamp)
\]

并说明 exactly one of `receiverUserID`, `receiverGroupID` is non-null。

方案 B：拆成两个 relationship table：

\[
UserMessage(\underline{msgID}, receiverUserID)
\]

\[
GroupMessage(\underline{msgID}, groupID)
\]

考试中任选一种，但 assumption 要写清楚。

### 2.6 Course-Enrolment 典型关系模式

题目中 Course、Lab、Student、LabTutor 的常见翻译可以写成：

\[
Course(\underline{courseID}, courseName, lecturerName, totalCapacity)
\]

\[
Lab(\underline{labID}, time, building, room, courseID)
\]

\[
LabWeek(\underline{labID, week})
\]

\[
Student(\underline{zID}, name, contactNo, email, currentDegree)
\]

\[
LabTutor(\underline{tutorID}, name, contactNo, email, salary)
\]

\[
CourseEnrolment(\underline{zID, courseID})
\]

\[
LabEnrolment(\underline{zID, labID})
\]

\[
CourseTutor(\underline{courseID, tutorID})
\]

\[
LabTutorDuty(\underline{labID, tutorID})
\]

可以写的 assumptions：

1. `total number of students enrolled in a course` is derived from `CourseEnrolment`.
2. `LabWeek` is separated because a lab can be held in multiple weeks.
3. A student enrolled in a lab is assumed to be enrolled in the course that owns that lab. This can be enforced by an additional constraint but is not fully captured by simple FK notation.

### 2.7 ER 建模常见失误

1. 把多值属性塞进一个列，例如 `weeks = "1,2,3"`。这违反 1NF。
2. 忘记 weak entity 的 owner key。
3. 忘记 relationship 的 minimum participation，例如 “must have at least one”。
4. 把 derived attribute 当普通 stored attribute，却没有说明维护方式。
5. 对 M:N relationship 直接在两边互放 FK，这会导致重复或无法表达多对多。
6. 不写 assumptions。题目明确要求 “state assumptions”，没有 assumption 也可以写 “No additional assumptions.”

---

## Q3. Relational Algebra 与 SQL

### 3.1 基本符号

| 操作 | 符号 | 含义 |
|---|---|---|
| selection | \(\sigma_{\text{condition}}(R)\) | 选行 |
| projection | \(\pi_{\text{attrs}}(R)\) | 选列，并去重 |
| rename | \(\rho_{NewName(...)}(R)\) | 重命名 relation 或 attributes |
| union | \(R \cup S\) | 并集，schema 要 compatible |
| difference | \(R - S\) | 差集 |
| cartesian product | \(R \times S\) | 笛卡尔积 |
| join | \(R \bowtie_{\theta} S\) | 条件连接 |
| natural join | \(R \bowtie S\) | 同名属性等值连接 |
| division | \(R \div S\) | “for all” 查询 |
| grouping | \(\gamma_{group;\ agg}(R)\) | 聚合，若课程允许则可用 |

如果题目限制“只能用 \(\sigma,\pi,\times,\rho\)”，就不能直接写 join。把 join 改写成：

\[
R \bowtie_{R.a=S.a} S
=
\sigma_{R.a=S.a}(R \times S)
\]

### 3.2 “only” 查询模板

“only” 通常翻译成：

\[
Good - Bad
\]

例如：restaurants only reviewed by male customers or only reviewed by female customers。

设：

\[
M = \pi_{rID}(\sigma_{gender='M'}(Customer \bowtie Review))
\]

\[
F = \pi_{rID}(\sigma_{gender='F'}(Customer \bowtie Review))
\]

则：

\[
Answer = (M - F) \cup (F - M)
\]

注意：这个写法只返回至少被某一性别 review 过的 restaurant。没有任何 review 的 restaurant 不应算作 “only reviewed by male/female”。

### 3.3 “none” 查询模板

“never reviewed any restaurant”：

\[
AllCustomers = \pi_{cID}(Customer)
\]

\[
Reviewed = \pi_{cID}(Review)
\]

\[
Answer = AllCustomers - Reviewed
\]

SQL：

```sql
SELECT c.cID
FROM Customer c
WHERE NOT EXISTS (
  SELECT 1
  FROM Review r
  WHERE r.cID = c.cID
);
```

### 3.4 “all” 查询模板：division

“customers who reviewed all restaurants”：

\[
ReviewedPairs = \pi_{cID,rID}(Review)
\]

\[
AllRestaurants = \pi_{rID}(Restaurant)
\]

\[
Answer = ReviewedPairs \div AllRestaurants
\]

不用 division 的写法：

\[
AllPairs = \pi_{cID}(Customer) \times \pi_{rID}(Restaurant)
\]

\[
Missing = AllPairs - \pi_{cID,rID}(Review)
\]

\[
BadCustomers = \pi_{cID}(Missing)
\]

\[
Answer = \pi_{cID}(Customer) - BadCustomers
\]

SQL 常用双重 `NOT EXISTS`：

```sql
SELECT c.cID
FROM Customer c
WHERE NOT EXISTS (
  SELECT 1
  FROM Restaurant rest
  WHERE NOT EXISTS (
    SELECT 1
    FROM Review rev
    WHERE rev.cID = c.cID
      AND rev.rID = rest.rID
  )
);
```

### 3.5 “all or none” 查询模板

“reviewed all restaurants or never reviewed any restaurant”：

\[
AllReviewers = \pi_{cID,rID}(Review) \div \pi_{rID}(Restaurant)
\]

\[
NoReview = \pi_{cID}(Customer) - \pi_{cID}(Review)
\]

\[
Answer = AllReviewers \cup NoReview
\]

这类题常见，建议直接背模板。

### 3.6 “most / highest / greatest” 查询模板

#### SQL 聚合写法

“movies with the greatest number of views”：

```sql
WITH ViewCount AS (
  SELECT mid, COUNT(*) AS cnt
  FROM watched
  GROUP BY mid
),
MaxCount AS (
  SELECT MAX(cnt) AS max_cnt
  FROM ViewCount
)
SELECT vc.mid
FROM ViewCount vc, MaxCount mc
WHERE vc.cnt = mc.max_cnt;
```

注意：不能用 `ORDER BY cnt DESC` 后只说第一行，除非题目允许 `LIMIT`，且要处理 tie。

#### Relational algebra 反支配写法

如果有 relation：

\[
Count(mid, cnt)
\]

找最大值可以用 self-join 找出非最大：

\[
C_1 = \rho_{C1(mid,cnt)}(Count)
\]

\[
C_2 = \rho_{C2(mid,cnt)}(Count)
\]

\[
NotMax = \pi_{C1.mid}(\sigma_{C1.cnt < C2.cnt}(C_1 \times C_2))
\]

\[
Answer = \pi_{mid}(Count) - NotMax
\]

这个模板也适用于 highest price、oldest age、max rating。

### 3.7 “same name at least twice” 模板

题目：customers who watched movies with the same name at least twice。

思路：同一个 customer，两个不同 movie ID，但 movie name 相同。

\[
W_1 = \rho_{W1}(watched)
\]

\[
W_2 = \rho_{W2}(watched)
\]

\[
M_1 = \rho_{M1}(movie)
\]

\[
M_2 = \rho_{M2}(movie)
\]

\[
Answer =
\pi_{W1.cid}
(
\sigma_{
W1.cid=W2.cid
\land W1.mid \ne W2.mid
\land W1.mid=M1.mid
\land W2.mid=M2.mid
\land M1.name=M2.name
}
(W_1 \times W_2 \times M_1 \times M_2)
)
\]

如果 `watched` 的 key 包含 `(cid, mid, year)`，同一人同一年同电影不会重复，但同名不同电影要通过 `mid != mid` 保证。

### 3.8 “either A or B but not both” 模板

题目：parks visited by either “Daniel” or “James”, but not both.

\[
D = \pi_{pID}(\sigma_{name='Daniel'}(Visitor \bowtie Visit))
\]

\[
J = \pi_{pID}(\sigma_{name='James'}(Visitor \bowtie Visit))
\]

\[
Answer = (D - J) \cup (J - D)
\]

这是 symmetric difference。

### 3.9 “not in category” 模板

题目：drugs not in antihypertensive category。

\[
AllDrugs = \pi_{dID,dname}(Drug)
\]

\[
Bad = \pi_{dID,dname}(Drug \bowtie \sigma_{category='antihypertensive'}(Category))
\]

\[
Answer = AllDrugs - Bad
\]

SQL：

```sql
SELECT d.dname
FROM Drug d
WHERE NOT EXISTS (
  SELECT 1
  FROM Category c
  WHERE c.dID = d.dID
    AND c.category = 'antihypertensive'
);
```

注意不要写：

```sql
WHERE category <> 'antihypertensive'
```

因为一个 drug 可以有多个 category。某个 drug 同时属于 `antihypertensive` 和 `other` 时，`<>` 会错误保留 `other` 那一行。

### 3.10 “available room” 日期重叠模板

Room 在 `[target_in, target_out)` 可用，如果没有 booking 与目标区间重叠。

两个区间 `[a,b)` 和 `[c,d)` 重叠的条件是：

\[
a < d \land c < b
\]

因此不可用房间：

```sql
SELECT b.hid, b.r_no
FROM Booking b
WHERE b.check_in < DATE '2022-12-26'
  AND DATE '2022-12-25' < b.check_out
```

可用房间：

```sql
SELECT DISTINCT r.type
FROM Room r
JOIN Hotel h ON h.hid = r.hid
WHERE h.name = 'Sydney Hilton Hotel'
  AND NOT EXISTS (
    SELECT 1
    FROM Booking b
    WHERE b.hid = r.hid
      AND b.r_no = r.r_no
      AND b.check_in < DATE '2022-12-26'
      AND DATE '2022-12-25' < b.check_out
  );
```

注意题目说 check-out today 不算当前入住时，通常用：

\[
check\_in \le today < check\_out
\]

### 3.11 “only by qualified people” 模板

题目类型：

- movies only watched by age ≥ 30
- drugs purchased only by female customers diagnosed with Fever
- restaurants whose exclusive raters are from Greece

一般写法：

\[
TargetObjects = \text{objects with at least one relevant event}
\]

\[
BadObjects = \text{objects with at least one event by unqualified person}
\]

\[
Answer = TargetObjects - BadObjects
\]

例如 “only watched by age ≥ 30”：

\[
WatchedMovie = \pi_{mid}(watched)
\]

\[
BadMovie = \pi_{mid}(\sigma_{age<30}(customer \bowtie watched))
\]

\[
QualifiedOnly = WatchedMovie - BadMovie
\]

如果还要 “greatest number of views”，先筛 `QualifiedOnly`，再 aggregate。

### 3.12 “recommend not yet rated + highest among similar users” 模板

这类题复杂，但结构固定：

1. 为每个 target customer 找 similar users。
2. 找 similar users 给过 rating 的 restaurant。
3. 排除 target customer 已经 rated 的 restaurant。
4. 在候选中保留 highest rated。

SQL 骨架：

```sql
WITH SimilarRating AS (
  SELECT target.cID AS cID, r.rID, rt.rate
  FROM Customer target
  JOIN Customer other
    ON other.nationality = target.nationality
   AND other.DoB = target.DoB       -- 或根据题目定义 age
   AND other.cID <> target.cID
  JOIN Rating rt
    ON rt.cID = other.cID
),
UnratedCandidate AS (
  SELECT sr.*
  FROM SimilarRating sr
  WHERE NOT EXISTS (
    SELECT 1
    FROM Rating my
    WHERE my.cID = sr.cID
      AND my.rID = sr.rID
  )
),
MaxRate AS (
  SELECT cID, MAX(rate) AS max_rate
  FROM UnratedCandidate
  GROUP BY cID
)
SELECT DISTINCT u.cID, u.rID
FROM UnratedCandidate u
JOIN MaxRate m
  ON m.cID = u.cID
 AND m.max_rate = u.rate;
```

考试中这类题允许 auxiliary views，建议拆开写，不要试图一行写完。

---

## Q4. Functional Dependencies 与 Normalisation

### 4.1 Armstrong’s Axioms

三条基本规则：

| 规则 | 形式 |
|---|---|
| Reflexivity | 若 \(Y \subseteq X\)，则 \(X \to Y\) |
| Augmentation | 若 \(X \to Y\)，则 \(XZ \to YZ\) |
| Transitivity | 若 \(X \to Y\)，且 \(Y \to Z\)，则 \(X \to Z\) |

常用派生规则：

| 规则 | 形式 |
|---|---|
| Union | \(X \to Y\) 且 \(X \to Z\)，则 \(X \to YZ\) |
| Decomposition | \(X \to YZ\)，则 \(X \to Y\)，\(X \to Z\) |
| Pseudotransitivity | \(X \to Y\)，\(WY \to Z\)，则 \(WX \to Z\) |

### 4.2 FD entailment：用 closure 判断

要判断：

\[
F \models X \to Y
\]

最直接方法是算 \(X^+\)。

步骤：

1. 初始 \(X^+ = X\)。
2. 如果某个 FD \(A \to B\) 满足 \(A \subseteq X^+\)，就把 \(B\) 加入 \(X^+\)。
3. 重复直到不能再加。
4. 如果 \(Y \subseteq X^+\)，则能推出；否则不能。

例子：

\[
F = \{A \to G,\; AG \to DE,\; E \to H,\; EC \to B,\; A \to D\}
\]

判断 \(AC \to H\)。

\[
(AC)^+ = \{A,C\}
\]

由 \(A \to G\)，加入 \(G\)。  
由 \(AG \to DE\)，加入 \(D,E\)。  
由 \(E \to H\)，加入 \(H\)。  
因此 \(H \in (AC)^+\)，所以 \(F \models AC \to H\)。

### 4.3 Candidate key 算法

给定 relation \(R\) 和 FD set \(F\)，求 candidate keys：

1. 先找 **must-have attributes**：从未出现在任何 RHS 的 attributes，必须出现在每个 key 中。
2. 计算 must-have set 的 closure。
3. 如果已经覆盖全部属性，它就是唯一 candidate key。
4. 否则，从剩余属性中逐步加入属性，枚举最小组合。
5. 每找到一个 key，就不要再考虑它的 superset，因为 candidate key 必须 minimal。

### 4.4 为什么 RHS 不出现的属性必须在 key 中

若某个 attribute \(A\) 从未出现在任何 FD 的 RHS 中，那么没有任何其他属性能推出 \(A\)。  
因此任何能推出全部属性的 set 必须一开始就包含 \(A\)。

这个 observation 能显著减少枚举量。

### 4.5 Prime attribute

Prime attribute 是**出现在至少一个 candidate key 中**的 attribute。

注意：

- 不是 “primary key 中的属性”。
- 不是 “所有 candidate key 都共有的属性”。
- 只要出现在任意一个 candidate key 中，就是 prime。

Prime attribute 用于判断 3NF：

\[
X \to A
\]

若 \(X\) 不是 superkey，但 \(A\) 是 prime attribute，则这条 FD 不违反 3NF。

### 4.6 Minimal cover / Minimum cover

Minimal cover 的标准步骤：

#### Step 1：拆 RHS

把每个 FD 的 RHS 拆成单属性：

\[
A \to BC
\]

拆成：

\[
A \to B,\quad A \to C
\]

#### Step 2：删除 LHS 中的 extraneous attribute

对 \(XA \to B\)，检查 \(A\) 是否多余：

1. 用 \(F\) 计算 \(X^+\)。
2. 如果 \(B \in X^+\)，则 \(A\) 多余，可以把 \(XA \to B\) 改为 \(X \to B\)。

更形式化地：

\[
A \text{ extraneous in } XA \to B
\iff
F \models X \to B
\]

#### Step 3：删除 redundant FD

对某条 FD \(X \to A\)，把它暂时从 \(F\) 中删除。  
如果在剩余 FD 下仍然有：

\[
A \in X^+
\]

则这条 FD redundant，可以删除。

#### Step 4：合并同 LHS

有些课程允许最后把同 LHS 合并：

\[
A \to B,\quad A \to C
\]

写成：

\[
A \to BC
\]

但如果后续要做 3NF synthesis，保留单 RHS 更清楚。

### 4.7 Minimal cover 常见失误

1. 没有先拆 RHS 就检查 extraneous。
2. 删除 LHS 属性时错误地把当前 FD 完全删掉。
3. 只检查 LHS extraneous，不检查 redundant FD。
4. 把 RHS attribute 也当作可删除对象，但没有用正确方法证明。
5. 最后得到的 cover 和原 FD 不等价。

### 4.8 Lossless join：二元分解快速判断

对二元分解：

\[
R \to R_1, R_2
\]

lossless iff：

\[
(R_1 \cap R_2) \to (R_1 - R_2)
\]

或

\[
(R_1 \cap R_2) \to (R_2 - R_1)
\]

在 \(F^+\) 中成立。

做题步骤：

1. 算交集 \(S = R_1 \cap R_2\)。
2. 算 \(S^+\)。
3. 若 \(R_1 \subseteq S^+\) 或 \(R_2 \subseteq S^+\)，则 lossless。
4. 否则 lossy。

例子：

\[
R_1=\{A,B,C,D,H\},\quad R_2=\{E,G,H,I,J\}
\]

交集：

\[
S = \{H\}
\]

若 \(H^+\) 只能推出 \(A,B,H\)，无法推出 \(R_1\) 或 \(R_2\) 的所有属性，则不是 lossless。

### 4.9 Lossless join：多元分解用 chase

对于 \(R \to R_1,\ldots,R_n\)，二元交集规则不能直接套。用 tableau / chase 更安全。

步骤：

1. 建一个表，每行对应一个 \(R_i\)，每列对应原 relation 的一个 attribute。
2. 如果 attribute \(A_j\) 在 \(R_i\) 中，填同一个 symbol \(a_j\)。
3. 如果不在，填独立 symbol \(b_{ij}\)。
4. 对每个 FD \(X \to Y\)，如果两行在 \(X\) 上符号相同，就令它们在 \(Y\) 上符号相同。
5. 重复直到不变。
6. 如果某一行全部变成 \(a_1,a_2,\ldots,a_n\)，则 lossless；否则不能由 chase 证明 lossless，通常为 lossy。

考试中如果时间紧，写清楚 chase 表的关键变化即可。

### 4.10 Dependency preserving 判断

分解 \(R \to R_1,\ldots,R_n\) 是 dependency preserving，如果：

\[
(F_1 \cup F_2 \cup \cdots \cup F_n)^+ = F^+
\]

其中 \(F_i\) 是 \(F^+\) 在 \(R_i\) 上的 projection。

实用检查方法：

对原来的每条 FD \(X \to Y\)：

1. 用分解后的 projected FDs 计算 \(X^+\)。
2. 如果能推出 \(Y\)，这条 FD 被 preserved。
3. 如果有任何一条推不出，则不是 dependency preserving。

注意：projection 应该来自 \(F^+\)，不只是把原始 FDs 中 attributes 全在 \(R_i\) 的留下。但考试中若要完整严谨，需要说明这一点。

### 4.11 Normal Forms

#### 1NF

每个 attribute value atomic，没有 repeating group / list / set。

#### 2NF

在 1NF 基础上，非 prime attribute 不能 partial depend on candidate key 的真子集。

更直接：

若存在 \(X \to A\)，其中：

- \(X\) 是某个 candidate key 的真子集；
- \(A\) 是 non-prime attribute；

则违反 2NF。

#### 3NF

对每条 non-trivial FD：

\[
X \to A
\]

至少满足一个：

1. \(X\) 是 superkey；
2. \(A\) 是 prime attribute。

若都不满足，违反 3NF。

#### BCNF

对每条 non-trivial FD：

\[
X \to Y
\]

都要求 \(X\) 是 superkey。

BCNF 比 3NF 更强。3NF 允许 “RHS 是 prime attribute” 的例外；BCNF 不允许。

### 4.12 Highest normal form 判断流程

1. 算 candidate keys。
2. 算 prime attributes。
3. 检查 2NF：有没有 candidate key 真子集推出 non-prime attribute。
4. 如果不满足 2NF，最高通常是 1NF。
5. 若满足 2NF，再检查 3NF。
6. 若满足 3NF，再检查 BCNF。
7. 若每条 non-trivial FD 的 LHS 都是 superkey，则 BCNF。

写答案时不要只说 “not BCNF”。题目问 highest normal form，必须明确是 1NF / 2NF / 3NF / BCNF。

### 4.13 3NF synthesis

输入：minimal cover \(F_m\)。

步骤：

1. 对每个 FD \(X \to A\)，创建 relation \(XA\)。
2. 若有多个 FD 同 LHS，可合并为 \(X A_1 A_2 \ldots\)。
3. 删除被其他 relation 完全包含的 relation。
4. 若没有任何 relation 包含原 relation 的 candidate key，则额外加入一个 key relation。
5. 标出每个 resulting relation 的 key。

性质：

- 一定 dependency preserving。
- 若加入 key relation，则 lossless。
- 不一定 BCNF。

### 4.14 BCNF decomposition

若 relation \(R\) 中存在违反 BCNF 的 FD：

\[
X \to Y
\]

其中 \(X\) 不是 superkey，则分解为：

\[
R_1 = X \cup Y
\]

\[
R_2 = R - (Y - X)
\]

然后对 \(R_1\) 和 \(R_2\) 递归检查。

性质：

- 每一步都是 lossless。
- 最终 BCNF。
- 可能不 dependency preserving。

### 4.15 Superkey 数量题

求 “superkeys that are not candidate keys”：

1. 先找所有 candidate keys。
2. 一个 superkey 是包含至少一个 candidate key 的 attribute set。
3. 用 inclusion-exclusion 或枚举计数。
4. 最后减去 candidate key 个数。

如果 candidate keys 两两重叠，不能简单相加：

\[
|\{S: K_1 \subseteq S\}| + |\{S: K_2 \subseteq S\}|
\]

会重复计算同时包含 \(K_1\) 和 \(K_2\) 的 set。需要 inclusion-exclusion。

### 4.16 完整例题：Minimal cover、keys、lossless、highest NF、BCNF

考虑：

\[
R(A,B,C,D,E,G,H,I,J)
\]

\[
F = \{BD \to CH,\; BC \to HI,\; EI \to H,\; H \to AB,\; I \to E,\; EJ \to I\}
\]

#### Step 1：拆 RHS

\[
BD \to C,\quad BD \to H
\]

\[
BC \to H,\quad BC \to I
\]

\[
EI \to H
\]

\[
H \to A,\quad H \to B
\]

\[
I \to E
\]

\[
EJ \to I
\]

#### Step 2：删 extraneous / redundant

因为 \(I \to E\)，所以在 \(EI \to H\) 中，`E` 是多余的：

\[
EI \to H \quad \Rightarrow \quad I \to H
\]

又因为：

\[
BD \to C,\quad BC \to I,\quad I \to H
\]

所以 \(BD \to H\) 是 redundant。

因为：

\[
BC \to I,\quad I \to H
\]

所以 \(BC \to H\) 是 redundant。

一个 minimal cover 可以写成：

\[
F_m =
\{BD \to C,\; BC \to I,\; I \to H,\; H \to B,\; H \to A,\; I \to E,\; EJ \to I\}
\]

#### Step 3：candidate keys

注意 `D,G,J` 从未出现在 RHS，因此每个 candidate key 都必须包含 `D,G,J`。

枚举补充属性后可得：

\[
BDGJ,\quad DEGJ,\quad DGHJ,\quad DGIJ
\]

这些都是 candidate keys。  
它们都能推出全部属性，而且任意真子集都不能推出全部属性。

#### Step 4：prime attributes

出现在至少一个 candidate key 中的 attributes：

\[
B,D,E,G,H,I,J
\]

因此 non-prime attributes 是：

\[
A,C
\]

#### Step 5：lossless check

给定：

\[
R_1 = \{A,B,C,D,H\}
\]

\[
R_2 = \{E,G,H,I,J\}
\]

交集：

\[
R_1 \cap R_2 = \{H\}
\]

计算：

\[
H^+ = \{H,A,B\}
\]

它既不能覆盖 \(R_1\)，也不能覆盖 \(R_2\)。因此该二元分解不是 lossless。

#### Step 6：highest normal form

存在：

\[
BD \to C
\]

其中 `BD` 是 candidate key `BDGJ` 的真子集，`C` 是 non-prime attribute。  
因此违反 2NF。最高 normal form 是 1NF。

#### Step 7：一个 BCNF decomposition

可按违反 BCNF 的 FD 逐步分解。一个可接受的结果是：

\[
R_1(\underline{H}, A, B)
\]

\[
R_2(\underline{I}, E, H)
\]

\[
R_3(\underline{D,I}, C)
\]

\[
R_4(\underline{D,G,I,J})
\]

说明：

- 在 \(R_1(H,A,B)\) 中，\(H \to AB\)，`H` 是 key。
- 在 \(R_2(I,E,H)\) 中，\(I \to EH\)，`I` 是 key。
- 在 \(R_3(D,I,C)\) 中，\(DI \to C\)，`DI` 是 key。
- \(R_4(D,G,I,J)\) 中没有非平凡 FD 违反 BCNF，整组属性可作为 key。

BCNF decomposition 不唯一。考试中只要每一步 lossless，并且最后每个 relation 满足 BCNF，就可以。

---

## Q5. SQL 写法模板

### 5.1 基本 join

```sql
SELECT DISTINCT c.name
FROM customer c
JOIN watched w ON w.cid = c.cid
JOIN movie m ON m.mid = w.mid
WHERE m.name = 'Lorem Ipsum';
```

使用 `DISTINCT` 的原因：同一个 customer 可能在不同年份看过同一部电影，或 join 产生重复。

### 5.2 “none” 用 `NOT EXISTS`

```sql
SELECT c.cid
FROM customer c
WHERE NOT EXISTS (
  SELECT 1
  FROM watched w
  WHERE w.cid = c.cid
);
```

### 5.3 “all” 用 double `NOT EXISTS`

```sql
SELECT c.cid
FROM customer c
WHERE NOT EXISTS (
  SELECT 1
  FROM movie m
  WHERE NOT EXISTS (
    SELECT 1
    FROM watched w
    WHERE w.cid = c.cid
      AND w.mid = m.mid
  )
);
```

读法：

> 不存在一部 movie，使得这个 customer 没有看过它。

### 5.4 “only” 用 `NOT EXISTS`

```sql
SELECT DISTINCT w.mid
FROM watched w
WHERE NOT EXISTS (
  SELECT 1
  FROM watched w2
  JOIN customer c2 ON c2.cid = w2.cid
  WHERE w2.mid = w.mid
    AND c2.age < 30
);
```

这表示该 movie 没有被 age < 30 的 customer 看过。  
若题目要求 “only watched by age ≥ 30” 且至少有人看过，该查询从 `watched w` 起步，天然保证至少一个观看记录。

### 5.5 “same name at least twice”

```sql
SELECT DISTINCT w1.cid
FROM watched w1
JOIN watched w2
  ON w1.cid = w2.cid
 AND w1.mid <> w2.mid
JOIN movie m1 ON m1.mid = w1.mid
JOIN movie m2 ON m2.mid = w2.mid
WHERE m1.name = m2.name;
```

如果允许同一 movie 不同 year 也算 watched twice，则把 `w1.mid <> w2.mid` 改成：

```sql
(w1.mid, w1.year) <> (w2.mid, w2.year)
```

但题目通常强调 “movies with the same name”，更可能要求不同 movie IDs。

### 5.6 “greatest count with ties”

```sql
WITH Counts AS (
  SELECT mid, COUNT(*) AS cnt
  FROM watched
  GROUP BY mid
),
MaxCnt AS (
  SELECT MAX(cnt) AS max_cnt
  FROM Counts
)
SELECT c.mid
FROM Counts c
JOIN MaxCnt m ON c.cnt = m.max_cnt;
```

不要用 `ORDER BY cnt DESC LIMIT 1`，除非题目不关心 ties。

### 5.7 `GROUP BY` 的安全写法

只要 SELECT 中出现非聚合列，该列通常必须在 `GROUP BY` 中。

正确：

```sql
SELECT gender, AVG(age) AS avg_age
FROM Customer c
WHERE EXISTS (
  SELECT 1
  FROM Review r
  WHERE r.cID = c.cID
)
GROUP BY gender;
```

错误：

```sql
SELECT gender, name, AVG(age)
FROM Customer
GROUP BY gender;
```

除非 DBMS 特殊允许，否则 `name` 不在 group key 中。

---

## Q6. Transactions、Serializability、Locks、Deadlocks

### 6.1 Conflict serializability

两个操作 conflict，当且仅当：

1. 属于不同 transactions；
2. 访问同一个 data item；
3. 至少一个是 write。

冲突类型：

| Earlier | Later | Conflict? |
|---|---|---|
| \(R_i(X)\) | \(R_j(X)\) | No |
| \(R_i(X)\) | \(W_j(X)\) | Yes |
| \(W_i(X)\) | \(R_j(X)\) | Yes |
| \(W_i(X)\) | \(W_j(X)\) | Yes |
| 访问不同 item | 任意 | No |

### 6.2 Precedence graph 做法

1. 每个 transaction 是一个 node。
2. 按时间顺序看 schedule。
3. 如果 \(T_i\) 的操作先于 \(T_j\)，且二者 conflict，则加边：

\[
T_i \to T_j
\]

4. 若 graph 有 cycle，则不是 conflict-serializable。
5. 若无 cycle，则 conflict-serializable，任意 topological order 是等价 serial schedule。

考试中要写边的来源。例如：

\[
W_2(Y) \text{ before } R_4(Y) \Rightarrow T_2 \to T_4
\]

不要只画图不解释。

### 6.3 Result serializability

Result-serializable 比 conflict-serializable 更弱。判断通常较麻烦，除非题目小。

基本思路：

1. 列出所有可能 serial orders。
2. 对每个 serial order，比较：
   - 每个 read 读到的是哪个 transaction 写的值；
   - 最终写入每个 data item 的 transaction 是谁。
3. 如果存在一个 serial schedule 与原 schedule 的 read-from 和 final write 效果相同，则 result-serializable。

考试中如果题目问 result-serializability，一般数据项和 transaction 数量较少，可以枚举 serial orders。

### 6.4 Lock compatibility

| Held \ Requested | RL | WL |
|---|---:|---:|
| RL | compatible | conflict |
| WL | conflict | conflict |

多个 read locks 可以共存。write lock 与任何其他 lock 冲突。

### 6.5 Wait-for graph

做 wait-for graph 时：

1. 维护当前每个 item 上持有哪些 locks。
2. 一个 transaction 请求 lock，如果与已有 lock compatible，则授予。
3. 如果不 compatible，则 requester 等待 holder，加边：

\[
Requester \to Holder
\]

4. 如果 unlock，则释放 lock，并删除相应等待关系或重新检查 waiting requests。
5. graph 中有 cycle iff deadlock exists。

例子：

若 \(T_2\) 请求 `WL(C)`，但 \(T_4\) 已持有 `WL(C)`，则加：

\[
T_2 \to T_4
\]

若 \(T_4\) 请求 `RL(B)`，但 \(T_1\) 已持有 `WL(B)`，则加：

\[
T_4 \to T_1
\]

如果存在：

\[
T_1 \to T_3 \to T_2 \to T_4 \to T_1
\]

则 deadlock。

### 6.6 2PL

Two-phase locking 要求每个 transaction 分两个阶段：

1. Growing phase：只获取 locks，不释放 locks。
2. Shrinking phase：只释放 locks，不再获取新 locks。

写 2PL schedule 时：

- read 前加 `RL_i(X)`；
- write 前加 `WL_i(X)`；
- 如果已经有 `RL_i(X)` 但之后要 write，可升级为 `WL_i(X)`；
- unlock 要放在该 transaction 最后一次访问对应 item 之后；
- 一旦释放任何 lock，就不能再获取新的 lock。

2PL 保证 conflict-serializability，但不保证无 deadlock。

### 6.7 Locking 题常见失误

1. `R(X)` 前忘记加 read lock。
2. `W(X)` 前只加 read lock，没加 write lock。
3. 一个 transaction 已 unlock 后又申请新 lock，违反 2PL。
4. read lock 与 read lock 错误地认为冲突。
5. wait-for edge 方向画反。方向永远是 “等待者 → 被等待者”。

---

## Q7. Recovery 与 Log Analysis

### 7.1 Log record 形式

常见 log：

\[
[start\_transaction, T_i]
\]

\[
[write\_item, T_i, X, old, new]
\]

\[
[commit, T_i]
\]

\[
[checkpoint]
\]

如果 crash，恢复时要判断每个 transaction：

- committed before crash：需要保证其 effects 被保留，可能 redo。
- not committed before crash：必须 undo。
- not started before crash：不用处理。

### 7.2 Redo / Undo 基本判断

| Transaction 状态 | 恢复动作 |
|---|---|
| 已 commit | redo，如果其更新可能尚未写回 disk |
| 未 commit 但写过 | undo |
| 未开始 | nothing |
| start 了但没写 | usually nothing 或 abort record，视题目要求 |

如果有 checkpoint，通常只需要从最近 checkpoint 附近开始分析，但考试答案仍应说明 transaction 是否 commit。

### 7.3 例子：crash immediately after a timestamp

假设 log 中：

- \(T_1\) commit 于 \(t_6\)；
- \(T_2\) commit 于 \(t_{11}\)；
- \(T_3\) start 于 \(t_9\)，在 \(t_{12}\) write，但尚未 commit；
- \(T_4\) start 于 \(t_{13}\)，在 \(t_{14}\) write，尚未 commit；
- \(T_3\) commit 于 \(t_{15}\)。

若 crash immediately after \(t_{12}\)：

- \(T_1\)：committed，redo if necessary。
- \(T_2\)：committed，redo if necessary。
- \(T_3\)：active/uncommitted，undo its write on `Z`。
- \(T_4\)：not started，nothing。

若 crash immediately after \(t_{15}\)：

- \(T_1\)：committed，redo if necessary。
- \(T_2\)：committed，redo if necessary。
- \(T_3\)：committed，redo if necessary。
- \(T_4\)：active/uncommitted，undo its write on `X`。

表达时可以写：

> All committed transactions are winners and should be redone if their updates are not guaranteed to be on disk. All uncommitted transactions are losers and should be undone.

---

## Q8. Buffer Management 与 Replacement Policies

### 8.1 Hit / Miss

- Hit：请求页已经在 buffer 中。
- Miss：请求页不在 buffer，需要从 disk 读入；若 buffer 满，还要替换某页。
- Hit rate：

\[
\frac{\#hits}{\#requests}
\]

### 8.2 LRU

LRU = Least Recently Used。  
miss 且 buffer 满时，替换最久没有被访问的页。

适合有 temporal locality 的访问模式。

### 8.3 MRU

MRU = Most Recently Used。  
miss 且 buffer 满时，替换最近刚访问过的页。

MRU 在 repeated sequential scan 中可能比 LRU 好。原因是：顺序扫描一个大表且 buffer 容量小于表大小时，刚刚访问过的页短期内不太会再用，而扫描起点附近的页可能应保留到下一轮。

### 8.4 FIFO

FIFO = First In First Out。  
miss 且 buffer 满时，替换最早进入 buffer 的页，不管最近是否访问过。

FIFO 可能出现 Belady anomaly，即 buffer 变大反而 miss 变多。因此题目若问 “same query set, different buffer size, FIFO worst/best”，要注意可以构造反例。

### 8.5 手动模拟表格格式

例如有 4 frames，请求序列：

\[
3,6,1,7,5,\ldots
\]

建议画表：

| Request | Hit/Miss | Frames after request | Replaced |
|---|---|---|---|
| 3 | M | 3 - - - | - |
| 6 | M | 3 6 - - | - |
| 1 | M | 3 6 1 - | - |
| 7 | M | 3 6 1 7 | - |
| 5 | M | 6 1 7 5 | 3 under LRU |

每一行写 request 后的 buffer 状态。最后统计 hits/misses。

### 8.6 Repeated scan 例题思路

表 \(A\) 有 10 pages，连续扫描 20 次，buffer frames = 4。

对 LRU：

- 每一轮顺序扫描 1 到 10。
- buffer 容量小于 table pages。
- 到下一轮开始时，前几页已经被上一轮末尾页挤出。
- 因此每次请求都是 miss。

总请求数：

\[
10 \times 20 = 200
\]

所以：

\[
hits = 0,\quad misses = 200
\]

对 MRU：

- 第一轮 10 次全 miss。
- 后续每轮通常能保留一部分旧页。
- 需要按 MRU 规则模拟，不要用直觉。
- 在该具体设置下，第一轮后每轮有若干 hits，写答案时应展示模拟规律或表格。

若题目只问解释，重点是：MRU 在大表反复 sequential scan 时可能更好，因为它淘汰刚访问过且短期不会再用的页。

---

## Q9. Index、Storage、Physical Design 概念

### 9.1 Hash index

适合：

```sql
WHERE id = 123
```

不适合：

```sql
WHERE age BETWEEN 20 AND 30
WHERE name > 'M'
```

因为 hash 打散 key 的顺序，不能有效支持 range scan。

### 9.2 Ordered index / B+ tree

适合：

- equality search；
- range search；
- ordered traversal；
- prefix range。

B+ tree 的叶子节点通常按 key 顺序链接，因此 range query 效率高。

### 9.3 ISAM

ISAM = Indexed Sequential Access Method。

特点：

- 主结构相对 static。
- 使用 index pages 指向 data pages。
- 插入后若目标 page 满，可能使用 overflow pages。
- 不像 B+ tree 那样自动通过 split/merge 保持动态平衡。

因此 “ISAM is dynamic” 通常是 false。

### 9.4 Free space management

数据库系统需要管理 disk pages、free blocks、buffer pool。  
OS file system 能提供基础文件和 block abstraction，但 DBMS 仍需要自己的 page layout、free-space map、record placement、buffer replacement 等策略，以减少 I/O 和支持事务。

考试短答可以写：

> The OS file system provides basic block storage, but the DBMS often maintains its own page and free-space information to choose pages efficiently and reduce disk I/O.

---

## Q10. 关系代数高频完整模板集

### 10.1 Customers over 50

\[
\pi_{cid,name}(\sigma_{age>50}(customer))
\]

SQL：

```sql
SELECT cid, name
FROM customer
WHERE age > 50;
```

### 10.2 Never watched any movie

\[
\pi_{cid}(customer) - \pi_{cid}(watched)
\]

SQL：

```sql
SELECT c.cid
FROM customer c
WHERE NOT EXISTS (
  SELECT 1
  FROM watched w
  WHERE w.cid = c.cid
);
```

### 10.3 Most watched movie in year 2022

若允许 grouping：

\[
W_{2022} = \sigma_{year=2022}(watched)
\]

\[
C = \gamma_{mov\_id;\ count(*) \to cnt}(W_{2022})
\]

\[
C_1 = \rho_{C1}(C),\quad C_2=\rho_{C2}(C)
\]

\[
NotMax = \pi_{C1.mov\_id}(\sigma_{C1.cnt<C2.cnt}(C_1 \times C_2))
\]

\[
Answer = \pi_{mov\_id}(C) - NotMax
\]

### 10.4 Only reviewed by one gender

\[
M = \pi_{rID}(\sigma_{gender='M'}(Customer \bowtie Review))
\]

\[
F = \pi_{rID}(\sigma_{gender='F'}(Customer \bowtie Review))
\]

\[
Answer = (M-F)\cup(F-M)
\]

### 10.5 Average age by gender of reviewers

\[
Reviewers = Customer \bowtie \pi_{cID}(Review)
\]

\[
Answer = \gamma_{gender;\ AVG(age)\to avgAge}(Reviewers)
\]

### 10.6 Properties represented by all agents but sold only by some

设：

\[
RepPair = \pi_{pID,aID}(Representation)
\]

\[
AllAgent = \pi_{aID}(Agent)
\]

被所有 agents represented 的 properties：

\[
AllRep = RepPair \div AllAgent
\]

property 被 agent sold 的定义：

\[
SoldBy =
\pi_{pID,aID}
(
\sigma_{Sale.pID=Representation.pID
\land Sale.saleDate \ge Representation.startDate
\land Sale.saleDate \le Representation.endDate}
(Sale \times Representation)
)
\]

被所有 agents sold 的 properties：

\[
AllSold = SoldBy \div AllAgent
\]

被 some agents sold：

\[
SomeSold = \pi_{pID}(SoldBy)
\]

答案：

\[
AllRep \cap (SomeSold - AllSold)
\]

若关系代数没有 intersection，可用：

\[
A \cap B = A - (A-B)
\]

### 10.7 Owners who only owned properties never represented by Joe Smith

先找 Joe Smith represented 的 properties：

\[
JoeProp =
\pi_{pID}
(
\sigma_{aName='Joe Smith'}(Agent \bowtie Representation)
)
\]

找拥有过这些 property 的 owners：

\[
BadOwner =
\pi_{oID}
(
Representation \bowtie JoeProp
)
\]

所有 owners who owned/represented properties：

\[
AllOwner = \pi_{oID}(Owner)
\]

答案：

\[
AllOwner - BadOwner
\]

这里 “only owned properties that are never represented by Joe Smith” 常被翻译成“没有任何 owned property 被 Joe Smith represented”。

### 10.8 Highest price sale per property and agent

\[
SaleAgent =
\pi_{pID,aID,price}
(
\sigma_{Sale.pID=Representation.pID
\land Sale.saleDate \ge startDate
\land Sale.saleDate \le endDate}
(Sale \times Representation)
)
\]

\[
S_1 = \rho_{S1}(SaleAgent),\quad S_2=\rho_{S2}(SaleAgent)
\]

\[
NotHighest =
\pi_{S1.pID,S1.aID}
(
\sigma_{S1.pID=S2.pID \land S1.price<S2.price}
(S_1 \times S_2)
)
\]

\[
Answer = \pi_{pID,aID}(SaleAgent) - NotHighest
\]

---

## Q11. 考前检查表

### 11.1 ER 题检查表

提交前检查：

- 每个 entity 是否有 key。
- 所有 “multiple” 是否用 multivalued attribute 或 relation 表达。
- 所有 “exactly one / at least one / zero or more” 是否体现在 cardinality / participation。
- 弱实体是否包含 owner key。
- relationship attribute 是否放在正确位置。
- derived attribute 是否标注或说明。
- assumptions 是否写了。

### 11.2 Relational algebra 检查表

- 投影前是否保留了后续需要的属性。
- set difference 两边 schema 是否一致。
- division 左边是否是 `(answerAttrs, divisorAttrs)`，右边是否是 divisor attributes。
- “only” 是否写成 `Good - Bad`。
- “none” 是否从全集减去存在记录者。
- “all” 是否用了 division 或 double anti-join。
- “highest” 是否保留 ties。
- self-join 是否 rename 了 relation。

### 11.3 SQL 检查表

- join condition 是否完整。
- `DISTINCT` 是否需要。
- `GROUP BY` 是否包含所有非聚合列。
- “not in category” 是否用了 `NOT EXISTS`，避免多值 category 的错误。
- date interval 是否用正确 overlap 条件。
- `ORDER BY` 是否被误用来代替 aggregate。
- ties 是否保留。

### 11.4 FD / Normalisation 检查表

- 是否先拆 RHS。
- closure 是否算到不能再扩展。
- candidate key 是否 minimal。
- prime attributes 是否来自所有 candidate keys。
- highest NF 是否从 2NF、3NF、BCNF 逐级检查。
- lossless 二元分解是否用 intersection closure。
- 多元 lossless 是否用 chase。
- dependency preserving 是否用 projected FDs 的 closure。
- 3NF synthesis 是否添加了 key relation。
- BCNF 分解是否每一步 lossless。

### 11.5 Transaction 检查表

- precedence graph 边是否只来自 conflict。
- wait-for graph 边方向是否是 requester → holder。
- read lock 是否与 read lock compatible。
- 2PL 中 unlock 后是否没有再 lock。
- recovery 中 committed 和 uncommitted 是否分清。
- crash timestamp 是否处理 “immediately after” 的含义。

---

## Q12. 最后复习建议

如果时间只够做三件事，优先级应是：

1. **FD/范式/分解**：这部分分值高，算法性强，练熟后稳定拿分。
2. **关系代数/SQL 模板**：all / none / only / most / same / available 是反复出现的模式。
3. **ER 建模转换**：把文字约束翻译成 cardinality、participation、weak entity、多值属性。

概念题不要用大段解释。每个概念用一句定义、一句对比、一句例子即可。  
FD 和关系代数不要靠直觉。写出中间集合或中间 relation，答案会更容易检查，也更容易拿过程分。
