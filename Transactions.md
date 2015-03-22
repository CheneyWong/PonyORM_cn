# 事务

事务是一组逻辑上的工作单元，可以由一个或多个语句组成。事务是原子的，那就是说，当通过事务改变数据库的时候，要么事务成功所有的改变都成功，要么事务回滚，所有的改变都没做。

Pony 使用数据库 session 来实现事务关系。

## 通过数据库 session 来工作
和数据库交互的代码都必须工作在数据库 session 之下。session 定义了数据库会话的边界。由数据库操作的每一个线程会建立一个单独的数据库 session ，使用一个单独的 Identity Map。Identity Map 像个缓存一样工作，当你通过主键或者唯一性键查询对象的时候，如果身份映射表已经有了，就避免了再次查询数据库。为了使用数据库 session 来操作数据库，你可以使用装饰器 `@db_session` 或者 `db_session` 上下文管理器。当一个 session 结束的时候，它完成了以下动作。

- 如果数据被改变了，而且没有异常，那么提交事务。如果有异常发生，回滚事务。
- 将数据库连接归还到数据库连接池。
- 清空 Identity Map 缓存。

如果你在必要的地方没有指定 `db_session`，Pony 会抛出异常 `TransactionError: db_session is required when working with the database`。

使用 `@db_session` 装饰器的例子:
```python
@db_session
def check_user(username):
    return User.exists(username=username)
```

使用 `db_session` 上下文管理器的例子:
```python
def process_request():
    ...
    with db_session:
        u = User.get(username=username)
        ...
```

>注
当你使用 Python 交互器来操作时，你不用考虑数据库会话的问题，因为 Pony 已经自动替你维护了。

当你尝试访问 `db_session` 区域外的实例的属性（非数据库自动创建的），你会得到一个异常 `DatabaseSessionIsOver`，例如：

```python
DatabaseSessionIsOver: Cannot load attribute Customer[3].name: the database session is over
```

之所以有这种事情是因为，数据库连接已经返还到连接池，事务已经关闭，已经不能传送任何命令到数据库。

当 Pony 从数据库中读取对象的时候，它将对象放到 Identity Map。之后，当你更新对象的属性，创建珊处对象时，改变会首先增量的保存到 Identity Map。改变会在事务提交的时候，或者在调用 `select()`、 `get()`、 `exists()`、 `execute()` 之前，保存到数据库。

### 事物的范围
一般情况下，使用 `db_session`，你就有了一个事务。没用明确的开启事务的命令。事务将在第一条 SQL 语句发送到数据库的时候执行。在发送第一条语句之前， Pony 会从连接池中获取一个数据库连接。之后的 SQL 语句都会使用相同的上下文执行。

>注
SQLite 的 Python 驱动不在 SELECT 语句上开启事务。只有在能改变数据库状态的语句上才会开启事务，有：INSERT, UPDATE, DELETE。其他驱动在任何命令上都会开启事务，包括 SELECT 。

事务在使用 `commit()` 提交的时候结束，或者在 `rollback()` 调用的时候回滚，或者离开 `db_session` 区域。

```python
@db_session
def func():
    # a new transaction is started
    p = Product[123]
    p.price += 10
    # commit() will be done automatically
    # database session cache will be cleared automatically
    # database connection will be returned to the pool
```

Several transactions within the same db_session
### 一个 db_session 下的多个事务 

如果你需要在一个事务中包含多个数据库 session，你可以 session 范围下任何时候调用 `commit()` 或 `rollback()`，同时下一条语句会开启一个新的事务，手动调用 `commit()` 之后, Identity Map 保留了数据缓存，你可以使用 `rollback()` 清除缓存。

```python
@db_session
def func1():
    p1 = Product[123]
    p1.price += 10
    commit()          # the first transaction is committed
    p2 = Product[456] # a new transaction is started
    p2.price -= 10
```

### db_session 的嵌套

如果你递归的进入 `db_session` 的区域，`@db_session`修饰的函数调用了另一个 `@db_session`修饰的函数，Pony 将不会创建新的 session，而是将原来的 session 共享给两个函数。数据库 session 将在离开最外层的 `db_session` 修饰器或这上下文管理器时关闭。

### db_session 的缓存

为了提高性能，Pony 在如下几种情况缓存数据：

