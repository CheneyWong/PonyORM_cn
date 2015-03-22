# Datebase

在使用实体对象之前，你必须先创建一个 `Datebase` 对象。这个对象通过连接池管理着数据库的连接。`Datebase` 对象是线程安全的，可以在你的应用的所有线程之间共享。通过`Datebase` 对象允许你直接是使用 SQL 操作数据库，但是大多数时候都是通过操作实体对象让 Pony 生成命令去修改数据库的对应部分。Pony 允许你同时操作多个数据库，但是一个实体对象只能对应一个数据库。

将实体对象映射到数据库可以分为四个步骤：

- 创建一个数据库对象
- 创建跟这个数据库关联的实体对象
- 将数据库对象绑定到指定的数据库
- 映射实体对象到数据库的表

## 创建数据库对象
这一步我用最简单的方法创建一个 `Database`类的实例：
`db = Database()`
虽然在这里你就可以传递数据库连接参数了，但是在下个阶段使用 `db.bind()` 方法通常是更方便的。这样你可以对测试环境和生产环境切换不同的数据库。
`Database` 实例有一个 `Entity` 属性作为实体类型声明时的基类使用。

## 定义与数据库对象关联的实体类型
实体对象需要从 `Database` 对象的基类继承。
```python
class MyEntity(db.Entity):
    attr1 = Required(str)
```
下一张我们将详细讨论实体对象定义的细节。现在我们先看下一步，映射实体对象到数据库。

## 映射实体对象到数据库
在我们映射实体对象到数据库时，我们需要先连接到数据库。这一步，我们需要使用 `bind()` 方法：
```python
db.bind('postgres', user='', password='', host='', database='')
```
第一个参数用来指定提供的数据库。数据库提供了一个模块，封装在 `pony.orm.dbproviders` 包中，这个模块实现了对具体数据库操作的能力。接下来的一个参数需要指定传递给一致性 DBAPI 中的 `connect()` 方法的参数。

## 数据库支持模块
目前，Pony 支持的操作系统有：SQLite，PostgreSQL，MySQL和 Oracle， Pony 提供相应的名字：'SQLite'，'PostgreSQL'，'MySQL'和 'Oracle'。可以很容易的扩展数据库支持模块。

在 `bind()` 执行期间，Pony 尝试建立一个测试连接到数据库。如果指定的参数不正确或者数据库不可用，会抛出一个异常。当数据库连接建立成功后，Pony 读取数据库的版本，然后归还书库据连接到连接池。

### SQLite

使用 SQLite 数据库是使用 Pony 最简单的方式，因为没有必要去额外的建立一个数据库系统 ——
 SQLite 数据库系统已经包含在 Pyhton 的包中。初学者在交互环境中体验 Pony 是最好的选择。为了绑定一个数据库对象到 SQLite 数据库，你可以使用如下命令：
```python
db.bind('sqlite', 'filename', create_db=True)
```
这里 'filename' 是需要存储数据的 SQLite 数据库的文件。文件名字可以是绝对路径也可以是相对路径。

>注
如果你指定了一个相对目录，这个目录将接在 Python 文件所在的文件夹下面（而不是当前工作目录的下面）。我们这么做是因为有些时候程序员没有操作当前工作目录的权限（例如在 mod_wsgi 应用中）。这个方法也可以让程序员创建由不同独立模块组合起来的应用，每个模块都可以有独立的数据库。

当在交互器中工作时， Pony 建议总是使用绝对路径指定存储文件。

如果参数 `create_db` 是 `True`，Pony 在指定文件不存在的时候将会尝试创建文件。默认的 `create_db` 值是 `False`。

>注
一般情况下， SQLite 数据库存储在硬盘的文件上，但是也可以完全被存储在内存中。如果是在交互器中玩玩 Pony，这种方式是非常方便的，但是你必须注意，所有的内存中的数据库将在程序退出的时候丢失。而且内存中的数据库在多线程的程序中也表现的不同，因为 SQLite 限制了所有的线程共享相同的连接来操作数据库。

>通过用 `:memory:` 替代文件名，可以在内纯中创建数据库。
`db.bind('sqlite', ':memory:')`
 如果是在内存中创建数据库，`create_db`也不要设置了。

 
