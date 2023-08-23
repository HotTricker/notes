# 数据库（上）

- [数据库（上）](#数据库上)
  - [关系模型基本概念](#关系模型基本概念)
    - [关系模型](#关系模型)
    - [关系](#关系)
    - [关系模型中的完整性](#关系模型中的完整性)
  - [关系模型之关系代数](#关系模型之关系代数)
    - [关系代数之基本操作](#关系代数之基本操作)
    - [关系代数之扩展操作](#关系代数之扩展操作)
  - [关系模型之关系演算](#关系模型之关系演算)
    - [关系演算之关系元组演算](#关系演算之关系元组演算)
    - [关系演算之关系域演算](#关系演算之关系域演算)
    - [关系演算之安全性](#关系演算之安全性)
    - [关于三种关系运算的一些观点](#关于三种关系运算的一些观点)
  - [sql语句](#sql语句)
  - [复杂查询与视图](#复杂查询与视图)
    - [子查询](#子查询)
    - [Sql语言进行结果计算和聚集计算](#sql语言进行结果计算和聚集计算)
    - [sql+关系代数](#sql关系代数)
    - [sql视图](#sql视图)
  - [数据库完整性和安全性](#数据库完整性和安全性)
    - [数据库静态完整性](#数据库静态完整性)
    - [数据库动态完整性](#数据库动态完整性)
    - [自主安全性](#自主安全性)
    - [强制安全性](#强制安全性)

## 关系模型基本概念

***

### 关系模型

关系模型由三部分组成：  

- 数据基本结构形式（Table）
- 表之间可进行的操作
  - 并、差、积、选择、投影、交、连接、除
- 操作应满足的约束
  - 实体完整性、参照完整性、用户自定义完整性

关系运算包括：

- 关系代数

  例如 $\pi_{姓名，课程名}(\sigma_{课程号=c2}(R\Join{S}))$

- 关系演算
  - 元组演算（基于逻辑）
  
    例如 $\{\,t\,|(\exist\,u)(R(t)\wedge{W(u)}\wedge{t[3] < u[1]})\}$

  - 域演算 （基于示例）

    例如$\{\,t_1,t_2,t_3\,|\,S(t_1,t_2,t_3) \wedge R(t_1,t_2,t_3) \wedge t_1<20 \wedge t_2>30 \}$

### 关系

域：一组相同数据类型的值的集合，集合中元素的个数称为域的基数

元组：从每一个域任取一个值所形成的一种组合，所有这种可能组合的集合称为笛卡尔积

关系：笛卡尔积中有意义的那些元组被称为一个关系

关系模式：$R(A_1:D_1,A_2:D_2,...,A_n:D_n), R是关系的名字，A_i是属性，D_i是属性对应的域，n是关系的度，关系中元组的数目为关系的基数$，例如：

$$家庭（丈夫：男人，妻子：女人，子女：儿童）$$
属性向域的映像一般直接说明为属性的类型，长度，比如char(8)

关系与关系模式：关系模式是关系的结构，关系是关系模式在某一时刻的值

关系以内容来区分，而不是位置

关系第一范式：属性不可再分

候选码/候选键：关系中的一个**属性组**，值能唯一的标识一个元组，若从属性组中任意去掉一个属性，则不具有此特性

主键：关系可以有很多候选码，可以选定一个为主键

主属性：包含在任意一个候选码中的属性称为主属性，反之为非主属性

外键：关系R中的一个属性组，不是R的候选码，但是另一个关系S的候选码，则成为外码/外键，两个关系通过外键链接

### 关系模型中的完整性

- 实体完整性
  
  关系的主键的属性值不能为空值
- 参照完整性

  关系A中的外键F对应关系B的主键P，则F的值必须对应B中某个元组的P的值，或者为空
- 用户自定义完整性

  用户针对具体的应用环境定义的完整性约束条件

## 关系模型之关系代数

***

关系代数具有一定的过程性，是一种抽象的语言

### 关系代数之基本操作

- 集合操作
  并、交、差、积
- 纯关系操作
  投影、选择、连接、除

并差交要满足并相容性，当且仅当两个关系的属性数目相同，且任意属性的域也相同（名字可以不同）
对于关系R和关系S：

- 并
  
  R $\cup$ S，由或者出现在R，或者出现在S中的元组组成, sql中使用union

  ```sql
  SELECT * from a where i=1  
  union  
  SELECT * from a where i=2;
  ```

- 差

  R - S，由出现在R，但不出现在S中的元组组成

  ```sql
  SELECT * from student where address
  not in
  (SELECT address from message where add = 'Shanghai');
  ```

- 积

  R x S，R中所有元组和S中所有元组进行串接，属性个数为两者之和，基数为两者之积

  ```sql
  SELECT * from A,B;
  ```

- 选择

  $\sigma{_{con}(R)}$ con由逻辑运算符连接比较表达式组成, 对应 where 关键字

- 投影

  从给定关系中选出某些列组成新的关系，记作$\prod{_{A}(R)}$, 对应 select 关键字

### 关系代数之扩展操作

- 交
  
  R $\cap$ S
  
  ```sql
  SELECT DISTINCT id from t1
  inner join
  t2 USING (id);
  ```

- $\theta$-连接
  
  R 和 S 的笛卡尔积中两个属性上的值满足某些条件的元组构成的关系

  注：和自身连接时往往需要更名操作，记作 $\rho$, mysql 中使用 as 关键字

- 等值连接
  
  $\theta$中运算符为 "=" 时就是等值连接

- 自然连接
  
  自然连接是一种特殊的等值连接，要求必须共有一相同属性组，属性组中的所有值都相等。结果中会去除掉相同的属性列

- 除
  
  R $\div$ S = $\prod_{R-S}(R)-\prod_{R-S}((\prod_{R-S}(R) \times S) - R)$
  
  比如：查询选修了全部课程的学生的学号
  > $$\pi_{S\#,C\#}(SC)\,\div\,\pi_{C\#}(Course)$$

- 外连接
  
  R与S进行连接时，若关系R(或S)中的元组在S(或R)中找不到相匹配的元组，为避免该元组信息丢失，将该元组与S(或R)中假定存在的全为空值的元组形成连接，放置在结果关系中

## 关系模型之关系演算

关系演算是以数理逻辑中的谓词演算为基础的，sql语言继承了关系代数和关系演算各自的优点

### 关系演算之关系元组演算

基本形式：$\{t|P(t)\}$, P(t)可以是以下三种形式之一的原子公式，也可以由公式加运算符递归的构造：
> $t\in R$
>
> $s[A]\,\theta\,c$，元组分量与常量c之间满足比较关系
>
> $s[A]\,\theta\,u[B]$，两个元组分量之间满足比较关系

可以使用存在量词和全称量词，比如：
> $\{t|t\in Student \wedge \exist(u \in Student)(t[Sage]>u[Sage])\}$

### 关系演算之关系域演算

$\{<x_1,x_2,...,x_n>|P(x_1,x_2,...,x_n)\}其中x_i代表域变量或常量，P为以x_i为变量的公式$

元组演算和域演算的区别：

- 元组演算是以元组为变量，先找到元组，再找到元组分量， 进行谓词判断
- 域演算是以域变量为基本处理单位，先有域变量，再判断由这些域变量组成的元组是否存在或满足谓词判断

### 关系演算之安全性

不产生无限关系和无穷验证的运算被称为是安全的

关系代数是安全的，关系演算不一定是安全的，需要对关系演算施加约束，在集合范围内而不是无限范围内进行操作

### 关于三种关系运算的一些观点

- 关系代数
  以集合为对象的操作思维，由集合到集合的变换

- 元组演算
  
  以元组为对象的操作思维，取出关系的每一个元组进行
验证

- 域演算
  
  以域变量为对象的操作思维，取出域的每一个变量进行验
证看其是否满足条件

三种运算是等价的，都可说是非过程性的，非过程性：域演算>元组演算>关系代数

## sql语句

1. 数据库操作

   ```sql
   Create Database TEST;
   Create Table test1;
   Alter Table test1 Add name char(10);
   Alter Table test1 Modify name char(40);
   Alter Table test1 Drop Unique(name);
   Drop Table test1;
   Drop Database TEST;
   use TEST;
   close TEST
   ```

2. 单表查询

   ```sql
   Select ... From ... Where ... ;
   Select DISTINCT name From test1 Where age>14;
   Select name,age From test1 Order By age ASC;
   ASC代表升序，换成DESC降序
   Select name From test1 Where name Like 't%';
   "%"匹配任意多个字符， "_"匹配任意单个字符，"\"匹配转义字符
   ```

3. 增删改

   ```sql
   insert into test1(name, sex, age)
   values ('test_1', 'girl', 14);
   insert into test1 子查询；
   Delete ... From ... Where ... ;
   Update Employee Set Salary = Salary*1.5 where Dname='engineer'
   ```

4. 多表查询

   ```sql
   Select age From test1,test2
   Where test1.name = test2.name and test2.score=100;
   如果两表属性名相同，采用表名.属性名
   Select age as Sage From test1,test2 as new test Where ...;
   ```

## 复杂查询与视图

### 子查询

出现在Where子句中的Select语句，有（NOT）IN-子查询、$\theta -Some/\theta-All$子查询、（NOT）EXISTS子查询三种

(NOT)IN

```sql
SELECT * from Student Where Sname in ("test1", test2);
SELECT * from Student Where Sname in (子查询);
```

$\theta-$查询

```sql
SELECT * from Student Where Score <= all(SELECT Score From Student);
SELECT * from Student Where Score <= some(SELECT Score From Student);
```

(NOT)EXISTS 子查询中有无元组存在

```sql
//检索学过001教师主讲的所有课的学生的名字
Select Sname From Student
Where not exists  //不存在
  (Select * From Course   //有一门001教师讲的课程
    Where Course.T#='001' and not exists //该同学没学过
      (Select * From SC Where S#=Student.S# and C#=Coures.C#));
```

"= some" 和 in 是等价的，"<>= all" 和 not in 是等价的

内层子查询独立进行时称为非相关子查询，需要依靠外层某些参量作为限定条件时称为相关子查询

### Sql语言进行结果计算和聚集计算

Select 后面不仅可以接列名，还可以接计算表达式或聚集函数

```sql
Select Sname,2022-Sage+1 as Syear From Student;
Select AVG(Sage) From Student;
//COUNT、SUM、AVG、MAX、MIN
```

分组：将检索到的元组按照某一条件进行分类，具有相同条件值
的元组划到一个组或一个集合中，同时处理多个组或集合的聚集运算

`Select From [Where ...] [Group by ...];`

比如`Select S#, AVG(Score) From SC Group by S#;`

聚集函数不能用于Where子句，分组过滤应该用Having（需要Group by）支持。`Select S#, AVG(Score) From SC  Where Score<60 Group by S# having Count(*)>2;`

### sql+关系代数

并运算UNION，交运算INTERSECT，差运算EXCEPT，若用ALL修饰，则保留重复元组。mysql不支持INTERSECT和EXCEPT

空值检测：is (not) Null

连接：

```sql
SELECT * FROM table1 natural join table2;//公共属性只出现一次
SELECT * FROM table1 inner join table2 on table1.a1=table2.a1;//a1出现两次
SELECT * FROM table1 left [outer] join table2 on table1.a1=table2.a1; //left,right,full三选一，outer可省略
SELECT * FROM table1 right join table2 using(a1，a2,...); //a1,a2是共有属性的子集，且只出现一次
```

### sql视图

`create view view_name \[列名\] as 子查询 [with check option];`

with check option指明当对视图进行操作时，要检查元组是否满足视图定义中子查询中定义的条件表达式。视图也使用Drop撤销

视图不保存数据，对视图的更新要反映到基本表上，但有时映射不可逆。不能更新的情况：

- SELECT目标列包含聚集函数
- SELECT子句使用UNIQUE、DISTINCT
- 使用了GROUP BY子句
- 包括经算术表达式计算出来的列
- 单个表的列构成但不包含主键

## 数据库完整性和安全性

### 数据库静态完整性

使用断言实现静态完整性，表约束和列约束是特殊的断言，断言创建后系统将检测气有效性，并在每一次更新中测试更新是否违反断言（尽量不使用复杂断言，减小系统负担）

### 数据库动态完整性

使用触发器Trigger实现动态完整性

### 自主安全性

- 通过存储矩阵判断权限
- 通过视图控制
授权机制：GRANT p ON tablename TO somebody;

REVOKE 取消授权

### 强制安全性

对数据对象和用户进行分级，用户权限大于等于该等级时才能读，小于等于该等级时才能写
