---
title: sqlalchemy十分钟入门
date: 2016-12-24 22:05:43
excerpt: "sqlalchemy十分钟入门"
tags: mysql  sqlalchemy
---

接触ORM是在两年前接手开发到一半的模块时，模块后端使用mysql作为存储，采用python开发。当时就惊讶于ORM开发的迅捷，想安利一把sqlalchemy很久了，今天终于得空，介绍下python ORM的利器--**sqlalchemy**.

<!--more-->

ORM即Object Relational Mappers，将数据库的一行转变为对象的操作，上层只需要关注对象的读写，真正与数据库的交互隐藏到了package里。使得开发变得非常简便。本文主要介绍sqlalchemy的入门用法，试图在短时间内让读者可以掌握使用的技巧，更多的细节会给出对应的文档章节，建议深入阅读。

话不多说，下面开始。

### 1. SQLAlchemy

SQLAlchemy支持多种数据库（如SQLite/MySQL/Postgres/Oracle)等，本文使用mysql作为后端数据库引擎。

安装使用`pip install sqlalchemy`即可。

安装好后可以导入看下版本号

```python
>>> import sqlalchemy
>>> sqlalchemy.__version__
'0.9.7'
```

### 2. 数据库的连接

数据库的连接使用`engine`完成

```python
#!/usr/bin/env python

from sqlalchemy import create_engine

connect_params = {
    "db": "mysql",
    "user": "test",
    "password": "123456",
    "ip": "x.x.x.x",
    "port": 3306,
    "database": "test"
}

connect_string = "%(db)s://%(user)s:%(password)s@%(ip)s:\
    %(port)s/%(database)s" % connect_params

engine = create_engine(connect_string, echo=True)
```

使用`create_engine`与数据库后端进行链接。

注意格式里`ip`以及其他字段替换成你的mysql端口，`echo=True`表示回显，设置后sqlalchemy发起的数据库请求都会打印到屏幕上。

这里sqlalchemy使用了`Lazy Connecting`的策略：

> The Engine,when first returned by create_engine(), has not actually tried to connect to the database yet;that happends only the first time it is asked to perform a task against the database.


### 3. 表的模型

创建engine后，我们先看下在sqlalchemy里，如何把对象映射成一张表，我们定义了`users`表。

```python
from sqlalchemy import Column, Integer, String

class User(Base):
    __tablename__ = 'users'
    __table_args__ = {'mysql_engine': 'InnoDB'}

    id = Column(Integer, primary_key=True)
    name = Column(String(32))
    fullname = Column(String(32))
    password = Column(String(32))

    def __repr__(self):
        return "<User(name='%s, fullname='%s', password='%s)" % (
            self.name, self.fullname, self.password)
```

`Interger String`对应了mysql定义时的数据类型，更多类型可以参考*3.2.5 Column and Data Types*一节。

可以看到我们建立了表`users`，使用InnoDB作为存储引擎，共有4列，执行下一句可以在后端建立对应的table

```python
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
Base.metadata.create_all(engine)
```

注意`create_all`会建立所有`Base`子类的表，`drop_all`则删除这些表。

执行完后，回显命令行显示建表成功

```
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(32) DEFAULT NULL,
  `fullname` varchar(32) DEFAULT NULL,
  `password` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8
```

可以看到table已经按照需要的格式建立，表格跟`User`类一一对应，更具体的对应方式，我们从数据库的操作来看下。

### 4. 数据库的操作

看下如何插入新的一行。

在ORM里，数据库插入新的一行，相当于构造一个新的对象传递给package。在这之前，我们需要基于`engine`发起一次transcation。

```python
from sqlalchemy.orm import sessionmaker

SuperSession = sessionmaker(bind=engine)
session = SuperSession()
```

基于`session`对象，我们就可以像操作本地对象一样，操作远程数据库了。

例如添加新的一行

```python
session = SuperSession()
ed_user = User(name='ed', fullname='Ed Jones', password='edspassword')
session.add(ed_user)
session.commit()
```

session.commit()提交了本次`transcation`，打开`echo=True`开关后，我们可以看到标准输出

```
2016-12-24 22:42:35,084 INFO sqlalchemy.engine.base.Engine BEGIN (implicit)
2016-12-24 22:42:35,085 INFO sqlalchemy.engine.base.Engine INSERT INTO users (name, fullname, password) VALUES (%s, %s, %s)
2016-12-24 22:42:35,085 INFO sqlalchemy.engine.base.Engine ('ed', 'Ed Jones', 'edspassword')
2016-12-24 22:42:35,086 INFO sqlalchemy.engine.base.Engine COMMIT
```

如果没有执行`session.commit`，那么本次修改相当于没有提交到数据库，实际更新是无效的。

接着使用`query`查询下修改是否生效

```python
print session.query(User).filter(User.name == 'ed').first()
```

输出

```
<User(name='ed, fullname='Ed Jones', password='edspassword)
```

关于`query`的更多细节，可以参考*2.1.9 Querying*

看下删除的用法

```python
local_jack = User(name='jack', fullname='Jack Sparrow', password='black perl')
session.add(local_jack)
session.commit()

print session.query(User).filter_by(name='jack').count()

db_jack = session.query(User).filter_by(name='jack').one()
session.delete(db_jack)
session.commit()

print session.query(User).filter_by(name='jack').count()
```

### 5. 外键约束

首先我们建立一个与`User`有约束的表

```python
class Address(Base):
    __tablename__ = 'address'
    
    id = Column(Integer, primary_key=True)
    email_address = Column(String(32), nullable=False)
    user_id = Column(Integer, ForeignKey('users.id'))

    user = relationship("User", backref=backref('address', order_by=id))
```

使用`Base.metadata.create_all(engine)`后，我们看下`address`表的定义格式

```
CREATE TABLE `address` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `email_address` varchar(32) NOT NULL,
  `user_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`),
  CONSTRAINT `address_ibfk_1` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

看下删除`users`关联的一行如果被删除是什么效果

```
mysql> select * from test.address;
+----+---------------+---------+
| id | email_address | user_id |
+----+---------------+---------+
|  1 | ufo@gmail.com |    NULL |
+----+---------------+---------+
```

可以看到`user_id`被设置了`NULL`，更多参考*ForeignKey*一节。

### 6. 实践经验

`session`需要及时释放，建议使用`enter/leave`的方式封装，例如

```python
    class InitedSession(object):
        def __init__(self, *arg, **warg):
            self.session = superSession(*arg, **warg)

        def __enter__(self):
            return self.session

        def __exit__(self, *arg, **warg):
            logging.debug('session exit get args.[arg:%s][warg:%s]' % (str(arg), str(warg)))
            self.session.close()
            if len(arg) >= 3 and arg[2] != None:
                logging.error(''.join(traceback.format_tb(arg[2])) + str(arg[1]))
            if arg[2] != None:
                traceback.print_exception(*arg)

        def __getattr__(self, name):
            return getattr(self.session, name)
```

`sqlalchemy`里的`connect`默认不是transcation，例如：

```
engine.connect().execute(text("show databases"));
```

返回一个`ResultProxy`对象，使用`for user in xxx`可以遍历所有符合条件的对象。

在使用过程中，也遇到了不少问题，例如[select * for update的用法](http://izualzhy.cn/sqlalchemy-with-lockmode-update-question)。

更详细的用法，建议参考sqlalchemy的文档，放了一份在[百度云](https://pan.baidu.com/s/1i5fWSst)上。
