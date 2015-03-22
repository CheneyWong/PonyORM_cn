# 实体对象

实体对象就是对象的状态存储在数据库中的 Python 类型。每个实体对象的实例对应数据库的表中的一行。实体对象一般都代表着真实世界的对象。（例如，客户，产品）

Pony 也提供一个实体对象关系表格编辑器，可以用来创建 Python 实体对象的定义。

在创建实体对象的实例之前，你需要先映射实体对象到数据表。Pony 能够将实体对象映射到已经存在的表或者创建新的表。当映射生成之后，你就可以查询数据或者创建新的实例。

## 定义一个实体对象
每个实体对象属于一个数据库。这就是为什么在定义实体对象之前，需要创建一个 `Database` 类型的对象：
```python
from pony.orm import *

db = Database("sqlite", "database.sqlite", create_db=True)

class MyEntity(db.Entity):
    attr1 = Required(str)
```
Pony 的 Database 对象有一个 `Entity` 属性，这个属性是需要存储在数据库的实体对象的必须继承的基类。

## Entity 属性
Entity 的属性就像类的属性一样，通过语法 `attr_name = kind(type)`定义。

```python
class Customer(db.Entity):
    name = Required(str)
    picture = Optional(buffer)
```
你也可以在属性类型的括号中指定其他类型。
我们将会在之后的章节中详细的讨论。
Pony 有如下几种属性：

- Required
- Optional
- PrimaryKey
- Set

### Required 和 Optional
基本上大部分的属性，都是 `Required` 和 `Optional` 类型。如果属性是被定义为 `Required`，那么任何时候都必须值，`Optional` 属性的值可以是空的。

### Optional 字符串属性
如果这个属性没有赋予值，大部分情况下默认值是 `None`。但是当一个属性是字符串时，默认值是空字符串。在数据库中使用空字符串比使用 `NULL` 更实用。大部分框架也都是这么做的。空字符串也能提高搜索的索引速度。如果你想分配 `NULL` 到一个可选的字符串属性，你会收到一个 `ConstraintError` 异常。你可以通过设置 `nullable=True` 属性改变这种行为。然后空字符串和 `NULL` 在空字符串属性列都将是允许的，但是很少需要这么做。

Oracle 数据库中，空字符串和`NULL`是同样处理的。因此所有的 Oracle 数据库中的 `Optional` 属性， `nullable` 将会自动设置为 `True`。

如果一个字符串属性被作为唯一键或者作为唯一键组合的一部分，它的 `nullable` 将会自动设置为 `True`。

### 主键
`PrimaryKey` 定义了数据库表中的主键。每一个实体都因该有一个主键。如果主键没有明确的指定，Pony 会隐含的创建。让我们看以下例子：
```python
class Product(db.Entity):
    name = Required(str, unique=True)
    price = Required(Decimal)
    description = Optional(str)
```
这个实体的定义方式和下面的方式是等价的：
```python
class Product(db.Entity):
    id = PrimaryKey(int, auto=True)
    name = Required(str, unique=True)
    price = Required(Decimal)
    description = Optional(str)
```
Pony 自动添加的的主键总是名字叫 `id` 类型是 `int`。`auto=True` 参数意味着这个属性将会使用数据的自增或者序列功能自动增加。

如果你自己指定了主键，那么类型和名字都是自定义的。例如我们可以指定 `Customer` 实例对象使用 custormer's 的 email 作为主键。
```python
class Customer(db.Entity):
   email = PrimaryKey(str)
   name = Required(str)
```
### Set
一个 `Set` 属性声明了一个集合。你可以指定另外一个实体对象作为 `Set` 属性的类型。这就是定义“对多”关系的方法，可以是“多对多”或者“一对多”。目前，Pony 还不能用 `Set`使用原始类型。我们计划以后加入这个特色功能。我们将在《使用关系》章节中更详细的讨论。

### Composite keys
Pony 完全支持 Composite keys，为了定义一组唯一键，你需要用 `Required` 定义每个键，然后将它们组合成组合主键。
```python
class Example(db.Entity):
    a = Required(int)
    b = Required(str)
    PrimaryKey(a, b)
```
这里 `PrimaryKey(a, b)` 并没有创建新的属性，而是将括号中指定的键值组合为组合主键。每个实体对象，有且只有一个组合主键。
为了创建第二个组合键，你需要像之前一样声明他们，然后使用 `composite_key`组合他们。
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_key(a, b)
```
在数据库中，`composite_key(a,b)` 将会对应为 `UNIQUE("a","b")`约束条件。
如果只有一个属性需要被设置为唯一的，你可以在创建键的地方指定 `unique=True` 属性。
```python
class Product(db.Entity):
    name = Required(str, unique=True)
