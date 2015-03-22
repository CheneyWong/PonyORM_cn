# 开始使用 Pony

使用下面的命令安装 Pony
`pip install pony`
Pony 可以被安装在 Python2 2.6以上版本，没有别的扩展。

为了确认 Pony 安装正常，打开 Python 交互器，输入：
`>>> from pony.orm import *`
这样将导入所有使用 Pony 所必须的类和函数（不是特别大）。实际上你也可以自己选择导入哪些内容，但是我们建议刚开始的时候还是使用先`import *`

熟悉 Pony 的最好的方法是在交互器下好好玩玩。让我们创建一个示例数据库用来存放 `Person`类的实体，给它添加三个对象，然后写一个查询。

## 创建数据库
在 Pony 中，对象的实体是存储在数据库中的，所以我们必须先创建一个数据库，在交互器中输入：
`>>> db = Database('sqlite', ':memory:')`
这个命令创建了对象和数据库的连接，第一个参数指出了我们要使用的 DBMS ，目前 Pony 支持 4 种类型的数据库：'sqlite', 'mysql', 'postgresql' 和 'oracle'。接下来的参数跟 DBMS 是关联的；它指定了你要要通过 DB-API 模块链接的数据库。对于 sqlite，必须指定数据库的创建位置，要么是数据库的名字，要么是一个字符串`:memory:`，如果数据库的未知是内存中，它将在 Python 交互器会话关闭的时候被删除。

如果需要在文件中存储的数据库，可以使用下面的代码替带。
`>>> db = Database('sqlite', 'test_db.sqlite', create_db=True)`
这样，如果数据库文件不存在，将会自动创建一个新的。我们的例子中还是使用内存的数据库。

## 定义实体对象
现在，让我们创建两个实体对象，Person 和 Car 。Person 有两个属性，name 和 age ，Car 有两个属性，make 和 model。这两个对象是一对多的关系。在 Python 交互器中输入如下代码：
```python
>>> class Person(db.Entity):
...     name = Required(str)
...     age = Required(int)
...     cars = Set("Car")
...
>>> class Car(db.Entity):
...     make = Required(str)
...     model = Required(str)
...     owner = Required(Person)
...
>>>
```
我们创建的类是从 `db.Entity` 继承来的，这意味着它们不是普通的类型，而是实例存储在 `db`对应的数据库中的实体对象。Pony 允许我们同时支持多个数据库，但是每个实体只能属于一个数据库。

在 Person 中，我们有三个属性，`name`，`age`和`cars`。`name`和`age`是必须的属性，也就是说，它们是非 `None` 的，`name`是个字符串，`age`是个数字。 

其中 `cars` 属性设置了 `Car` 类型，这是一条关系声明。它能够存储一系列 `Car` 类型的实例， `"Car"` 作为字符串出现，是因为类型 `Car` 在那个时候还没有定义。

`Car` 实体对象有三个必须的属性。`make` 和 `model` 是字符串，`owner` 是一对多关系的另外一边。Pony 中的关系总是需要被分别出现在关系两边的类型中的两条语句所定义。

如果我们需要创建一个多对多关系，在关系两头的类型中，我们都要声明 `Set` 属性。Pony 会自动创建中间的关系表。

在 Python3 中 `str` 表示 unicode 字符串，Python2 中有两种方式表示字符串，`str` 和 `unicode`。从 Pony Release 0.6 开始，无论是使用 `str` 还是 `unicode`，都表示 unicode 字符串。我们建议使用 `str` 作为字符串声明，因为这样在 Python3 中看起来自然一点。

如果你想在交互器中查看一个实例类型的定义，可以使用 `show()` 函数。传递一个一个实例类型给这个函数可以看到对应的定义。

```python
>>> show(Person)
class Person(Entity):
    id = PrimaryKey(int, auto=True)
    name = Required(str)
    age = Required(int)
    cars = Set(Car)
```
你可能注意到了，实例类型多了一个额外的叫 `id` 的属性。为什么会这样？

每个实例都必须包含一个主键，用以区分不同的实例。由于我们没有手动设置主键，所以它自动创建了一个。如果主键是自动创建的，那它就是一个名字叫 `id` 的数字格式。如果主键是手动创建的，那么可以指定它的名字，类型也可以是数字或者字符串。Pony 也支持多主键。

当主键是自动创建时，它的 `auto` 属性总是被定义为 `True`。也就是说，它的值会自动增加，使用了数据库的计数器或者序列功能。

## 映射实例到数据库
现在我们需要创建数据表以存储对象的数据，我们可以通过调用 `Database` 对象的如下方法来实现：
`>>> db.generate_mapping(create_tables=True)`
参数`create_tables=True`标志了如果数据表不存在就调用 `CREATE TABLE` 明亮创建它们。

所有需要映射到数据库的实例都必须在调用 `generate_mapping()` 之前声明。

## 延后数据库绑定

