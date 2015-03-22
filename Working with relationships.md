# 关系的使用

在 Pony 中，一个实体对象可以和其他实体对象建立关系。每个关系有两端，分别由两个实体对象的属性定义。

```python
class Person(db.Entity):
    cars = Set('Car')

class Car(db.Entity):
    owner = Optional(Person)
```
在上面的例子中，我们在 `Person` 和 `Car` 实体对象之间，使用属性 `cars` 和 `owner` 定义了一个个一对多关系。让我们给我们的实体对象增加一组数据属性，然后尝试是一些例子：
```python
from pony.orm import *

db = Database('sqlite', ':memory:')

class Person(db.Entity):
    name = Required(str)
    cars = Set('Car')

class Car(db.Entity):
    make = Required(str)
    model = Required(str)
    owner = Optional(Person)

db.generate_mapping(create_tables=True)
```
现在让我们来创建 `Person` 和 `Car` 的实例：
```python
>>> p1 = Person(name='John')
>>> c1 = Car(make='Toyota', model='Camry')
>>> commit()
```
通常，在你的程序中，你不需要手动调用 `commit` 函数，因为 `db_session` 会自动调用。但是当你在交互器中工作的时候，因为没有离开 `db_session` 的动作，所以如果需要将数据存储到数据库，就得手动提交。

## 关系的建立
刚刚我们建立了实例 `p1` 和 `c1`,但是他们之间没有建立关系。然我们检查一下关系属性的值。
```python
>>> print c1.owner
None

>>> print p1.cars
CarSet([])
```
`cars` 属性的值是空的。
现在让我们建立这两个实体对象之间的关系：
```python
>>> c1.owner = p1
```
如果我们现在打印关系属性的值，我们会看到：
```python
>>> print c1.owner
Person[1]

>>> print p1.cars
CarSet([Car[1]])
```

当我们给 `Car` 的实例指定了所有者之后，`Person` 的关系属性 `cars` 也同时改变了。

我们也可以在创建 `Car` 的实例的时候，通过指定关系属性来建立关系。
```python
>>> p1 = Person(name='John')
>>> c1 = Car(make='Toyota', model='Camry', owner=p1)
```
在我们的例子中，属性 `owner` 是可选的，我已我们可以在任何时候给它赋值， 在创建 `Car` 实例的时候，或者以后。

## 集合的操作
`Person` 的 `cars` 属性被声明为集合，因此我们可以使用集合的方式来操作： add、remove、in、len、clear。

你可以使用 `add()` 和 `remove()` 方法增加或移除关系。

```python
>>> p1.cars.remove(Car[1])
>>> print p1.cars
CarSet([])

>>> p1.cars.add(Car[1])
>>> print p1.cars
CarSet([Car[1]])
```
你可以检查一个对象是否包含在集合中：
```python
>>> Car[1] in p1.cars
True
```

或者，确定一个对象不在这个集合中：
```python
>>> Car[1] not in p1.cars
False
```

获取集合的大小：
```python
>>> len(p1.cars)
1
```

这里有几种方法让你创建一个属于特定的 `person` 实例的 `car`。

有一种选择是使用 `create()` 方法：
```python
>>> p1.cars.create(model='Toyota', make='Prius')
>>> commit()
```

现在我们可以验证， `Car` 的实例已经被添加到 `Person` 实例的 `cars` 集合属性了：
```python
>>> print p1.cars
CarSet([Car[2], Car[1]])
>>> p1.cars.count()
2
```

你可以枚举遍历一个集合属性：
```python
>>> for car in p1.cars:
...     print car.model
Toyota
Camry
```

## 属性提升

在 Pony 中，集合属性提供了属性提升的能力：一个集合获取她的元素的属性。
```python
>>> show(Car)
class Car(Entity):
    id = PrimaryKey(int, auto=True)
    make = Required(str)
    model = Required(str)
    owner = Optional(Person)
>>> p1 = Person[1]
>>> print p1.cars.model
Multiset({u'Camry': 1, u'Prius': 1})
```
这里，我们使用 `show()` 方法打印了实体对象类型的属性，然后打印了 `cars` 关系属性的 `model` 方法。`cars` 属性拥有所有 `Car` 实力类型的属性: `id`、 `make`、 `model` 和 `owner`。在 Pony 中，我们把这个叫做多重集合，是使用字典实现的。字典的键显示了属性的值 —— 例如我们的例子中的，“Camry” 和 “Prius” 。字典的值表示了集合中这些值出现的次数。
```python
>>> print p1.cars.make
Multiset({u'Toyota': 2})
```
`Person[1]` 有两辆 Toyota 。
我们可以枚举这个多重集合：
```python
>>> for m in p1.cars.make:
...     print m
...
Toyota
Toyota
```

### 多重集合
待决[TBD]

- 聚合函数
- 明确的
- 请求中的子请求

### 集合属性的参数
集合属性用来定义“对多”一边的关系。他们可以用来定义一对多或多对多关系。例如：
```python
class Photo(db.Entity):
    tags = Set('Tag', lazy=True, table='Photo_to_Tag')

class Tag(db.Entity):
    photos = Set(Photo)
```
这里，`tags` 和 `photos` 都是集合。
下面是能够在集合属性定义的时候指定的参数。

class Set
定义对多关系

