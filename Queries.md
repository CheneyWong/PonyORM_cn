# 请求

Pony 提供了一个非常方便的途径，使用生成器语法去请求数据库。Pony 让程序员操作存储在数据库中的对象和操作内存中的对象一样，使用原生的 Python 语法。这让开发工作更为简单。

## Pony ORM 中用来请求数据库的函数

- select(gen[, globals[, locals])
这个方法把生成器语法转化为 SQL 请求，并返回 `Query` 类型的实例。如果有必要，你可以在结果上使用任何 `Query` 的方法，例如，`Query.order_by()` 或 `Query.count()` 。如果你只是想获取一个对象的列表，你可以使用枚举遍历结果或者使用完整的切片。
```python
for p in select(p for p in Product):
    print p.name, p.price

prod_list = select(p for p in Product)[:]
```

`select()` 函数也可以返回一个单独的属性的列表，或者元组的列表：
```python
select(p.name for p in Product)

select((p1, p2) for p1 in Product
                for p2 in Product if p1.name == p2.name and p1 != p2)

select((p.name, count(p.orders)) for p in Product)
```
你可以传递 `globals` 和 `locals` 字典，将它们用作 `global` 和 `local` 名字空间。
你也可以将 `select` 用于关系属性，请看[例子]()。

- get(gen[, globals[, locals])
从数据库中提取一个实体对象的实例。如果指定参数的对象存在，返回这个对象，如果对象不存在，返回 `None`。如果满足条件的对象有多个，会抛出异常：
`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：
`get(o for o in Order if o.id == 123)`
`Query.get()` 也会生成相同的请求：
`select(o for o in Order if o.id == 123).get()`

- left_join(gen[, globals[, locals])
左连接的查询结果总是包含 "left" 表中的结果，即使在 "right" 表中没有找到任何匹配项。
假如我们需要计算每位顾客的账单的金额。让我们使用 Pony 做这个例子。
```python
from pony.orm.examples.estore import *
populate_database()

select((c, count(o)) for c in Customer for o in c.orders)[:]
```
这将会被转换为如下 SQL 语句：
```sql
SELECT "c"."id", COUNT(DISTINCT "o"."id")
FROM "Customer" "c", "Order" "o"
WHERE "c"."id" = "o"."customer"
GROUP BY "c"."id"
```
返回如下结果：
```python
[(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1)]
```
 但是没有账单的顾客将不会被这个请求选中，因为条件是 `WHERE "c"."id" = "o"."customer"` ，不会在账单表中找到匹配的记录。为了得到所有顾客的列表，我们应该使用 `left_join` 函数：
 `left_join((c, count(o)) for c in Customer for o in c.orders)[:]`
 ```sql
 SELECT "c"."id", COUNT(DISTINCT "o"."id")
FROM "Customer" "c"
  LEFT JOIN "Order" "o"
    ON "c"."id" = "o"."customer"
GROUP BY "c"."id"
 ```
 现在我们得到了所有顾客的列表，包含没有账单的顾客。
 ```python
 [(Customer[1], 2), (Customer[2], 1), (Customer[3], 1), (Customer[4], 1), (Customer[5], 0)]
 ```
 我们也知道，大多数情况 Pony 能够自动判断 LEFT JOIN 是必须的，例如，相同的请求我们可以已这种方式来写。
 `select((c, count(c.orders)) for c in Customer)[:]`
 ```python
 SELECT "c"."id", COUNT(DISTINCT "order-1"."id")
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
GROUP BY "c"."id"
 ```
- count(gen)
返回符合查询条件的对象的个数，例如：
`count(c for c in Customer if len(c.orders) > 2)`
这个请求将会被转换为如下 SQL 语句：
```sql
SELECT COUNT(*)
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
GROUP BY "c"."id"
HAVING COUNT(DISTINCT "order-1"."id") > 2
```
`Query.count()` 方法也会生成等价的请求：
```python
select(c for c in Customer if len(c.orders) > 2).count()
```
- min(gen)
返回数据库中最小的值，请求会返回一个单独的属性的值。
`min(p.price for p in Product)`
`Query.min()` 方法也会生成等价的请求：
`select(p.price for p in Product).min()`

- max(gen)
- 返回数据库中最大的值，请求会返回一个单独的属性的值。
`max(p.price for p in Product)`
`Query.max()` 方法也会生成等价的请求：
`select(p.price for p in Product).max()`

- sum(gen)
返回被选中的属性的值的和：
`sum(o.total_price for o in Order)`
`Query.sum()` 方法也会生成等价的请求：
`select(o.total_price for o in Order).sum()`
如果没有匹配到元素，`sum` 方法返回 0。

- avg(gen)
返回被选中的属性的值的平均值：
`avg(o.total_price for o in Order)`
`Query.avg()` 方法也会生成等价的请求：
`select(o.total_price for o in Order).avg()`

- exists(gen[, globals[, locals])
如果至少一条指定条件的对象存在，返回 `True`，其他情况返回 `False`。
`exists(o for o in Order if o.date_delivered is None)`

- distinct(gen)
当你想要强制的返回唯一的请求，可以使用  `distinct()` 函数。
`distinct(o.date_shipped for o in Order)`
通常情况下这是非必须的，因为 Pony 会聪明的自动添加 DISTINCT 。查看下一节 Automatic DISTINCT 获取更多信息。
另外一个使用 `distinct()` 的地方是和 `sum` 一起用。你可以这么写：
```python
select(sum(distinct(x.val)) for x in X)
```
会生成如下 SQL：
```sql
SELECT SUM(DISTINCT x.val)
FROM X x
```
但是很少在生产中使用。

- desc()
- desc(attribute)
在函数 `Query.order_by()` 内部调用，为了实现倒序排列。


## Query 对象的方法
class Query

- [start:end]
- limit(limit, offset=None)
这个方法用来限定从数据库中选中的实例的数量。下面的例子选中了前十个元素。
`select(c for c in Customer).order_by(Customer.name).limit(10)`
会生成如下 SQL：
```python
SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
FROM "Customer" "c"
ORDER BY "c"."name"
LIMIT 10
```
Python 切片操作也可以达到相同的目的：
`select(c for c in Customer).order_by(Customer.name)[:10]`
如果我们需要选中指定偏移量的实例，可以使用第二个参数：
`select(c for c in Customer).order_by(Customer.name).limit(10, 20)`
或者使用切片操作：
`select(c for c in Customer).order_by(Customer.name)[20:30]`
会生成如下 SQL：
```python
SELECT "c"."id", "c"."email", "c"."password", "c"."name", "c"."country", "c"."address"
FROM "Customer" "c"
ORDER BY "c"."name"
LIMIT 10 OFFSET 20
```
`page()` 方法也会实现相同的目的。

- avg()
返回选中的属性的平均值
`select(o.total_price for o in Order).avg()`
`avg()` 函数做了相同的工作。

- count()
返回满足条件的对象的个数
`select(c for c in Customer if len(c.orders) > 2).count()`
`count()` 函数做了相同的工作。

- distinct()
强制开启 DISTINCT 请求：
`select(c.name for c in Customer).distinct()`
通常情况下这是非必须的，因为 Pony 会聪明的自动添加 DISTINCT 。查看下一节 Automatic DISTINCT 获取更多信息。`distinct()` 函数做了相同的工作。

- exists()
如果至少一条指定条件的对象存在，返回 `True`，其他情况返回 `False`。
`select(c for c in Customer if len(c.cart_items) > 10).exists()`
这个请求生成了如下 SQL:
```python
SELECT "c"."id"
FROM "Customer" "c"
  LEFT JOIN "CartItem" "cartitem-1"
    ON "c"."id" = "cartitem-1"."customer"
GROUP BY "c"."id"
HAVING COUNT(DISTINCT "cartitem-1"."id") > 20
LIMIT 1
```

- filter(lambda[, globals[, locals])
- filter(str)
`Query` 对象的 `filter()` 方法，是为了对查询结果进行过滤。做为`filter()` 方法参数传递的过滤条件将会被翻译成 WHERE 区的 SQL 请求。
`filter()` 参数的个数应该匹配请求结果。`filter()`可以用 lambda 表达式作为过滤条件：
```python
q = select(p for p in Product)
q2 = q.filter(lambda x: x.price > 100)

q = select((p.name, p.price) for p in Product)
q2 = q.filter(lambda n, p: n.name.startswith("A") and p > 100)
```
`filter()` 方法也可以传递一个表达式的字符串：
```python
q = select(p for p in Product)
x = 100
q2 = q.filter("p.price > x")
```
另外一种过滤请求结果的方式是传递指定参数的名字：
```python
q = select(p for p in Product)
q2 = q.filter(price=100, name="iPod")
```

- first()
返回选中的结果的第一个对象，当没有对象的时候返回 `None`：
`select(p for p in Product if p.price > 100).first()`

- for_update(nowait=False)
有些时候需要锁定数据库中的对象，保证别的事务不是在相同的时候修改相同的对象。在数据库中，这种锁使用过使用 SELECT FOR UPDATE 实现的。如果要在 Pony 中使用这样的锁，可以使用 `for_update` 方法：
`select(p for p in Product if p.picture is None).for_update()[:]`
这个请求选中了没有图片的产品的实例并且锁定了它们。锁将会在提交或者回滚的时候解除。

- get()
从数据库中讲一个实例还原。如果对象存在返回满足参数的对象，当没有对象的时候返回 `None`，如果有多个对象满足要求，抛出异常：`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：
`select(o for o in Order if o.id == 123).get()`
`get()` 函数做了相同的工作。

- max()
返回数据库中最大的值，请求会返回一个单独的属性的值。
`select(o.date_shipped for o in Order).max()`
`max()` 函数也会生成等价的请求：

- min()
返回数据库中最小的值，请求会返回一个单独的属性的值。
`select(o.date_shipped for o in Order).min()`
`min()` 函数也会生成等价的请求：

- order_by(attr1[, attr2, ...])
- order_by(pos1[, pos2, ...])
- order_by(lambda[, globals[, locals])
- order_by(str)
这个方法用来给请求结果排序。可用的用法有：
 - 基于实体对象属性
`select(o for o in Order).order_by(Order.customer, Order.date_created)`
对任何属性使用 `desc()` 函数或 `desc()` 方法，可以实现倒排序：
`select(o for o in Order).order_by(Order.date_created.desc())`
也等价于：
`select(o for o in Order).order_by(desc(Order.date_created))`
 - 基于请求结构变量的位置
`select((o.customer.name, o.total_price) for o in Order).order_by(-2, 1)`
负号意为倒排序。在例子中，我们按照总价倒序顾客名正序的方式进行了排序。
 - 基于 lambda
`select(o for o in Order).order_by(lambda o: (o.customer.name, o.date_shipped))`
如果 lambda 有一个参数（在这个例子中是 `o`），那么 `o` 传递了 `select` 的结果。如果你指定了一个没有参数的 lambda，那么 `query` 中的所有名字都能访问：
`select(o.total_price for o in Order).order_by(lambda: o.customer.id)`
 - 基于字符串
这个方法和上一个方法类似，但是你将 lambda 的内容作为一个字符串传递了。
`select(o.total_price for o in Order).order_by(lambda: o.customer.id)`

- page(pagenum, pagesize=10)
当你想在多个页面展示请求结果，你会用到分页。页码从 1  开始计数。这个方法返回一个切片 [start:stop] ,其中 `start = (pagenum - 1) * pagesize, stop = pagenum * pagesize`。

- prefetch()
允许你指定，哪些关联的对象和属性需要一起被载入。
通常没必要 prefetch 关联的对象。当你在 `@db_session` 下操作请求结果的时候，Pony 获取所有你需要的关联的对象。Pony 使用最有效率的方法将关联的对象从数据库中获取出来。避免了 N + 1 问题
。
当你使用 Flask 框架的时候，就推荐使用这种方法，将`@db_session`修饰器放在函数上面，和 `app.route` 的修饰器一起。
```python
@app.route('/index')
@db_session
def index():
    ...
    objects = select(...)
    ...
    return render_template('template.html', objects=objects)
```
或者，一样有效的，使用 `db_session` 包装 wsgi 应用：
`app.wsgi_app = db_session(app.wsgi_app)`
有些情况之下，你需要传递选中的实例和关联的对象到 `db_session` 外，那么你可以使用这个方法。要不然，当你尝试在 `db_session` 外访问关联的对象的时候会得到异常 ： `DatabaseSessionIsOver`，例如：
`DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over`
关于 `db_session` 的更多工作可以在这里找到。
你可以指定实体对象或者和属性作为对象。当你指定一个实体对象，那么“对以”关系和非懒惰属性将被预存取。“对多”关系只有在指定的时候才会预存取。
如果你指定了一个属性，那么只有这个属性会被预存。你可以指定属性链，例如， order.customer.address。预存取是递归的 —— 对每个指定的对象使用。
例如：
`from pony.orm.examples.presentation import *`

载入学生对象，不进行预存取：
`students = select(s for s in Student)[:]`
同时载入学生、组和部门：
```python
students = select(s for s in Student).prefetch(Group, Department)[:]

for s in students: # no additional query to the DB will be sent
    print s.name, s.group.major, s.group.dept.name
```
跟上面的一样，但是指定属性而不是实体对象。
```python
students = select(s for s in Student).prefetch(Student.group, Group.dept)[:]

for s in students: # no additional query to the DB will be sent
    print s.name, s.group.major, s.group.dept.name
```
载入学生和关联的课程。（多对多关系）
```python
students = select(s for s in Student).prefetch(Student.courses)

for s in students:
    print s.name
    for c in s.courses: # no additional query to the DB will be sent
        print c.name
```

- random(limit)
从数据库中选择 `limit` 个随机对象。这个方法会转义为 `ORDER BY RANDOM()` SQL 语句。实体对象的方法 `select_random()` 提供了更好的性能，虽然不允许指定查询条件。

- show()
将请求结果打印到控制台。结果会被格式化为表格的形式。这个方法不会显示对多属性，因为那可能会需要额外的查询而且可能很大。

- sum()
返回选择的元素的和。只可以用开请求数字类型的属性。例如：
`select(o.total_price for o in Order).sum()`
如果查询结果没有元素，那么请求结果将会是 0。

- without_distinct()
默认情况下，Pony 尝试避免重复的请求结果，当必要的时候，智能的添加 `DISTINCT` SQL 关键词。如果你不需要添加 `DISTINCT`，允许获取可能重复的值，那么你可以使用这个方法：
`select(p.name for p in Person).without_distinct().order_by(Person.name)`
Pony 从 Release 0.6 开始，`without_distinct()` 返回一个请求结果，不是一个新的请求实例。

## Automatic DISTINCT
Pony 尝试避免重复的请求结果，当必要的时候，智能的添加 `DISTINCT` SQL 关键词，因为有用的重复结果很少见。当人们想按照指定规则恢复对象的时候，通常不会希望相同的对象返回多个。而且，避免重复让让查询结果更可控：你不需要再去过滤相同的结果。

Pony 只有在可能有重复内容的时候才添加 `DISTINCT`。让我们来看一些例子。

1. 按条件恢复对象
`select(p for p in Person if p.age > 20 and p.name == "John")`
在这个例子中，请求不会出现冲虚，因为结果中包含主键。简单的重复在这里不可能发生，所以没必要添加 `DISTINCT` 关键词，Pony 也没有添加：
```sql
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."age" > 20
  AND "p"."name" = 'John'
```
2. 恢复对象属性
`select(p.name for p in Person)`
这个查询结果返回的不是对象而是它的属性，请求结果可能是重复的，所以 Pony 添加了 `DISTINCT`。
```sql
SELECT DISTINCT "p"."name"
FROM "Person" "p"
```
这种请求的结果通常用在下拉列表中，一般是不希望有重复的。也可以很简单的切换到有重复的查询结果。
如果你想统计名字相同的人的个数，建议你这么做：
`select((p.name, count(p)) for p in Person)`
如果非常有必要获取人们的名字，包含重复的，你可以使用 ` Query.without_distinct()` 方法：
`select(p.name for p in Person).without_distinct()`

3. 并表查询对象
`select(p for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))`
这个请求结果可能是有重复的，所以 Pony 添加了 `DISTINCT`。
```sql
SELECT DISTINCT "p"."id", "p"."name", "p"."age"
FROM "Person" "p", "Car" "c"
WHERE "c"."make" IN ('Toyota', 'Honda')
  AND "p"."id" = "c"."owner"
```
不使用 `DISTINCT`数据是可能重复的，因为这个请求使用了两个表（人和 车），但是只有一个表在选择条件下。请求结果只返回人（不包含车），因此这是一个经典的，让人不满的，结果中获取多个重复的人。我们相信，去掉重复的内容，结果看起来更明了。
但是，如果有些原因不需要执行 desirable ，你总可以给请求中添加 `.without_disctinct()`。
```python
select(p for p in Person for c in p.cars
         if c.make in ("Toyota", "Honda")).without_distinct()
```
在请求结果中包含车和它的所有者，这种情况用户应该更想要看到重复的人的对象。这种情况 Pony 的请求是不同的。
`select((p, c) for p in Person for c in p.cars if c.make in ("Toyota", "Honda"))`
Pony 没有添加 `DISTINCT`。
总结一下：
    1. 原则上，“默认请求不返回重复的结果”，这很容易理解，也不意外。
    2. 这种行为时大多数人在大多数情况下需要的。
    3. 请求不想要 duplicates 的时候， Pony 不添加 DISTINCT。
    4. 使用 `without_distinct()` 可以消除 Pony 使用 DISTINCT。


## 请求中可以使用的函数
这里是能够在生成器中使用的函数的列表：
min、 max、 avg、 sum、 count、 len、 concat、 abs、 random、 select、 exists。

例子：
`select(avg(c.orders.total_price) for c in Customer)[:]`
```sql
SELECT AVG("order-1"."total_price")
FROM "Customer" "c"
  LEFT JOIN "Order" "order-1"
    ON "c"."id" = "order-1"."customer"
```
```sql
select(o for o in Order if o.customer in
       select(c for c in Customer if c.name.startswith('A')))[:]
```
```sql
SELECT "o"."id", "o"."state", "o"."date_created", "o"."date_shipped",
       "o"."date_delivered", "o"."total_price", "o"."customer"
FROM "Order" "o"
WHERE "o"."customer" IN (
    SELECT "c"."id"
    FROM "Customer" "c"
    WHERE "c"."name" LIKE 'A%'
    )
```




