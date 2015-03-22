# 实体对象的实例的使用

## 创建一个实体对象的实例
Pony 中创建一个实体对象的实例跟创建一个 Python 对象是一样的：
```python
customer1 = Customer(login="John", password="***",
                     name="John", email="john@google.com")
```
当创建一个 Pony 对象的时候，所有的参数需要以关键词参数的形式指定。如果属性有默认值的，可以省略。

所有创建的实例都属于当前数据库 session。在一些对象-关系映射器中，你需要调用对象的 `save()` 方法来保存它。这是不方便的，作为程序员，必须跟踪有哪些对象被创建或更新了，必须要记得在每个对象上调用 `save()` 方法。

pony 自动跟踪有哪些对象被创建或更新了，当当前 `db_session` 结束的时候自动保存。如果你想在 `db_session` 结束前保存新建的对象，可以使用 `flush()` 或 `commit()` 函数。

## 从数据库中载入对象

### 通过主键获取对象
最简单的情况是，当我们使用主键恢复一个对象的时候。在 Pony 中实现这个功能，用户只需要将主键放在类名后面的方括号中。例如，提取主键值是 123 的用户，我们可以写为：
`customer1 = Customer[123]`
对于组合键的对象语法也一样，我们需要将组合键的元素一一列出，使用逗号分隔，顺序是实体对象类型定义的时候属性的顺序。
`order_item = OrderItem[order1, product1]`
如果指定主键的对象不存在，Pony 抛出 `ObjectNotFound` 异常。

### 使用唯一性组合键获取唯一对象
当我们不使用主键，而是使用其他属性的结合来提取对象的时候，我们可以在实体对象上使用 `get` 方法。在大多数情况下，我们使用这种方法是通过次位的唯一性属性搜索对象，其实也可以使用其他属性的结合来搜索对象。作为 `get` 方法的参数，我们指定属性的名字和值。例如，我们想要恢复 name 是 “Product 1” 的 product 对象，我们认为数据库中只有一个这样的对象，那么可以这么写：
`product1 = Product.get(name="Product1")`

如果没有找到任何对象，`get` 返回 `None`。如果找到了多个对象，会抛出 `MultipleObjectsFoundError` 异常。

当想要在对象不存在的情况下获得 `None` 而不是 `ObjectNotFound` 异常时，通过主键调用 `get`方法是可以的。

`get()` 方法也可以接收一个 lambda 表达式作为唯一的位置型参数，跟之后要讨论的 `select()` 方法类似，但是返回结果是实体对象，而不是 `Query` 类型的对象。

### 获取多个对象
为了从数据库恢复多个对象，我们使用每个实体对象都有的 `select()` 方法。他的参数是一个 lambda 表达式，接收一个数据库对象实例的别名作为参数。在表达式中，我们可以编写我们想要检索的条件。例如，我们想要找出 products 中 price 大于 100 的，我们可以这么写：
`products = Product.select(lambda p: p.price > 100)`
这个 lambda 表达式将不会被 Python 执行，而是被转化为 SQL 语句：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > 100
```
`select()` 返回 Query 类型的对象。如果你在这个对象上使用迭代，SQL 语句将被发送到数据库，你将得到一个实体对象的数组。例如，这就是我们怎样打印所有符合要求的的产品的名字和价格。
```python
for p in Product.select(lambda p: p.price > 100):
    print p.name, p.price
```
如果我们不想完全迭代一个查询结果，仅仅是需要一个对象的列表，我们可以这样做：
`product_list = Product.select(lambda p: p.price > 100)[:]`
这里，我们从结果中你获取了一个完整切片，这和将结果转为列表是等价的。
`product_list = list(Product.select(lambda p: p.price > 100))`

###  向语句中传递参数
在 lambda 表达式中，可以使用之前声明的参数。在这种情况下，查询中会将这些变量的值传递给参数。Pony 查询语句的一个重要的优势是提供完全的 SQL 注入保护，像这种外面声明的参数的值都将被适当的转换。

例如，我们想要找出price 高于 x  的 products，我们可以简单的写成。
```python
x = 100
products = Product.select(lambda p: p.price > x)
```
这个 SQL 查询将被生成为：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > ?
```
`x` 的值将会传递给 SQL 语句，而且完全避免 SQL 注入风险。