- lazy
当访问指定的集合中的元素的时候（检查一个元素是否属于集合，增加或删除元素），Pony 总是将整个集合载入到 `db_session` 的缓存中。通常这能减少数据库请求从而提高性能。但是如果你的集合特别大，你最好不要把它们载入到内存。设置 `lazy=True` 告诉 Pony 不能将集合载入到内存，而是每次都发送数据库请求。默认情况 `lazy=False`。

- reverse
指定要绑定关系的实体对象的属性的名字。当在两个实体对象之间有多个关系的时候使用。

- table
这个参数只有在多对多关系中使用，让你可以指定数据库中关系中间表的名字。

- column
- columns
- reverse_column
- reverse_columns
这些参数仅用于多对多关系，允许你指定中间列的名字。`columns` 和 `reverse_columns` 参数接收一个列表，当实体对象使用组合键的时候使用。通常使用 `column` 或 `columns`，如果你不喜欢列的名字，可以在所有的关系属性中使用。

- cascade_delete
布尔值，控制着关联的对象的级联删除。默认值基于关系另一边的定义。如果那边是 `Optional` - 默认值是 `False`，如果那边是 `Required`, 默认值是 `True`。

- nplus1_threshold
这个参数用来微调 N+1 问题的阈值。

### 集合属性的方法
你可以像 Python 的集合那样使用集合属性，标准操作有 : `in`、 `not in`、 `len`：
class Set

- len()
返回集合中对象的个数。如果集合没有被载入缓存，这个方法会现将集合载入缓存，然后返回对象的数量。只有当你之后想要遍历枚举这些对象并要将它们载入内存的时候才使用这个方法。如果你不需要将集合再入内存，你可以使用个 `count` 方法。
```python
>>> p1 = Person[1]
>>> Car[1] in p1.cars
True
>>> len(p1.cars)
2
```
还有一些你可以在集合属性上调用的方法。
class Set
下面是一些你可以在“对多”关系属性上调用的方法。

- add(item)
- add(iter)
增加实例到集合，建立两个实体对象实例之间的双向关系。
```python
photo = Photo[123]
photo.tags.add(Tag['Outdoors'])
```
现在 `Photo` 实体对象的主键为 123 的实例和 `Tag['Outdoors']` 实例建立了关系。`Tag['Outdoors']` 的  `photos` 属性也包含了 `Photo123` 对象。

通过给 `add()` 方法传递一个列表，我们也可以一次性建立多个关系：
`photo.tags.add([Tag['Party'], Tag['New Year']])`

- remove(item)
- remove(iter)
在集合中移除一个或多个元素，这样就断开了实体对象实例间的关系。

- clear()
从集合中移除所有元素，同时实体对象间的关系也断开了。

- id_empty()
如果实例没有关联的兑现，返回 `True`。如果实例至少有一条对象，返回 `False`。

- copy()
返回一个 Python 中的 `set` 对象，其中包含了给定集合的所有元素。

- count()
返回集合中对象的个数。这个方法不会将集合中的对象都载入到缓存，而是生成一个 SQL 请求，从数据库中查询对象的个数。如果你之后要使用集合对象（枚举集合对象，使用对象的属性），你可以使用 `len()` 方法。

- creat(**kwargs)
创建一个关联的实体对象的实例，并建立和它的关系：
```python
new_tag = Photo[123].tags.create(name='New tag')
```
相当于：
```python
new_tag = Tag(name='New tag')
Photo[123].tags.add(new_tag)
```

- load()
将所有关联的对象从数据库中载入。

### 集合类的方法
这个方法可以使用实体对象类型定义，为不是实例。例如：
```python
from pony.orm import *
db = Database('sqlite', ':memory:')

class Photo(db.Entity):
    tags = Set('Tag')

class Tag(db.Entity):
    photos = Set(Photo)

db.generate_mapping(create_tables=True)
Photo.tags.drop_table() # drops the Photo-Tag intermediate table
```

class Set

- drop_table(with_all_data=False)
删除为了创建多对多关系的中间表。如果表是空的，而且 `with_all_data=False`，这个方法抛出异常： `TableIsNotEmpty`，而且不会删除任何东西。设置 `with_all_data=True`，允许你删除非空的表。

### 集合的请求

从 release 0.6.1 开始，Pony 引入了关系属性的查询。

你可以在对多关系中使用 `select()`、 `Query.filter()`、 `Query.order_by()`、 `Query.page()`、 `Query.limit()`、 `Query.random()` 方法。`select` 和 `filter` 是一样的。

下面我们列举几个使用这些方法的例子。我们将使用大学课程作为请求展示。已经定义了 Python 实体对象和关系表。

下面的例子选取了，group 1 中 gpa 大于 3 的所有学生。
```python
g = Group[101]
g.students.filter(lambda student: student.gpa > 3)[:]
```

这个请求用于显示，group 1 中按名字排序，第二页的学生：
`g.students.order_by(Student.name).page(2, pagesize=3)`

这个请求也可以这么写：
`g.students.order_by(lambda s: s.name).limit(3, offset=3)`

下面的请求返回 group 1 中随机的两个学生。
`g.students.random(2)`

接下来，这个请求返回了 `Student[1]` 第二学期选择的课程，按名字排序的第一页。

```python
s = Student[1]
s.courses.select(lambda c: c.semester == 2).order_by(Course.name).page(1)
```
