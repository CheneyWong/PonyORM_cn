# 关系

实体对象可以和其他实体对象建立关系。一条关系是由关系两边的实体对象的两条属性定义的：
```python
class Customer(db.Entity):
    orders = Set("Order")

class Order(db.Entity):
    customer = Required(Customer)
```
在上面的示例中，我们有两个关系属性：`orders` 和 `customer`。当我们定义 `Customer` 的时候， `Orders` 还没有定义，所以需要使用引号将 `Orders`o 括起来。也可以使用 lambda 表达式：

```python
class Customer(db.Entity):
    orders = Set(lambda: Order)
```
如果你想要 IDE 帮你检查实体对象名字的拼写高亮拼写错误，这种方法很有用。

有一些映射器（例如，django）只需要在一边定义关系就可以了。Pony 需要在关系的两边都明确的定义（就像 Python 之禅说的：明确比暗示好），用户在每个角度都能看到所有的关系定义。

所有的关系都是双向的。如果你更新了关系的一边，另一边会自动更新。如果我们创建了一个 `Orders` 的实例，customer 的 orders 集合会更新，添加了这条新的 order。

有三种类型的关系：一对一，一对多，多对多。一对一关系很少使用，大多数关系形式都是一对多和多对多。如果两个实体对象拥有一对一关系，那基本上就意味着这两个实体对象可以合并为一个。如果你的数据表中包含很多一对一关系，那就意味着你需要重新思考实体对象的定义了。

## 一对多关系
这里有一个一对多关系的例子：
```python
class Order(db.Entity):
    items = Set("OrderItem")

class OrderItem(db.Entity):
    order = Required(Order)
```
上例中，没有 order 的情况下就不能建立 `OrderItem` 的实例，如果我们想建立一个不需要指定 order 的 `OrderItem` 实例，我们可以指定 `order` 属性为 `Optional`。

```python
class Order(db.Entity):
    items = Set("OrderItem")

class OrderItem(db.Entity):
    order = Optional(Order)
```

## 多对多关系
为了建立多对多关系，你需要在关系的两边都使用 `Set` 属性。
```python
class Product(db.Entity):
    tags = Set("Tag")

class Tag(db.Entity):
    products = Set(Product)
```
为了在数据库中实现这个关系，Pony 将会创建一个中间表。这是众所周知的在数据库中建立多对多关系的方法。

## 一对一关系
为了建立一对一关系，关系属性需要设置为 `Optional-Required` 或 `Optional-Optional`:

```python
class Person(db.Entity):
    passport = Optional("Passport")

class Passport(db.Entity):
    person = Required("Person")
```
两个属性都是 `Required` 是不允许的，因为这不合理。

## 自引用
实体对象可以通过自引用建立指向自己的关系。这种关系可以有两种类型：相称的和不相称的。不相称的关系就是一个实体对象里的两个属性建立的关系。

相称的关系特殊的地方就是这条关系只有一条属性，这条属性定义了关系的两边。这种关系可以是一对一也可以是多对多。如下是自引用关系的例子：
```python
class Person(db.Entity):
    name = Required(str)
    spouse = Optional("Person", reverse="spouse") # symmetric one-to-one
    friends = Set("Person", reverse="friends")    # symmetric many-to-many
    manager = Optional("Person", reverse="employees") # one side of non-symmetric
    employees = Set("Person", reverse="manager") # another side of non-symmetric
```

## 两个实体对象间的多种关系
当两个实体类型之间的关系多于一条的时候，Pony 需要 reverse 属性来加以区分。这是为了让 Pony 知道那两个属性是一对。像我们考虑一下这个 user 既可以写微博，又可以给微博点赞的数据表。
```python
class User(db.Entity):
    tweets = Set("Tweet", reverse="author")
    favorites = Set("Tweet", reverse="favorited")

class Tweet(db.Entity):
    author = Required(User, reverse="tweets")
    favorited = Set(User, reverse="favorites")
```
在上例中，我们必须指定 `reverse` 属性。当你尝试生成映射但是没有提供`reverse` 属性时，你会得到一个异常  `pony.orm.core.ERDiagramError: "Ambiguous reverse attribute for Tweet.author"`。之所以会这样是因为 `author` 属性在技术角度既可以和 `tweets` 配对，又可以和 `favorites` 配对，Pony 没有足够的信息区分。