```
### Composite indexes
使用 `composite_index()` 就可以直接创建组合索引以加速数据检索，它可以用来组合两个或多个属性：
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_index(a, b)
```
既可以使用属性，也可以使用属性的名字。
```python
class Example(db.Entity):
    a = Required(str)
    b = Optional(int)
    composite_index(a, 'b')
```
如果你想对一个字段创建非唯一性的索引，你可以给这个属性指定 `index` 属性。这个属性将在本章之后的章节详细描述。

## 属性的数据类型
实体对象的属性就像的类型的一个属性那样声明，使用语句 `sttr_name = kind(type)`。定义属性的时候能用的数据类型如下：

- str
- unicode
- int
- float
- Decimal
- datetime
- date
- time
- timedelta
- bool
- buffer - used for binary data in Python 2 and 3
- bytes - used for binary data in Python 3
- LongStr - used for large strings
- LongUnicode - used for large strings
- UUID

`buffer` 和 `bytes` 类型是以二进制形式（BLOB）存储在数据库中的。`LongStr` 和 `LongUnicode` 是以 CLOB 形式存储。

从 Pony Release 0.6 开始，增加了 Python3 的支持，我们建议使用  `str` 和 `LongStr` 替代 `Unicode` 和 `LongUnicode`。

我们知道，Python3 和 Python2 处理字符串的方法是不同的。Python2 提供两种字符串类型 —— `str`(byte string) 和 `unicode` (unicode string)，因为 Python3 中用 `str` 代替了 unicode string ，移除 `unicode`。

在 release 0.6 之前，Pony 将 `str` 和 `unicode` 在数据库中都存储为 unicode 。但是对于 `str`的属性，在读取的时候必须将 unicode 转为 byte string。从 Pony Release 0.6 开始，Python2 中的 `str` 类型和 `unicode` 类型具有相同的行为。`str` 和 `unicode` 定义的属性完全一样 —— 在 Python 中和数据库中都是 unicode string。`LongStr` 和 `LongUnicode` 也是这样。。`LongStr` 现在只是 `LongUnicode` 的别称。如下定义，在 Python 中和数据库中都是 unicode。
```python
attr1 = Required(str)
# is the same as
attr2 = Required(unicode)

attr3 = Required(LongStr)
# is the same as
attr4 = Required(LongUnicode)
``` 
Python2 里，如果你需要表示字节数组，可以使用 `buffer` 类型，Python3 里，已经移除了 `buffer` 类型，可使 `bytes` 达到相同的目的。但是为了向后兼容性，我们依然保留了 `buffer` 作为 `bytes` 的别称。如果你使用 `import * from pony.orm`，别称也就已经导入了。