从 Pony release 0.5 开始，有一个额外的方法来指定数据库参数。现在你可以先创建数据库对象，然后就可以定义实例的属性了，最后再将数据库对象绑定到指定的数据库。
```python
### module my_project.my_entities.py
from pony.orm import *

db = Database()
class Person(db.Entity):
    name = Required(str)
    age = Required(int)
    cars = Set("Car")

class Car(db.Entity):
    make = Required(str)
    model = Required(str)
    owner = Required(Person)
```
```python
### module my_project.my_settings.py
from my_project.my_entities import db

db.bind('sqlite', 'test_db.sqlite', create_db=True)
db.generate_mapping(create_tables=True)
```

这样，你可以分离实例定义和将它绑定到指定数据库的过程。这对测试会有帮助。

## 使用 debug 模式
Pony 允许你在屏幕上查看发送到数据库的 SQL 命令（或者在 log 文件，如果是配置了的）。输入如下命令可以打开这个模式：
`>>> sql_debug(True)`
如果这个命令是在 `generate_mapping()` 之前调用的，那么当创建表时，你也能看到生成表的 SQL 语句。

Pony 默认将调试信息输出到 stdout，如果你导入了 Python 的标准日志模块，Pony 将会使用它替代 stdout。使用 Python 的标准模块，你可以将调试信息保存到文件。

```python
import logging
logging.basicConfig(filename='pony.log', level=logging.INFO)
```
注，必须配置 `level=logging.INFO`，因为日志模块默认的输出等级是 WARNING ，Pony 默认使用 INFO 等级输出信息。Pony 有两个日志记录者： `pony.orm.sql` 记录发送到数据库的 SQL 语句， `pony.orm` 记录其他的。

## 创建实例对象并插入数据库
现在，让我们创建 5 个对象，描述 3 个人和 2 辆车，然后将这些信息存入数据库。使用如下命令实现：

```python
>>> p1 = Person(name='John', age=20)
>>> p2 = Person(name='Mary', age=22)
>>> p3 = Person(name='Bob', age=30)
>>> c1 = Car(make='Toyota', model='Prius', owner=p2)
>>> c2 = Car(make='Ford', model='Explorer', owner=p3)
>>> commit()
```

Pony 并不是在对象创建的时候就就将它们存入数据库的，而是只有当 `commit()` 命令执行的时候才会保存。如果在调用 `commit()` 命令之前开启了 debug 模式，你将会看到 5 个 `INSERT` 命令将对象存入到了数据库。

## 编写查询命令
现在我们的数据库中存储了 5 个对象，我们可以尝试一些查询语句。例如，如下命令就返回了一个年龄大于 20 的 person 的列表。
```pyhton
>>> select(p for p in Person if p.age > 20)
<pony.orm.core.Query at 0x105e74d10>
```
`select` 函数将 Python 的生成器翻译为 SQL 查询语句，返回一个 `Query` 类型的实例。当遍历这个 query 的时候， SQL 查询语句被发送了一次。获取对象的列表的一个方法是对它进行一次切片操作 `[:]`。
```python
>>> select(p for p in Person if p.age > 20)[:]

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."age" > 20

[Person[2], Person[3]]
```
在返回结果中，你可以看到发送到数据库的 SQL 查询语句和提取的对象的列表。当我们 print 一个查询结果时，对象的实例由实例类型的名字和方括号中的主键组成：`Person[2]`。

我们可以用过 `order_by` 方法对查询结果排序。如果我们只想要结果的一部分，可以使用 Python 中的切片的方法。例如，我们让 people 按照名字排序，提取前两个对象，可以这么写：
```python
>>> select(p for p in Person).order_by(Person.name)[:2]

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
ORDER BY "p"."name"
LIMIT 2

[Person[3], Person[1]]
```
有时候在交互器模式下，我们想要以列表的形式查看对象所有属性的值。这可以通过对查询结果列表调用 `.show()`方法来实现。
```python
>>> select(p for p in Person).order_by(Person.name)[:2].show()

SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
ORDER BY "p"."name"
LIMIT 2

id|name|age
--+----+---
3 |Bob |30
1 |John|20
```
`.show()` 方法不显示“对多”属性，因为那将需要调用额外的查询，而且或许会很庞大。那就是为什么之前你看不到下挂的 car 的信息。但是如果对象有“对一”属性，那它将一并显示。
```python
>>> Car.select().show()
id|make  |model   |owner
--+------+--------+---------
1 |Toyota|Prius   |Person[2]
2 |Ford  |Explorer|Person[3]
```
如果我们不需要对象的列表，但是需要遍历结果集，可以直接使用 `for` 循环，不需要切片操作。
```python
>>> persons = select(p for p in Person if 'o' in p.name)
>>> for p in persons:
...     print p.name, p.age
...
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."name" LIKE '%o%'

John 20
Bob 30
```
在上面的例子中，我们获取到所有的名字中带有小写字母 'o' 的人，然后将名字和年龄都打印出来。