### 查询结果排序
如果我们要将对象按照一定的顺序排序，我们可以使用 `Query` 对象的 `order_by()` 方法。如果我们想倒序显示价格高于 100 的产品的名字和价格，我们可以这么做：
`Product.select(lambda p: p.price > 100).order_by(desc(Product.price))`
`Query` 对象的方法会修改发往数据库的 SQL 语句。在如上的例子中，将会生成下面的 SQL：
```sql
SELECT "p"."id", "p"."name", "p"."description",
       "p"."picture", "p"."price", "p"."quantity"
FROM "Product" "p"
WHERE "p"."price" > 100
ORDER BY "p"."price" DESC
```
`order_by()` 方法也可以接收一个 lambda 表达式作为参数：
`Product.select(lambda p: p.price > 100).order_by(lambda p: desc(p.price))`

`order_by()` 方法中使用 lambda 表达式可以实现更高级的排序方法。例如，我们要将客户按照客户的账单总价倒序排列：
`Customer.select().order_by(lambda c: desc(sum(c.orders.total_price)))`

为了使用多个属性对结果排序，我们用逗号分割它们。例如，如果我们希望按照产品价格排序，如果价格相同，则使用产品名字母顺序排序，我们可以这么做。
`Product.select(lambda p: p.price > 100).order_by(desc(Product.price), Product.name)`

相同的请求，但是使用 lambda 实现，如下：
`Product.select(lambda p: p.price > 100).order_by(lambda p: (desc(p.price), p.name))`

注意，为了和 Python 的语法一致，如果希望从 lambda 返回多个元素，我们需要使用括号包裹它们。

### 限定选中的对象的个数

使用 Query 的 limit() 方法，或者更简洁的 Python 的 slice 标记，可以限定请求的对象的数量。例如，这是我们如果和获取最贵的是个产品的方法。
`Product.select().order_by(lambda p: desc(p.price))[:10]`
slice 的结果不是 Query 对象，而是实体对象实例的列表。

你也可以使用 Query.page() 方法作为请求结果分页实现的简单方法：
`Product.select().order_by(lambda p: desc(p.price)).page(1)`

### 反查关系
在 Pony 中你可以很简单的反查关系：
```python
order = Order[123]
customer = order.customer
print customer.name
```
Pony 尝试最小化发送到数据库的请求的数量。在上面的例子中，如果 `Customer` 的对象已经在缓存中存在，Pony 将不会发送数据库请求直接返回缓存中的对象。但是如果对象没有被载入，Pony 也还是不会立即发送请求。而是先创建了一个 “种子” 对象。种子就是只有主键被初始化的对象。Pony 并不知道这个对象什么时候会被使用，总是存在一个可能，只需要主键就足够了。

在上面的例子中，Pony 在第三行执行的时候从数据库获取对象，即当我们访问 name 属性的时候。通过“种子”思想，Pony 获得了极高的效率和解决了“N+1”问题，这些是其他映射器的软肋。

在“对多”关系中反查也是可以的。例如，如果我们有一个 `Customer` 对象，我们想要遍历他的账单，我们可以这么做：

```python
c = Customer[123]
for order in c.orders:
    print order.state, order.price
```

## 更新一个对象
当你给对象的属性赋新值之后，不需要对更新的对象手动保存。修改将会在离开 `db_session` 区域的时候自动保存。
例如，为了给主键为 123 的对象数量加 10，可以执行如下代码：
`Product[123].quantity += 10`
如果我们需要盖面一个对象的多个方法，我们可以分开做：
```python
order = Order[123]
order.state = "Shipped"
order.date_shipped = datetime.now()
```
或者使用 `set()` 方法，只有一行：
```python
order = Order[123]
order.set(state="Shipped", date_shipped=datetime.now())
```
`set()` 方法在使用字典更新一个对象的多个属性时很方便：
`order.set(**dict_with_new_values)`

如果你想在 `db_session` 结束前保存新建的对象，可以使用 `flush()` 或 `commit()` 函数。

在执行 `select()`、 `get()`、 `exists()`、 `execute()` 和 `commit()` 之前，Pony 总是自动增量的保存 `db_session` 的缓存到数据库。

未来，Pony 将会支持大块更新。那将允许在硬盘中直接更新大量对象而不需要将他们读取到内存：
`update(p.set(price=price * 1.1) for p in Product if p.category.name == "T-Shirt")`