- 生成器表达式的执行结果。如果一条生成器表达式被调用了多次，只会向数据库发送一次数据。这个缓存是实体对象程序全局的，不属于单个的 session 。
- 数据库创建或载入的对象。Pony 保留这些对象到 Identity Map ，离开 `db_session` 区域或者事务回滚的时候清除。
- 执行结果。相同参数的相同语句会直接从缓存中返回结果。实例改变一次缓存清空一次。离开 `db_session` 区域或者事务回滚的时候缓存清除。

### 多数据库

Pony 可以同时使用多个数据库。下面的例子中我们使用 PostgreSQL 存储用户信息，用 MySQL 存储地址信息。
```python
db1 = Database("postgres", ...)

class User(db1.Entity):
    ...

db2 = Database("mysql", ...)

class Address(db2.Entity):
    ...

@db_session
def do_something(user_id, address_id):
    u = User[user_id]
    a = Address[address_id]
    ...
```
当退出 `do_something()` 函数时， Pony 对所有的数据库执行 `commit()` 或 `rollback()`，如果有必要。

## 事务的相关函数

- commit()
使用 flush() 函数存储当前 `db_session`下的所有修改，提交事务到数据库。顶层的 `commit()` 会调用 当前事务所用到的数据库对象的 commit() 方法。

- rollback()
回滚当前事务。顶层的 `rollback()` 会调用 当前事务所用到的数据库对象的 rollback() 方法。

- flush()
将 `db_session` 缓存中的变动保存到数据库，不包含提交数据变动。大多数情况，Pony 自动从数据库会话缓存中将数据保存到数据库，不需要你自己调用这个函数。有一种情况你需要调用它，当你想在 commit 之前获取一个新对象自动获取的主键的时候。

Pony 总是在执行
`select()`、 `get()`、 `exists()`、 `execute()` 和 `commit()` 前自动增量式保存变化到 `db_session` 缓存。
`flush()` 函数让 `db_session` 缓存中的更新在当前事务下的数据库访问中生效。同时，`flush()`并没有真正将数据存入数据库。
顶层的 `flush()` 会调用 当前事务所用到的数据库对象的 flush() 方法。

## db_session 的参数
之前已经提到 `db_session` 可以作为修饰器或者上下文管理器。`db_session` 可以接收以下参数。

- retry
接收一个整数值，指定尝试提交这个事务的次数。这个参数只能在修饰器形式下使用。被修饰的函数不能直接调用 `commit()` 和 `rollback()`。当指定了这个参数，Pony 将缓存 `TransactionError` 以及它的派生类的异常，然后重置事务。默认情况下 Pony 只缓存 `TransactionError` 异常，但是这个列表可以被 `retry_exceptions`参数重写。

- retry_exceptions
接收一个列表，指定这些异常将会导致事务重启。默认情况下，这个参数值是 `[TransactionError]`。另外的选择是指定一个回调函数，这个回调函数接收一个参数 —— 当前发生的异常。如果这个函数返回 `True`，事务将会重启。

- allowed_exceptions
这个参数接收一个异常列表，当这些异常发生时，失误不会回滚。例如，一些 HTTP 框架通过异常来触发跳转。

- immediate
接受一个布尔值，默认值是 `False`。一些数据库（例如，SQLite，Postgres）只有当提交更改数据的语句（UPDATE，INSERT，DELETE）时开启一个事务，SELECT 命令则不会。如果你想在 SELECT 时也开启事务，可以通过传递 `True` 到这个参数。通常情况下没必要改变这个值。

- serializable
接受一个布尔值，默认值是 `False`。允许你设置 SERIALIZABLE 串行化的等级。

## 乐观的并发控制
为了提升性能，Pony 默认使用乐观的并发控制。基于这个观点，Pony 并不获取数据的锁。而是确认并没有别的会话尝试读取或修改相同的数据。如果检测到冲突的修改，事务提交时抛出异常 `OptimisticCheckError, 'Object XYZ was updated outside of current transaction'`，然后回滚。