>注
SQLite 默认不检查外键限制。自 release 0.4.9 起，可以通过发送命令 `PRAGMA foreign_keys = ON;` 来让 Pony 十种开启外键检测。

### PostgreSQL
Pony 使用 psycopg2 驱动 PostgreSQL 。绑定到 PostgreSQL 数据库对象可以使用如下方法：
`db.bind('postgres', user='', password='', host='', database='')`
接下来的所有参数将会传递给 `psycopg2.connect()` 方法。通过 [psycopg2.connect documentation](http://initd.org/psycopg/docs/module.html#psycopg2.connect) 可以查到更多的可以传递的参数。

### MySQL
`db.bind('mysql', host='', user='', passwd='', db='')`
Pony 尝试使用 MySQLdb 驱动 MySQL，如果这个模块不能导入，Pony 会尝试使用 pymysql。耕作内容参见：[MySQLdb](http://mysql-python.sourceforge.net/MySQLdb.html#functions-and-attributes) and [pymysql](https://pypi.python.org/pypi/PyMySQL)。

### Oracle
`db.bind('oracle', 'user/password@dsn')`
Pony 使用 cx_Oracle 驱动来连接 Oracle 数据库。更多关于可以传递到数据库连接的函数的参数请参考[这里](http://cx-oracle.sourceforge.net/html/module.html)。

## 映射实体对象到数据库表
`Database`对象创建了，实体对象定义了，数据库也连接了，下一步就是映射实体对象到数据库表。
`db.generate_mapping(check_tables=True, create_tables=False)`
如果 `create_tables` 被设置为 `True`，那么 Pony 会尝试创建不存在的表。`create_tables` 的默认值是 `False` 因为大多数情况下表都是存在的。Pony 会自动生成数据库表和列的名字，但是如果你愿意，可以重写这个行为。具体细节参考 自定义映射 章节。这个参数也让 Pony 检查外键和索引是否存在，如果不存在则创建它们。

如果传递了 `create_tables` 参数，Pony 做了一个简单的检查：发送一个检测实体对象的表和列名字均存在的 SQL 命令。但是，不检测表是否有额外的列，或者列的类型和实体类型定义的不匹配。通过传递 `check_tables=True` 参数可以关闭检测过程。当你想使用 `db.create_tables()` 方法在稍后的时候再生成映射和表的时候，这个设置是有用的。

##　更早的数据库绑定
通过在创建数据库对象的时候传递数据可参数，可以合并“创建数据库对象”和“绑定数据库对象到指定的数据库”这两步为一步。
```python
db = Database('sqlite', 'filename', create_db=True)

db = Database('postgres', user='', password='', host='', database='')

db = Database('mysql', host='', user='', passwd='', db='')

db = Database('oracle', 'user/password@dsn')
```
参数的设置都跟传递到 `bind()` 方法的相同。如果在数据库对象创建啊时传递了这些参数，那么之后就不用再调用 `bind()` 方法了 —— 数据库对象和数据库已经绑定了。

##　数据库对象的方法和属性
class Datebase

- generate_mapping(check_tables=True, create_tables=False)
映射声明的属性到数据库中相应的表。`create_tables=True` - 创建不存在的数据表，外键和索引。`check_tables=False` 关闭表校验。只检测数据库表名称和属性名称与实体对象的声明匹配，不检测数据表拥有额外的列和列的属性与声明不匹配的情况。

- create_tables()
检查实体的对象的映射和创建不存在的表，Pony 同时也检查外键和索引是否存在。

- drop_all_tables(with_all_data=False)
删除所有存在映射关系的中表。当这个方法以无参数形式调用的时候，Pony 只会在所有的表中都没有数据的时候删除这些表。为了防止误删，只要任何数据表中有数据，那么这个方法会抛出一个 `TableIsNotEmpty` 异常，为了删除带数据的表，需要传递参数 `with_all_data=False`。

- drop_table(table_name, if_exists=False, with_all_data=False)
删除 `table_name` 指定的表，如果表不存在，抛出 `TableDoesNotExist` 异常。注意，table_name 是大小写敏感的。
可以传递实体对象的类名作为 `table_name`，这种情况下，Pony 将会尝试删除和实体对象关联的表。
如果参数 `if_exists` 被设置为 `True`，即使表不存在也不会抛出 `TableDoesNotExist` 异常。当表非空的时候会抛出 `TableIsNotEmpty` 异常。
使用参数 `with_all_data=True` 可以删除包含数据的表。如果需要删除实体对象对应的表，可以调用实体对象的 `drop_tables()` 方法，它总会删除正确对应的大小写拼写的名字的表。

### 传输相关的方法
class Database

- commit()
通过当前 `db_session` 的 `flush()` 方法，保存所有的变化并提交事务到数据库。
程序员可以在一个 `db_session` 中多次调用 `commit()`。这种情况下，`db_session` 缓存了上次提交之后的对对象的所有操作。这允许 Pony 使用相同的一个对象实现链式的事务调用。当 `db_session` 调用结束或者或者事务回滚的时候，缓存会被清空。

- rollback()
回滚事务和清空 `db_session`中的缓存。

- flush()
更新 `db_session` 累积 的缓存到数据库。你可能永远不需要手动调用这个方法。Pony 在执行：select()，get()， exists()， execute() 和 commit()
等方法之前会自动的保存累积的。

### Database 对象属性
- Enity
这个属性提供了一个应该被所有需要映射到制定数据库的实体对象的类型继承的基类。
```python
db = Database()

class Person(db.Entity):
    name = Required(str)
    age = Required(int)
```
- last_sql
这是一个保留最后一条 SQL 命令的只读属性。它可以被用来调试程序。

### 使用原始数据的方法
class Database

- select(sql)
在数据库中执行 SQL 语句，返回一个元组的列表。Pony 会从连接池中取出一个数据库连接，执行完查询语句后再将连接送回到数据库中。SQL 语句中的 `select` 单词是可以省略的 —— Pony 会在必要的时候帮你加上。
`select("* from Person")`
我们这么做因为这样方法的名字已经说明了语句的用途，这样查询语句看起来会比较简明。如果一个查询语句的返回结果是只有一列，那么，为了方便，返回结果将会一个列表，没有包含元组。
`db.select("name from Person")`
以上语句返回: [“John”, “Mary”, “Bob”]
如果一个查询语句返回了多列，而且数据表的列的名字可以作为 Python identifiers，这时也可以通过类似调用属性的方法调用。
```python
for row in db.select("name, age from Person"):
    print row.name, row.age
```
Pony 有一个对 `select` 方法能够返回列的数量的限制。这个限制是通过 `pony.options.MAX_FETCH_COUNT` 参数指定的（默认是1000）。如果一个 `select` 返回超过 `MAX_FETCH_COUNT` Pony 抛出一个 `TooManyRowsFound`异常。你可以改变这个值，但是我们不建议这么做，因为如果一个查询语句返回超过 1000 列，那么很可能是这个应用有设计上的问题。`select` 的返回结果是存储在内存中的，如果列的数量是非常大的，应用可能面临着性能问题。

- get(sql)
如果你只需要从数据库中拿出一行数据或者一个值，可以使用 `get` 方法：
`age = db.get("age from Person where id = $id")`
SQL 语句中的 `select` 单词可以被省略 —— Pony 将会在必要的时候自动添加 select 关键词。如果 Person 数据表中存在指定的 id，age 变量将会赋予 int 类型的对应的值。`Get`方法假定返回一行的值，如果查询结果为空，会抛出 `RowNotFound` 异常，如果查询结果不只一行，会抛出 `MultipleRowsFound` 异常。

如果你需要选择多个列，可以使用这个方法：
`name, age = db.get("name, age from Person where id = $id")`

如果你的的查询返回许多列，你可以将结果存入到变量中，还是可以使用 `select`方法中描述的相同的方法来操作。

在执行 `get` 方法前，Pony 使用 `flush()` 方法刷新所有的缓存中的变化。

- exists(sql)
`exists` 方法是用来检查指定参数在数据库中是否存在至少一个数据，返回结果可能是 `True` 或者 `False`。
```python
if db.exists("* from Person where name = $name"):
    print "Person exists in the database"
```
SQL 命令的的 select 单词可以省略。
在执行此方法前，Pony 使用 `flush()` 方法刷新所有的缓存中的变化。

- insert(table_name, returning=None, **kwargs)
数据库中插入新的数据行。这个命令旁路身份映射缓存[^identity-map-cache]，可以用来在创建很多对象但是不需要同时操作它们的时候提升性能。你也可以使用 `db.execute()`达到相同的目的。如果我们需要在创建的同时右又要操作它们，那么创建实体对象的实例，然后通过 Pony 用 `commit()` 命令保存它们将是更好的方法。

`table_name` —— 是要插入数据的数据表的名字，大小写敏感的。

`returning` 参数让你可以指定自动生成的主键列。例如你想要 `insert` 方法返回数据库自动生成的数据，你应该指定主键列的名字。
`new_id = db.insert("Person", name="Ben", age=33, returning='id')`

- execute(sql)
这个方法让你可以执行任意的（原生的）SQL语句。
```python
cursor = db.execute("""create table Person (
                           id integer primary key autoincrement,
                           name text,
                           age integer
                    )""")
name, age = "Ben", 33
cursor = db.execute("insert into Person (name, age) values ($name, $age)")
```
基于不同的 DBAPI，Pony 使用 `$` 符号统一的传递所有的参数。在上例中，我们就传递了 `name` 和 `age` 两个参数到命令中。
命令中使用 Python 表达式也是可以的，例如：
```python
x = 10
a = 20
b = 30
db.execute("SELECT * FROM Table1 WHERE column1 = $x and column2 = $(a + b)")
```
如果需要让 `$` 作为字符串原始的意思，你需要另外一个 `$` 转义它（用两个连续的`$`：`$$`）。

- get_connection()
获取一个可用的数据库连接。如果想直接调用 DBAPI 接口，这个函数是很有用的。ORM 自己也是用的这样的方式。在 离开`db_session`上下文或者数据库事务回滚的时候，连接将被重置并归还到连接池。

- disconnect()
关闭当前线程已开始的数据库连接。

### Database 状态统计
class Database
`Database` 对象保留了命令执行的统计结果。你可以查看哪个命令执行的最多花费了多少时间，以及其他很多参数。Pony 对每个线程保留分离的统计结果。如果你想看所有线程的总计结果，需要调用 `merge_local_stats()` 方法。

- local_stats
这是一个当前线程的 SQL 语句的字典。字典的键是 SQL 语句，值是 `QueryStat` 类型的一个对象。

- class QueryStat
这个类型有一系列的属性，用来累积对应则值：
`avg_time`, `cache_count`, `db_count`, `max_time`, `merge`, `min_time`, `query_executed`, `sql`, `sum_time`.

- merge_local_stats()
这个方法合并当前线程的统计结果到全局的统计结果。你可以在 HTTP 请求结束的时候调用这个方法。

- global_stats
这是一个汇总了所有线程的 SQL 语句的字典。字典的键是 SQL 语句，值是 `QueryStat` 类型的一个对象。在访问这个对象之前，你应该调用 `db.global_stats_lock`。

## 使用 Database 对象做原生 SQL 命令
使用 Pony 可以使用方便的给 SQL 命令传递参数。为了传递变量，你需要将 $ 放置在变量名之前。
```python
x = "John"
data = db.select("* from Person where name = $x")
```
当 Pony 在 SQL 命令中遇到这种变量，它会从当前上下文获取变量的值（从 globals 和 locals）,或者从第二个变量传递的一个字典。上面的例子中，Pony 尝试从变量 x 中获取值给 $x，同时排除 SQL 注入风险。下面的例子展示了如何传递一个包含参数值的字典。
```python
data = db.select("* from Person where name = $x", {"x" : "Susan"})
```
这种传递参数到 SQL 命令的方式非常灵活，而且不只是传递一个值，还可以支持 Python 表达式。为了使用表达式，需要将表达式跟在 $ 符号后面。

```python
data = db.select("* from Person where name = $(x.lower()) and age > $(y + 2)")
```

[^identity-map-cache]: identity map cache