## 删除一个对象
当你调用实体对象的实例的 `delect()` 方法时，这个对象将被标记为已删除。在之后的提交之后，对象将在数据库中被删除。
例如，这就是我们怎样删除一个主键等于 123 的对象的方法。
`Order[123].delete()`

未来，Pony 将会支持大块删除。那将允许在硬盘中直接删除大量对象而不需要将他们读取到内存：
`delete(p for p in Product if p.category.name == "Floppy disk")`

### 关联删除
当 Pony 删除一个实体对象的实例的时候，需要同时删除和其他对象的关系。两个对象的关系是由两条关系属性定义的。如果关系的另一边声明的是 `Set`，那么我们只需要从集合中删除那个对象就好了。如果另一边声明的是 `Optional`，那么我们将它设置为 `None`，如果另一边声明的是 `Required`，我们不能直接将关系属性改为 `None`。这种情况下，`Pony` 尝试继续删除关联的对象。这个默认的行为可以被 `cascade_delete` 属性改写。如果关系的另一边是 `Required` ，这个属性默认的值是 `True` ，其他类型的关系，默认是 `False`。

`True` 意味着 Pony 总是做关联的删除，即时另一边定义的是 `Optional`。 `False` 意味着 Pony 总是不对这个关系做关联的删除。如果另一边关系定义为 `Required`，而且 `cascade_delete=False` Pony 在尝试删除的时候会抛出异常
`ConstraintError`。

一对多关系中关联删除的例子。尝试删除和学生关联的小组时，会抛出异常 `ConstraintError`：

```python
class Group(db.Entity):
    major = Required(str)
    items = Set("Student", cascade_delete=False)

class Student(db.Entity):
    name = Required(str)
    group = Required(Group)
```

一对一关系中关联删除的例子。当删除 `Person` 的时候，会删除关联的 `Passport`：

```python
class Person(db.Entity):
    name = Required(str)
    passport = Optional("Passport", cascade_delete=True)

class Passport(db.Entity):
    number = Required(str)
    person = Required("Person")\
```

## 实体对象的方法
class Entity
- []
返回通过主键选定的实体对象的实例。如果没有对象，抛出 `ObjectNotFound` 异常。例如:
`p = Product[123]`
对于使用复合主键的对象，使用逗号分隔两个主件。
```python
order_id = 123
product_id = 456
item = OrderItem[123, 456]
```
如果主键指定的实体对象在 `db_session` 中被载入，Pony 直接从缓存中返回数据，不再给数据库发请求。

- describe()
返回一个描述实体对象的字符串。例如：
```python
>>> from pony.orm.examples.estore import *
>>> print OrderItem.describe()

class OrderItem(Entity):
    quantity = Required(int)
    price = Required(Decimal)
    order = Required(Order)
    product = Required(Product)
    PrimaryKey(order, product)
```

- drop_table(with_all_data = False)
删除数据库中与实体对象对应的表。如果 `with_all_data = False` 而且表不为空，这个方法会抛出 `TableIsNotEmpty` 异常，不会删除任何东西。设置 `with_all_data = True` 可以让你删除任何数据表，即使不为空。

如果你要删除多对多关系中的中间表，你可以使用实体对象类型的（不是实例的） `drop_table` 方法。