查询也不是必须要返回实例对象。例如我们可以获取对象的列表。
```python
>>> persons = select(p for p in Person if 'o' in p.name)
>>> for p in persons:
...     print p.name, p.age
...
SELECT "p"."id", "p"."name", "p"."age"
FROM "Person" "p"
WHERE "p"."name" LIKE '%o%'

John 20
Bob 30
```
或者一个元组
```python
>>> select((p, count(p.cars)) for p in Person)[:]

SELECT "p"."id", COUNT(DISTINCT "car-1"."id")
FROM "Person" "p"
  LEFT JOIN "Car" "car-1"
    ON "p"."id" = "car-1"."owner"
GROUP BY "p"."id"

[(Person[1], 0), (Person[2], 1), (Person[3], 1)]
```
上面的例子中我们就获取了一个由 person 和他拥有的 cars 的数量做成的元组的列表。

我们也可以运行集合查询。以下是一个查询最大用户年龄的例子。
```python
>>> print max(p.age for p in Person)
SELECT MAX("p"."age")
FROM "Person" "p"

30
```
Pony 允许你编写比目前示例中复杂的多的查询语句。请参考手册的以下章节。

## 获取对象
如果要通过主键查询对象，可以在方括号中填写主键。
```python
>>> p1 = Person[1]
>>> print p1.name
John
```
你可能会注意到，实际上并没有给数据库发送数据。那是因为这个对象已经在 session 缓存中存在了。缓存可以减少提交到数据库的请求数量。
通过其他属性获取对象：
```python
>>> mary = Person.get(name='Mary')

SELECT "id", "name", "age"
FROM "Person"
WHERE "name" = ?
[u'Mary']

>>> print mary.age
22
```
这种情况下，即使对象已经被缓存，查询依然会被发送到数据库，因为 `name` 不是唯一性的键。数据库 session 缓存只会在通过主键或唯一性键查询时起作用。

可以通过传递实例对象到 `show()` 函数，用于显示所属实例类和属性值。

```python
>>> show(mary)
instance of Person
id|name|age
--+----+---
2 |Mary|22
```

## 更新一个对象
```python
>>> mary.age += 1
>>> commit()
```
Pony 持续跟踪对象属性的变化。当 `commit()` 执行时，在此期间所有改变的对象都将同步到数据库。Pony 只会更新改变了的属性。

## db_session
当你使用 Python 交互器来操作时，你不用考虑数据库会话的问题，因为 Pony 已经自动替你维护了。但是当你在应用中使用 Pony 时，所有的数据库交互都需要基于数据库会话。为了达到这个目的，你需要用修饰器 `@db_session` 包装你的需要数据库操作的函数。
```python
@db_session
def print_person_name(person_id):
    p = Person[person_id]
    print p.name
    # database session cache will be cleared automatically
    # database connection will be returned to the pool

@db_session
def add_car(person_id, make, model):
    Car(make=make, model=model, owner=Person[person_id])
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```
`@db_session` 修饰器在函数退出后执行几个非常重要的操作：
- 如果函数抛出异常，执行回滚操作
- 如果没有异常，数据被改变了，执行 commits
- 归还数据库连接到连接池
- 清空数据库 session 缓存

即使一个函数仅仅是读取数据而不会做任何改变，也应该使用`@db_session` 修饰器，为了将数据库连接归还到连接池。

实例对象只有在 `@db_session` 修饰下才有效。如果你需要只用这个对象渲染一个 HTML 模版，也许应该工作在 `db_session` 之下。

另外一个使用 `db_session` 的选择是作为上下文管理器而不是修饰器。
```python
with db_session:
    p = Person(name='Kate', age=33)
    Car(make='Audi', model='R8', owner=p)
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```

## 手动编写 SQL
如果你需要手动编写 SQL ，你可以这样做：
```python
>>> x = 25
>>> Person.select_by_sql('SELECT * FROM Person p WHERE p.age < $x')

SELECT * FROM Person p WHERE p.age < ?
[25]

[Person[1], Person[2]]
```
如果你想直接操作数据库，避免使用实体对象，可以通过 `Database` 对象的 `select` 方法。
```python
>>> x = 20
>>> db.select('name FROM Person WHERE age > $x')
SELECT name FROM Person WHERE age > ?
[20]

[u'Mary', u'Bob']
```

## Pony 示例
相比于直接手动创建模型，从 Pony 已经做好的例子中导入类似的模型是非常简单的。下例是一个简单的在线商店的例子，你可以在 Pony 的网站上看到数据表的结构。https://editor.ponyorm.com/user/pony/eStore 
导入示例的方法：
`>>> from pony.orm.examples.estore import *`

在配置开始时，SQLite 数据库需要创建必要的数据表。为了迁移它们以及示例数据，可以执行如下程序：
`>>> populate_database()`
这个函数将会创建对象，然后把他们放到数据库中。
当对象都被创建了，你可以编写一个查询，例如，你可以找到你可以找到访客最多的城市。
```python
>>> select((customer.country, count(customer))
...        for customer in Customer).order_by(-2).first()

SELECT "customer"."country", COUNT(DISTINCT "customer"."id")
FROM "Customer" "customer"
GROUP BY "customer"."country"
ORDER BY 2 DESC
LIMIT 1
```
这次我们选择 country 对象的集合，通过第二列（访客数量）反转排序，拿出访客数最高的 country 。

在 `pony.orm.examples.estore` 模块中的 `test_queries()`函数，可以找到更多的查询示例。 