如果你的程序需要兼容 Python2 和 Python3 ，你应该使用 `buffer` 类型来表示二进制的属性。如果你的程序值运行在 Python3 下，你可以使用 `bytes`。
```python
attr1 = Required(buffer) # Python 2 and 3

attr2 = Required(bytes) # Python 3 only
```
如果 Python2 中能使用 `bytes` 作为 `buffer`的别称那就`bytes` 作为最好了。但是那是不可能的，因为 [Python2.6 中增加了`bytes` 作为 `str` 的同义词](https://docs.python.org/2/whatsnew/2.6.html#pep-3112-byte-literals)。

为了定义两个类型之间的关系，你也可以使用另外一个实体类型作为属性的类型。

## 属性的选项

基于变量的位置或关键词，你可以为属性定义额外的选项。

### 字符串最大长度
`String` 类型接收一个基于变量位置的参数作为字段最大长度的限定。
```python
name = Required(str, 40)   #  VARCHAR(40)
```
### 整数最大值
对于 `int` 类型，你可以使用`size`关键词指定数据库中使用的整数类型的大小。这个参数接收一个数作为数据库中整数占用的位数。允许的值是 8,16,24,32和64。
```python
attr1 = Required(int, size=8)   # 8 bit - TINYINT in MySQL
attr2 = Required(int, size=16)  # 16 bit - SMALLINT in MySQL
attr3 = Required(int, size=24)  # 24 bit - MEDIUMINT in MySQL
attr4 = Required(int, size=32)  # 32 bit - INTEGER in MySQL
attr5 = Required(int, size=64)  # 64 bit - BIGINT in MySQL
```
你可以使用 `unsigned` 参数指定这个属性是无符号的。
```python
attr1 = Required(int, size=8, unsigned=True) # TINYINT UNSIGNED in MySQL
```

`unsigned` 参数默认是 `False` 如果 `unsigned` 是 `True`，但是没有指定 `size` 参数，`size` 将默认为 32位。

如果当前数据库不支持指定的属性大小，则使用下一个更大的属性。例如，PostgreSQL 没有 `MEDIUMINT` 类型，如果指定 24位，那么使用 `INTEGER` 类型。

只有 MySQL 完全支持无符号类型。对于其他数据库将使用能够包含指定的无符号类型的所有值的有符号类型的类型。例如，在 PostgreSQL 中，16位无符号的属性，将会使用 `INTEGER` 类型。 64位的无符号数只能在 MySQL 和 Oracle 中实现。

当数据的大小指定了， Pony 会自动分配这个属性的 `min` 和 `max` 值。例如，一个有符号的 8位属性，可以接收 `min` 为 -128 和 `max` 为 127，8位无符号属性，可以接收 `min` 为 0，`max` 为 255。如果有需要你可以覆盖默认的 `min` 和 `max` 的值，但是不能超过位数指定的范围。

>注
从 Pony Release 0.6 开始，不建议使用 `long` 类型，如果你要存储 64位的整数，需要使用 `int` 类型和 `size=64` 作为替代。如果没有指定 `size` 参数，Pony 会使用数据库的默认整型。

### 小数的精度和进制
对于 `Decimal` 类型， 你可以指定精度和进制。
```python
price = Required(Decimal, 10, 2)   #  DECIMAL(10, 2)
```

### 日期和时间的精度

`datetime` 和 `time` 类型接收基于变量位置的精度参数。对于大多数数据库为 6。

对于 MySQL 数据库，默认值是 0。MySQL 5.6.4 之前，`DATETIME` 和 `TIME` 字段，[无法存储分秒](http://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)。MySQL 5.6.4 之后，如果你指定了精度参数为 6，就可以使用分秒。
`dt = Required(datetime, 6)`

### 其他属性选项

其他能被设为属性选项的参数：

- unique 
布尔值，决定属性是否唯一

- auto
布尔值，只能主键属性使用。如果 `auto=True`，那么属性的值将会自增，基于数据库提供的计数器或者序列。

- default
允许你给属性指定默认值

- sql_default
这个属性允许你指定默认的 SQL 语句，这将在 `CREATE TABLE SQL` 命令执行时起作用。例如：
`created_at = Required(datetime, sql_default='CURRENT_TIMESTAMP')`
当你有一个 `Required` 属性，而且它的值能在 `INSERT` 命令执行时计算出来，那么使用 `sql_default=True` 会很方便（例如，触发器）。默认是 `None`。

- index
允许你控制字段的索引生成。`index=True` —— 将会使用默认的名字创建索引。 `index='index_name'` —— 使用指定的名字创建索引。`index=False` —— 跳过索引的生成。如果没有指定 `index`选项，Pony 依然会使用默认名字创建外键的索引。

- sql_type
如果你想对字段创建指定的 SQL 类型，使用这个选项。

- lazy
布尔值，载入对象时延缓载入属性。设置为 `True`，意味着当你直接调用这个属性之前，这个属性是不会载入的。`LongStr` 和 `LongUnicode` 的 `lazy` 默认设置为 `True`，其他类型默认设置为 `False`。

- cascade_delete
布尔值，控制是否删除关联的对象。`True` 意为 Pony 总是级联的删除，即使关系的另外一边定义是 `Optional`， `False`意为 Pony 总是不做级联的删除。如果关系的另外一边定义是 `Required`，而且 `cascade_delete=False`，那么尝试删除时会抛出一个异常 `ConstraintError`。

- column
指定数据库表中对应字段的名字。

- columns
制定一个列表，用作数据库表中对应字段组合的名字

- reverse
指定关系中另外一边，对应的属性。

- reverse_column
用于多对多关系，指定中间表中对应字段的名字。

- reverse_columns
用于多对多关系，如果实体的主键是组合键时，指定中间表中对应字段的名字。

- table
多对多关系中指定中间表的名字。

- nullable
布尔值，`True` 意为数据库中该字段可以设置为 `NULL`，大多数时候你不需要指定这个参数，因为 Pony 会设置核实的值。

- volatile
通常是你指定了 Python 属性的值，Pony 存储这个值到数据库。但是有时候在数据库中有一些逻辑，可能修改一个字段的值。例如，你可以在数据库中有一个触发器，会修改最后访问的时间戳。这种情况下你可能需要让 Pony 忘掉对象的属性更新时发送到数据库中的值，在下次访问时使用数据库中的值。设置 `volatile=True` 可以让 Pony 知道这个属性的值可能在数据库中被修改。
`volatile=True` 可以和 `sql_default=True` 一起使用，如果这个属性完全是由数据库创建和更新的。
如果当前事务正在使用的值被别的事务修改了，你会得到一个异常：`UnrepeatableReadError: Value ... was updated outside of current transaction`。Pony 之所以要明确提示是因为这种情况可能会破坏应用的业务逻辑，如果你不需要这种竞争性保护，你可以设置 `volatile=True`。

- sequence_name
对于 Oracle 数据库，指定 `PrimaryKey` 属性使用的序列的名字。

- py_check
这个属性允许你制定一个用来检查赋给属性的值的函数。这个函数必须返回 `True` 或者`False`。也可以在检查出错时抛出 `ValueError` 异常。

- min
指定数字型属性（int, float, Decimal）的最小值，如果你赋值的值小于这个值，会抛出 `ValueError` 异常。

- max
指定数字型属性（int, float, Decimal）的最大值，如果你赋值的值大于这个值，会抛出 `ValueError` 异常。

## 实体类型的继承
Pony 中的实体对象的继承和 Python 中规则类似。让我们考虑一个例子，实体对象 `Student` 和 `Professor` 继承自 `Person` ：

```python
class Person(db.Entity):
    name = Required(str)

class Student(Person):
    gpa = Optional(Decimal)
    mentor = Optional("Professor")

class Professor(Person):
    degree = Required(str)
    students = Set("Student")
```

子类继承了基类 `Person` 的所有属性和关系。有些映射器（例如，Django）存在一个问题，当使用基类进行查询时没有返回正确的类型：对于指定的实例，查询结果只包含实例的继承来的部分。Pony 没有这个问题，你总是可以得到正确的实体对象的实例：
```python
for p in Person.select():
    if isinstance(p, Professor):
        print p.name, p.degree
    elif isinstance(p, Student):
        print p.name, p.gpa
    else:  # somebody else
        print p.name
```
为了得到正确的实体对象实例，Pony 使用了额外的区分字段。默认情况下这是一个字符串字段，Pony 使用它来存储实体对象类型的名字。
`classtype = Discriminator(str)`
默认情况下，对于继承的实体类型，Pony 隐式的的创建了 `classtype` 属性。你可以自定义区分字段的名字和类型。如果你改变了区分字段的类型，那就必须指定每一个实体对象的 `_discrimintator_`。让我们看下以下的例子，使用 `cls_id` 作为区分字段，使用 `int` 类型。
```python
class Person(db.Entity):
    cls_id = Discriminator(int)
    _discriminator_ = 1
    ...

class Student(Person):
    _discriminator_ = 2
    ...

class Professor(Person):
    _discriminator_ = 3
    ...
```

## 多重继承
Pony 也支持多重继承。如果要使用多重继承，那么当前类的所有父类都必须继承自同一个基类。（菱形继承）。
让我们看一下这个学生可以当助教的例子。我们引入 `Teacher` 类型，并从它衍生出 `Professor` 和 `TeachingAssistant`。其中， `TeachingAssistant` 继承自 `Student` 和 `Teacher` 两个基类。

```python
class Person(db.Entity):
    name = Required(str)

class Student(Person):
    ...

class Teacher(Person):
    ...

class Professor(Teacher):
    ...

class TeachingAssistant(Student, Teacher):
    ...
```
`TeachingAssistant` 的对象，既是 `Student` 也是 `Teacher` 的实例，继承了它们所有的属性。多重继承在这里是可行的是因为 `Student` 和 `Teacher` 有相同的基类 `Person`。
继承是非常强大的，但是要明智的使用它，对于非常简单的数据表，继承的用处不大。

## 继承在数据库中的应用
在数据库中有三种方法实现继承：
1. 单一表继承法：继承树中的的所有子类都映射到一个表。
2. 分类表继承法：继承树中的的每层子类都应到到一个单独的表，但是每个表只存储不是从父类继承的部分。
3. 完整表继承法：继承中的的每层子类都应到到一个单独的表，每个表都存储了存储实体对象的属性和它的所有的父类。

第三种途径的问题是没有单独的一个表可以存储主键，那就是为什么这种方法很少使用。

第二种方法是最常用的，这就 Django 中继承实现所使用的。这种方法的缺点是取数据的时候需要并表，可能导致性能下降。

Pony 使用第一种方式，所有继承树中的实体对象到映射到一张单独的表。这是最有效率的方法，因为不需要并表查询。这种方法也有它的缺点：

- 数据表中的每一条数据可能都有一些字段不能用，因为他们属于其他的实体类型。这不是一个大问题，因为空的字段都会被置位 `NULL`，不占太大空间。
- 如果继树中有很多的实体对象，数据表可能会有非常多的字段。不同的数据库对最大字段数的限制是不一样的，但是一本都非常大。

第二种方法还有一个好处：当需要增加一个新的实体对象的时候，基类的表是不需要变动的。Pony 以后将会这种方式。

## 自定义映射
当 Pony 从实体对象创建数据表的时候，默认使用实体对象的名字作为表名，使用属性的名字作为字段名，但是你可以重写这个行为。

数据表名字也不总是等于实体对象的名字： 在 MySQL 和 PostgreSQL，数据表名来自实体对象的名字转换为小写，在 Oracle 中转为大写。总是可以通过读取 `_table_` 属性来知道对应的数据表。

如果你需要设置你自己的数据库表名，使用 `_table_`属性。
```python
class Person(db.Entity):
    _table_ = "person_table"
    name = Required(str)
```
如果你需要设置你自己的字段名，使用 `column` 选项。
```python
class Person(db.Entity):
    _table_ = "person_table"
    name = Required(str, column="person_name")
```
对于组合属性，使用 `columns` 和名字的列表：
```python
class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    lectures = Set("Lecture")
    PrimaryKey(name, semester)

class Lecture(db.Entity):
    date = Required(datetime)
    course = Required(Course, columns=["name_of_course", "semester"])
```
在这个例子中，我们重写了组合属性  `Lecture.course` 的字段名。默认情况 Pony 将使用 `"course_name"` 和 `"course_semester"`， 由实体对象名字和属性名字连接而成方便程序员识别。

如果你想要自定义多对多关系中的中间表的名字，你需要使用对 `Set` 属性使用 `column` 或 `columns` 选项，让我们看下面的例子：

```python
from pony.orm import *

db = Database("sqlite", ":memory:")

class Student(db.Entity):
    name = Required(str)
    courses = Set("Course")

class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    students = Set(Student)
    PrimaryKey(name, semester)

sql_debug(True)
db.generate_mapping(create_tables=True)
```
默认情况下，为了存储 `Student` 和 `Course` 的多对多关系, Pony 会创建 `"Course_Student"` 中间表（中间表的名字是实体对象的名字按照首字母字母表的顺序相连）。这个表将会有三个字段 `"course_name"`, `"course_semester"` 和 `"student"` ，`Course` 的组合主键占用两个，和 `Student` 的占用一个字段。如果我们要 `"Study_Plans"` 包含字段： `"course"`, `"semester"` 和 `"student_id"`，以下代码片段实现了这个目的：

```python
class Student(db.Entity):
    name = Required(str)
    courses = Set("Course", table="Study_Plans", columns=["course", "semester"]))

class Course(db.Entity):
    name = Required(str)
    semester = Required(int)
    students = Set(Student, column="student_id")
    PrimaryKey(name, semester)
```
更多关于自定义映射的内容可以查看 [PonyORM 包中带的例子](https://github.com/ponyorm/pony/blob/orm/pony/orm/examples/university.py)。 