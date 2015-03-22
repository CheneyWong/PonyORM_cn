# 聚合

Pony 在声明请求的时候允许使用 5 个聚合函数： `sum`、`count`、`min`、`max` 和 `avg`。让我们用这些函数做一些简单的请求。
101组的学生的总GPA：
`sum(s.gpa for s in Student if s.group.number == 101)`
GPA大于3的学生的数量：
`count(s for s in Student if s.gpa > 3)`
学习哲学的学生的名字，按照字母表排序：
`min(s.name for s in Student if "Philosophy" in s.courses.name)`
101组学生中年龄最小的人的生日：
`max(s.dob for s in Student if s.group.number == 101)`
44部门的平均GPA：
`avg(s.gpa for s in Student if s.group.dept.number == 44)`
>注
即使 Python 有标准函数 `sum`、`count`、`min` 和 `max`，Pony 添加了自定义的同名函数，而且 Pony 添加了 `avg` 函数。这些函数在 `pony.orm` 中实现，他们可以通过星号或者指定名字的方式导入。
Pony 中的函数扩展了 Python 的标准函数的功能。这样，如果程序中用到他们的标准声明，导入不会影响原来的功能。但是她让你可以在函数中指定一个数据库请求。
如果有人忘了导入，在请求中使用 Python 标准函数`sum`、`count`、`min` 和 `max`会抛出异常：
`TypeError: Use a declarative query in order to iterate over entity`

聚合函数在一个请求内部也可以使用，例如，如果我们不仅要找团队中最年轻的学生的生日，而且还有这个学生本身：
```python
select(s for s in Student
       if s.group.number == 101
       and s.dob == max(s.dob for s in Student
                        if s.group.number == 101))
```
或者，作为示例，列出平均 GPA 大于 4.5 的组：
`select(g for g in Group if avg(s.gpa for s in g.students) > 4.5)`
如果使用 Pony 的属性的扩展特性，这个请求可以更短。
`select(g for g in Group if avg(g.students.gpa) > 4.5)`

## 一个请求中使用多个聚合函数
SQL 允许你在一个请求中包含多个聚合函数。例如，我们想要每个组中 GPA 最高和最低的值。在 SQL 中，这个请求大致如此：
```sql
SELECT s.group_number, MIN(s.gpa), MAX(s.gpa)
FROM Student s
GROUP BY s.group_number
```
这个请求返回了每个组中 GPA 最高和最低的值。Pony 也可以让你达到相同的目地。
`select((s.group, min(s.gpa), max(s.gpa)) for s in Student)`

## Grouping
相同的请求，Pony 中比 SQL 中看起来更短，因为在 SQL 中我们必须包含 “GROUP BY” 标签。而 Pony 能够理解这个需求，自动进行了添加。Pony 是怎么做的呢？
人们可能会注意到，通过SQL查询部分组包括的列，也包括在 SELECT 段，但不包含在聚集函数。那就是为什么要在 SQL 中重复这些列。Pony 可以 避免这种重复，因为它明白如果一个表达式包含在查询结果中，但是不包含在聚集函数中，它应该自动添加到 GROUP BY 段。