面对这种情况我们应该怎么做？首先，这种行为在使用 [MVCC](http://en.wikipedia.org/wiki/Multiversion_concurrency_control) 模式的数据库（例如，Postgre，Oracle）是很常见的。例如，在 Postgres 中，在不同事务同时修改相同的数据时，你会得到如下错误：
`ERROR: could not serialize access due to concurrent update`
当前事务被回滚，但是它可以被重新开始。为了自动重试事务，你可以在 `db_session` 修饰器使用 `retry` 参数（在本章后面查看更多细节）。

Pony 怎么做这种乐观的检查？Pony 跟踪每个对象的属性的访问。当用户的代码读或修改一个对象的属性时，Pony 检查这个属性的值是否有待提交到数据库的残余内容。这种方式可以避免丢失数据更新，有一种情况，当当前事务和并发的别的事务修改相同的对象时，当前事务覆盖了数据了，掩盖了之前的修改。

在乐观的检查中，Pony 只检查用户读或写的属性。在 Pony 更新对象的时候，也是只更新用户修改了的属性。在这种运行方式下，两个不同的事务更新对象的不同属性，然后都成功了，这是可能的。

通常乐观的并发控制可以提升性能，因为事务执行可以避免请求锁和等待别的事务释放锁。这种方法在冲突很少和读笔写多得多的情况下表现良好。

## 悲观的锁

有些时候我们需要锁定数据库中的对象，来避免其他事务修改相同的记录。在数据库中可以使用 `SELECT FOR UPDATE` 语句。在 Pony 中要生成这种可以使用 `for_update` 方法。
 `select(p for p in Product if p.price > 100).for_update()`
上面语句选定的符合价格大于 100 的 Product 的实例都将被锁定。在当前事务提交或回滚之后，锁会被释放。

如果你需要锁定一个对象，你可以使用 `get_for_update` 方法。
`Product.get_for_update(id=123)`
当你尝试用 `for_update` 锁定一个对象的时候，如果它已经被另外的事务锁定了，你的请求将不得不等待行级别的锁被释放。为了避免这种情况，使用 `nowait=True`:
```python
select(p for p in Product if p.price > 100).for_update(nowait=True)

Product.get_for_update(id=123, nowait=True)
```
这种情况下，如果选定的行不能被马上锁定，当前请求将会报告一个错误而不是等待。
消极的锁的主要问题是性能下降，因为数据库锁的消耗和对并发的限制。

## 事务隔离和数据库差别
隔离是一个属性，定义了当一个事务修改了数据之后，多久别的并行事务能够看到修改。ANSI SQL 标准规定了四个等级。

- READ UNCOMMITTED - 最不安全的等级
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE - 最安全的等级

当使用 SERIALIZABLE 等级的时候，每个事务都将数据库看成事务开始时的一个快照。这个等级提供最大程度的数据隔离，但是比其他等级需要等多的资源。

这就是为什么大多数数据库使用更低的隔离等级以允许更好的并发。默认情况下，Oracle 和 PostgreSQL 使用 READ COMMITTED, MySQL - REPEATABLE READ. SQLite 只支持 SERIALIZABLE，但是 Pony 模拟了 READ COMMITTED，支持更好的并发。

如果你希望 Pony 使用 `SERIALIZABLE` 等级的事务，你可以给 `@db_session` 修饰器或 `db_session` 上文管理器指定 `serializable=True` 参数。

## READ COMMITTED vs. SERIALIZABLE 模式
在 `SERIALIZABLE` 模式，你总是需要应对得到 `“Can’t serialize access due to concurrent update”` 错误，而且还要重试事务直到成功。在 SERIALIZABLE 模式事务写数据库，总是需要在你的应用中写重试循环。

在 `READ COMMITTED` 模式，如果你希望在并发的事务中避免新修改相同的数据，你应该使用 `SELECT FOR UPDATE`。但是这种情况下有可能导致数据库[死锁](http://en.wikipedia.org/wiki/Deadlock) —— 这种情况就是一个事务在等待另一个事务锁定的资源。如果你的事务陷入死锁，就需要重启事务。所以你无论如何都需要一个重试循环。Pony 可以自动重试一个事务，如果你指定了 `retry` 参数给 `@db_session` 修饰器（ 不能是 `db_session` 上文管理器）。

```python
@db_session(retry=3)
def your_function():
    ...
```

### PostgreSQL
PostgreSQL 默认使用 READ COMMITTED 隔离等级。PostgreSQL 也提供自动提交模式。在这种模式下，每条 SQL 语句都运行在一个独立的事务中。当你的应用只是读取数据库中的数据，自动提交模式可以更有效，因为没必要开启和关闭事务。数据库直接提供了这个功能。从隔离这个角度来说，自动提交模式和 READ COMMITTED 隔离等级没有什么不同，在两种情况下，你的应用都是马上看到已经提交的数据。

Pony 自动在自动提交模式和明确的开启一个事务之间切换，当你的应用需要使用 INSERT、 UPDATE 或 DELETE SQL 来原子的修改数据。

### SQLite
当使用 SQLite 的时候，Pony 的行为和使用 PostgreSQL 时类似：当事务启动之后，读取数据将会在自动提交模式执行。这种模式的隔离等级相当于 READ COMMITTED。这种方式下，并发的事务可以被同时执行而没有死锁的风险（Pony 不会抛出 `sqlite3.OperationalError: database is locked` 异常）。当你的代码发布了非独占声明，Pony 会开启一个事务，然后之后的 SQL 语句也会使用这个事务执行。事务将会是 SERIALIZABLE 隔离等级。

### MySQL
MySQL 默认使用 REPEATABLE READ 隔离等级。Pony 将不会在 MySQL 使用自动提交模式，因为这无利可图。事务在第一条 SQL 语句发送到数据库的时候开启，即使这条语句是 SELECT 。

### Oracle
Oracle 默认使用 READ COMMITTED 隔离等级。Oracle 没有自动提交模式。事务在第一条 SQL 语句发送到数据库的时候开启，即使这条语句是 SELECT 。

## Pony 怎样避免丢失更新数据
低的隔离等级在许多用户同时访问数据的时候提升了性能，但是这也有可能导致数据库异常，例如丢失更新。

让我们考虑一种情境。我们有两个账户，我们需要提供一个函数将一个账户的钱转到另一个账户。在转账的过程中，我们需要检查账户中的钱是否足够。

假如我们使用 Django ORM 。下面是一个可能的例子实现了这个函数。

```python
@transaction.atomic
def transfer_money(account_id1, account_id2, amount):
    account1 = Account.objects.get(pk=account_id1)
    account2 = Account.objects.get(pk=account_id2)
    if amount > account1.amount:    # validation
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account1.save()
    account2.amount += amount
    account2.save()
```
在 Django 中默认，`save()` 在单独的一个事务中运行。如果在第一个 `save()` 之后有一个错误，这笔钱就消失了。即使没有错误，如果另外一个事务在两个 `save()` 之间访问账户，结果将是错误的。为了避免这个问题，所有的操作都必须在一个事务中完成。我们可以通过用 `@transaction.atomic` 装饰器装饰这个函数。

但是否即使这样，我们还会碰到一个问题。如果两个支行同时向第三个账户汇款，操作都将会被执行。每个函数都传递了值但是后完成的事务将会覆盖先完成的那个。这种异常叫做“丢失更新数据”。

有三种方法来避免这种异常:

- 使用 SERIALIZABLE 隔离等级
- SELECT FOR UPDATE 替代 SELECT
- 使用乐观的检查

如果你使用 SERIALIZABLE 隔离等级，数据库将不会允许第二个请求，在提交阶段会抛出一个异常。这种方法的缺点是需要更多的系统资源。

如果你使用 SELECT FOR UPDATE ，事务将会竞争数据库，先到的会锁定这行数据，另外一个将会等待。

乐观的检查会不多占用系统资源，也不会锁定数据库。它通过确保数据不会在从数据库读取到提交的期间被改变来排除丢失数据异常。

Django 中避免丢失数据的唯一方法是使用 SELECT FOR UPDATE，而且你必须明确的指定它。如果你遗忘了或者没有意识到，那么这个问题就是就会存在于你的业务逻辑中，有可能导致数据遗失。

三种方法 Pony 都允许使用，第三种方法，乐观的检查，默认是开启的。使用这种方式，Pony 完全避免了数据更新时的丢失问题。使用乐观的检查也允许更高的并发，因为它并不锁定数据库，也不需要额外的资源。

在 Pony 中类似的转账函数将会是这个样子：

```python
@db_session(serializable=True)
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

使用 SELECT FOR UPDATE 的方法
```python
@db_session
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

使用乐观检查的方法
```python
@db_session
def transfer_money(account_id1, account_id2, amount):
    account1 = Account[account_id1]
    account2 = Account[account_id2]
    if amount > account1.amount:
        raise ValueError("Not enough funds")
    account1.amount -= amount
    account2.amount += amount
```

最后这种方法是 Pony 默认的，而且不需要明确的增加别的东西。