```python
class Product(db.Entity):
    tags = Set('Tag')

class Tag(db.Entity):
    products = Set(Product)

Product.tags.drop_table(with_all_data=True) # removes the intermediate table
```
- exists(lambda[, globals[, locals])¶
- exists(**kwargs)
如果满足指定条件或属性值的实例存在，返回`True`，反之，返回 `False`。例如：
```python
Product.exists(price=1000)

Product.exists(lambda p: p.price > 1000)
```
- get(lambda[, globals[, locals])
- get(**kwargs)

用于从数据库中恢复一个实体对象。如果满足条件的对象存在，返回这个兑现个。如果没有这个对象，返回 `None`。如果有多个对象满足条件，抛出异常
`MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`，例如：

```python
Product.get(price=1000)

Product.get(lambda p: p.name.startswith('A'))
```

- get_by_sql(sql, globals=None, locals=None)
- select_by_sql(sql, globals=None, locals=None)

如果你发现你不能用标准的 Pony 请求表达一次请求，你可以使用你自己的 SQL 请求，Pony 会根据返回结果建立实体对象的实例。当 Pony 获得 SQL 请求的结果时，会分析数据库游标返回的字段的名字。如果你是用的请求语句是 `SELECT * ...` ，这样可能也获取到了足够的的信息来构建实体对象的实例。你也可以给请求中传递参数，查看使用原始 SQL 章节获取更多信息。

- get_for_update(lambda,[globals[, locals], nowait=False)
- get_for_update(**kwargs, nowait=False)

跟 `get()` 方法一样，但是使用 `SELECT ... FOR UPDATE` SQL 请求锁定的这行数据。如果设置了 `nowait=True`, 如果该行已经被锁定，这个方法将会抛出异常。如果设置了 `nowait=False`,它会等待直到这行被释放。

如果你需要对多行使用 `SELECT ... FOR UPDATE`，你可以使用 `Query` 对象的 `for_update()` 方法。

- load()
- load(args)

载入所有惰性非惰性的属性，但是不包括还没有从数据库恢复的属性。如果一个属性已经被载入过了，将不会再次载入。你可以指定需要载入的属性的列表将他们全部载入，或者指定它的名字让 Pony 只载入他们：

```python
obj.load(Person.biography, Person.some_other_field)
obj.load('biography', 'some_other_field')
```

- select()
- select(lambda[, globals[, locals])

选取满足 lambda 指定的筛选条件的对象。如果没有指定 lambda 将选中所有对象。

`select` 方法返回 `Query` 类型的实例。实体对象的实例将会在枚举 `Query` 对象的时候一次性从数据库中请求。例如：
`Product.select(lambda p: p.price > 100 and count(p.order_items) > 1)[:]`

上面的请求返回了所有价格高于 100，售出大于一次的所有产品。

- select_random(limit)
选中 `limit` 个随机对象。这个方法使用的逻辑比 `ORDER BY RANDOM()` SQL 语句更高效。这个方法使用如下逻辑：
1. 确定表中的最大 id
2. 在范围内（0，max_id）生成一些随机的 id
3. 将这些随机的 id 还原成对象。如果某个 id 的对象不存在（例如，被删除了），尝试另外一个 id

只要需要就持续重复步骤 2-3，直到获得足够多的对象。

即使在数据很多的表中这个逻辑也不影响性能，但是这个方法也有一些限制：

- 主件必须是顺序的整数类型
- 两个 id 之间的间隙（已删除的对象的个数）应该相对的小。

如果你的请求没有任何特定的标准，你可以使用 `select_random`，如果需要其他限定，你可以使用 `Query` 对象的 `random()`方法。

## 实体对象实例的方法
class Entity

- get_pk()
返回对象的主键的值。
```python
>>> c = Customer[1]
>>> c.get_pk()
1
```
如果主键是组合键，这个方法返回一个包含主键值的元组。
```python
>>> oi = OrderItem[1,4]
>>> oi.get_pk()
(1, 4)
```

- delete()
删除一个实体对象的实例。对象将被标记为删除，当离开 `db_session` 或者当发送下一条数据库请求之前，将会自动调用 `flush()` 提交到当前事务。

- set(**kwargs)
一次性设置对象的多个属性：
```python
Customer[123].set(email='new@example.com', address='New address')
```
这个方法在使用字典更新一个对象的多个属性时很方便:
```python
d = {'email': 'new@example.com', 'address': 'New address'}
Customer[123].set(**d)
```
- to_dict(only=None, exclude=None, with_collections=False, with_lazy=False, related_objects=False)

返回一个属性的名字和值的字典。这个方法在将对象序列化为 JSON 或其他格式的时候很有用。
默认情况下，这个方法不包含集合（对多关系）和惰性属性。如果一个属性的值是实体对象实例，那么只有主键会被加入到字典。

`only` - 如果你只想得到指定的属性，可以使用这个参数。这个属性可以作为第一个位置参数。你可以指定一个属性名字的列表 `obj.to_dict(['id', 'name'])`, 用空格分开的字符串 `obj.to_dict('id name')`，或者用逗号分隔的字符串 `obj.to_dict('id, name')`。

`exclude` - 这个参数允许你排除指定的属性。属性的名字可以使用类似 `only` 中的指定方式。

`related_objects` - 默认情况下，所有关联的对象将会显示为主键的形式。如果 `related_objects=True`，与当前对象有关系的对象都将会被加入到对象的结果字典中，不包含主键。如果你原本打算遍历关联的对象然后递归调用 `to_dict()` 方法的，这是一个有用的选项。

`with_collections` - 默认情况下，结果字典不包含集合（对多关系）。如果你设置了这个参数为 `True`，那么对多关系将会以列表的形式展示。如果 `related_objects=False`（默认情况），这些列表将会由关联的实例的主键组成。如果 related_objects=True，这些列表将会以对象的列表展示。

`with_lazy` - 如果为 `True`，那么惰性属性（例如，BLBOBs 二进制块和 使用 `lazy=True` 指定的属性）将会被包含在结果字典中。

为了演示这个方法的使用，我们使用 Pony 附带的 eStore 例子。让我们获取一个 id = 1 的用户，然后将他转换为字典。
```python
>>> from pony.orm.examples.estore import *
>>> c1 = Customer[1]
>>> c1.to_dict()

{'address': u'address 1',
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'password': u'***'}
```

如果我们不想序列化密码属性，我们可以用这种方式执行:

```python
>>> c1.to_dict(exclude='password')

{'address': u'address 1',
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith'}
```
如果你想排除多个属性，你可以以列表的指定它们：`exclude=['id', 'password']`、 `exclude='id, password'`、 `exclude='id password'`。

你也可以通过 `only` 来指定需要序列化的属性。

```python
>>> c1.to_dict(only=['id', 'name'])

{'id': 1, 'name': u'John Smith'}

>>> c1.to_dict('name email') # 'only' parameter as a positional argument

{'email': u'john@example.com', 'name': u'John Smith'}
```
默认情况下，集合不包含在结果字典中。如果你想要包含他们，你想要要指定 `with_collections=True`。也可以在 `only` 参数中指定集合属性。

```python
>>> c1.to_dict(with_collections=True)

{'address': u'address 1',
'cart_items': [1, 2],
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'orders': [1, 2],
'password': u'***'}
```
默认情况下，所有关联的对象（卡，账单）显示为它们的主键的列表。如果你想要得到关联的对象的实例，你可以使用 `related_objects=True`:
```python
>>> c1.to_dict(with_collections=True, related_objects=True)

{'address': u'address 1',
'cart_items': [CartItem[1], CartItem[2]],
'country': u'USA',
'email': u'john@example.com',
'id': 1,
'name': u'John Smith',
'orders': [Order[1], Order[2]],
'password': u'***'}
```
如果你需要序列化多个实体对象实例，或者序列化一个关联者别的对象的实例，你可以使用 `pony.orm.serialization` 模块中的函数 `to_dict`。

- flush()
将对象的改变保存到数据库中。通常 Pony 会自动保存改变，你不需要自己调用这个方法。当你想要获取一个新增对象的主键，而主键是数据库自动累加的情况下，你可以使用这个方法。

## 实体对象钩子
有时候，你可能需要在实体对象的实例即将在数据库中创建、更新或删除时执行一个动作。为了达到这个目的，你可以使用实体对象的钩子。你可以在实体对象定义的时候对如下函数做你自己的实现。

class Entity
- before_insert()
只有在新创建的对象插入到数据库的时候被调用。

before_update()
只有在对象更新修改到数据库的时候被调用。

before_delete()
只有在对象从数据库删除的时候被调用。

after_insert()
当数据库中插入了新的行之后调用。

after_update()
当数据库中更新了一行之后调用。

after_delete()
只有在对象从数据库删除的之后被调用。

```python
>>> class Message(db.Entity):
...     title = Required(str)
...     content = Required(str)
...     def before_insert(self):
...         print "Before insert!"

>>> m = Message(title='First message', content='Hello, world!')
>>> commit()
Before insert!
INSERT INTO "Message" ("title", "content") VALUES (?, ?)
[u'First message', u'Hello, world!']
```

## 序列化实体对象的实例

### 使用 pickle 做序列化
Pony 允许序列化实体对象实例、请求结果和集合。当你想要讲实体对象的实例存储到外部的缓存中时( 例如, memcache )。当 Pony 序列化实体对象的实例的时候，会保存除了集合之外的所有属性，为了避免序列化一个超大的序列。如果你想要序列化一个集合属性，你需要单独序列化它。例如：
```python
>>> from pony.orm.examples.estore import *
>>> products = select(p for p in Product if p.price > 100)[:]
>>> products
[Product[1], Product[2], Product[6]]
>>> import cPickle
>>> pickled_data = cPickle.dumps(products)
```
现在我们可以将序列化之后的数据放到缓存，稍后，当我们再次需要实例的时候，我们可以反序列化它：
```python
>>> products = cPickle.loads(pickled_data)
>>> products
[Product[1], Product[2], Product[6]]
```
为了提升性能，你可以将对象序列化存储到外部缓存。当你反序列化一个对象的时候，Pony 将他们放入当前 `db_session`，看起来就像它们是从数据库中载入的。即使对象的当前状态和数据库中的不符，Pony 也不做检查。

### 序列化为字典并转为 JSON
另外一种序列化实体对象实例的方法是使用实体对象实例的 `to_dict()` 方法，或者使用 `pony.orm.serialization` 中的 `to_dict()` 和 `to_json()` 函数。实例的 `to_dict()` 方法返回对应对象的键值字典结构。有时候你需要序列化的不是实例自己，而是和实例关联的对象。这种情况下你可以使用下面介绍的 `to_dict()` 函数。

- to_dict()
 这个函数用作将实体对象实例序列化为字典。它接收一个实体对象的实例或者任何返回实体对象实例的迭代器。这个函数返回一个多级字典，其中包含传递给函数的对象以及和她直接关联的对象。这里有一个结果字典的结构：

```python
{
    'entity_name': {
        primary_key_value: {
            attr: value,
            ...
        },
        ...
    },
    ...
}
```
让我们使用在线商店的模型例子（ [ER diagram](https://editor.ponyorm.com/user/pony/eStore) ），打印 `to_dict()` 函数的结果。注意，我们需要单独导入 `to_dict `。

```python
from pony.orm.examples.estore import *
from pony.orm.serialization import to_dict

print to_dict(Order[1])

{
    'Order': {
        1: {
            'id': 1,
            'state': u'DELIVERED',
            'date_created': datetime.datetime(2012, 10, 20, 15, 22)
            'date_shipped': datetime.datetime(2012, 10, 21, 11, 34),
            'date_delivered': datetime.datetime(2012, 10, 26, 17, 23),
            'total_price': Decimal('292.00'),
            'customer': 1,
            'items': ['1,1', '1,4'],
            }
        }
    },
    'Customer': {
        1: {
            'id': 1
            'email': u'john@example.com',
            'password': u'***',
            'name': u'John Smith',
            'country': u'USA',
            'address': u'address 1',
        }
    },
    'OrderItem': {
        '1,1': {
            'quantity': 1
            'price': Decimal('274.00'),
            'order': 1,
            'product': 1,
        },
        '1,4': {
            'quantity': 2
            'price': Decimal('9.98'),
            'order': 1,
            'product': 4,
        }
    }
}
```
上面的例子中，结果包含了序列化的 `Order[1]` 的实例，和直接关联的对象 `Customer[1]`, `OrderItem[1, 1]` 和 `OrderItem[1, 4]`。

- to_json()

这个函数使用了 `to_dict()` 的输出，将它的 JSON 形式返回。

## 使用原始 SQL
即使 Pony 几乎能够将任意的 Python 写的条件转为 SQL，有些时候还是需要原始 SQL ，例如，为了调用预置的程序或者使用指定的数据库的方言特性。在这种情况下， Pony 允许用户已原始 SQL 的方式编写请求，将它们作为字符串放在 `select_by_sql()` 或 `get_by_sql()` 中。
`products = Product.select_by_sql("SELECT * FROM Products")`
不同于 `select()`, `select_by_sql` 方法不返回 `Query` 对象，而是实体对象的列表。

使用以下语法可以将参数传递给 SQL 字符串：“$name_variable” or “$(expression in Python)”

```python
x = 1000
y = 500
Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)")
```

当 Pony 碰到 SQL 中的参数的时候，会从当前框架中获取变量的值（从 globals 和 locals），或者从以参数形式传递的字典中获取。

```python
Product.select_by_sql("SELECT * FROM Product WHERE price > $x OR price = $(y * 2)",
                       globals={'x': 100}, locals={'y': 200})
```

跟随在 $ 符号之后的变量和表达式，将会以参数的形式自动计算和传递到数据库请求中，已做了 SQL 注入检测。Pony 自动将请求字符串中的 $x 替换为 ”?”, “%S” 或者当前数据库系统中使用的其他参数形式。

如果请求中自己用到 `$` （例如，系统表的名字），它必须被写成两个连接的 `$`，就像这样 `$$`。

## 将对象存入数据库

### 保存对象的顺序
通常情况下，Pony 将对象按照创建和修改的顺序存入数据库。有些情况，Pony 可以重新设计 SQL INSERT 语句，如果保存对象的时候需要。让我们考虑如下情况：
```python
from pony.orm import *

db = Database('sqlite', ':memory:')

class TeamMember(db.Entity):
    name = Required(str)
    team = Optional('Team')

class Team(db.Entity):
    name = Required(str)
    team_members = Set(TeamMember)

db.generate_mapping(create_tables=True)
sql_debug(True)

with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    team = Team(name='Tenacity', team_members=[john, mary])
```
在上面的例子中，我们创建了两个团队成员和一个团队对象，将这两个成员分派给这个团队。团队成员和团队之间的关系是在团队成员的表中声明的一个字段。

```python
CREATE TABLE "Team" (
  "id" INTEGER PRIMARY KEY AUTOINCREMENT,
  "name" TEXT NOT NULL
)

CREATE TABLE "TeamMember" (
  "id" INTEGER PRIMARY KEY AUTOINCREMENT,
  "name" TEXT NOT NULL,
  "team" INTEGER REFERENCES "Team" ("id")
)
```
当 Pony 创建了 `john`、 `mary` 和 `team` 对象，它明白必须要重排 SQL INSERT 语句的顺序，首先在数据库中创建 `Team` 对象的实例，那样才能在团队成员的字段中保存团队的 id。
```python
INSERT INTO "Team" ("name") VALUES (?)
[u'Tenacity']

INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
[u'John', 1]

INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
[u'Mary', 1]
```

### 保存对象的循环链
现在让我们设想，我们想要给团队设置队长。为了达到这个目的，我们需要给我们的实体对象增加一对属性： `Team.captain` 和反转的 `TeamMember.captain_of`
```python
class TeamMember(db.Entity):
    name = Required(str)
    team = Optional('Team')
    captain_of = Optional('Team')

class Team(db.Entity):
    name = Required(str)
    team_members = Set(TeamMember)
    captain = Optional(TeamMember, reverse='captain_of')
```
创建实体对象的实例并指定队长的代码如下：
```python
with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    team = Team(name='Tenacity', team_members=[john, mary], captain=mary)
```
当 Pony 执行上面的代码的时候，会抛出异常：
`pony.orm.core.CommitException: Cannot save cyclic chain: TeamMember -> Team -> TeamMember`

为什么会这样？让我们看看。Pony 看到要保存 `john` 和 `mary` 对象到数据库，必须知道团队的 id，然后就尝试重排 insert 语句。但是当保存 `team` 对象的时候需要指定 `captain` 属性，那又需要知道 `mary` 对象的属性。这种情况下， Pony 不能解决这个循环链，然后抛出了异常。

为了保存这样一个循环链，你需要添加 `flush()` 命令帮助 Pony：
```python
with db_session:
    john = TeamMember(name='John')
    mary = TeamMember(name='Mary')
    flush() # saves objects created by this moment in the database
    team = Team(name='Tenacity', team_members=[john, mary], captain=mary)
```
这种情况下，Pony 将会先将 `john` 和 `mary` 对象保存到数据库。之后使用 SQL UPDATE 语句建立和 `team` 之间的关系。

```python
INSERT INTO "TeamMember" ("name") VALUES (?)
[u'John']

INSERT INTO "TeamMember" ("name") VALUES (?)
[u'Mary']

INSERT INTO "Team" ("name", "captain") VALUES (?, ?)
[u'Tenacity', 2]

UPDATE "TeamMember"
SET "team" = ?
WHERE "id" = ?
[1, 2]

UPDATE "TeamMember"
SET "team" = ?
WHERE "id" = ?
[1, 1]
```
