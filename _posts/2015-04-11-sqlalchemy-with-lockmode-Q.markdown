---
title:  "sqlalchemy使用with_lockmode的问题记录"
date: 2015-04-11 20:37:35
excerpt: "sqlalchemy使用with_lockmode的问题记录"
tags: [sqlalchemy]
---

最近在使用sqlalchemy，关于`select * for update`  
有一些疑惑的地方，记录在此。  
当Isolation Level是RR级别时，在一个会话里是允许幻读的，即`select * from x`的结果是会话刚开始的一个快照，无法读取其他会话修改commit的内容。  
如果需要当前读，可以使用`select * for update`。  
但是当前读在使用sqlalchemy里的with_lockmode时却表现不一致，即仍然是快照读。问题的原因我一直没有想出来，希望看到的朋友能够指点下。  

<!--more-->

首先创建一个测试用表：  

```
CREATE TABLE `user_test` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` char(30) DEFAULT NULL,
     PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

插入几条数据:

```
insert into `user_test` (name) values ('a');
insert into `user_test` (name) values ('b');
insert into `user_test` (name) values ('c');
```
sqlalchemy对应的model:  

```
class User(BaseModel):
    __tablename__ = 'user_test'
    __table_args__ = {
        'mysql_engine': 'InnoDB'
    }

    id = Column(Integer, primary_key=True)
    name = Column(CHAR(30))

    def __str__(self):
        return 'User[%d, %s]' % (self.id, self.name)
```
数据构造完毕后，写了一个简单的测试用例如下：

```
session = DB_Session()
u = session.query(User).filter(User.name == 'a').first()
locking_u = session.query(User).\
    with_for_update().\
    filter(User.id == u.id).\
    first()
for u in engine.connect().execute("select * from user_test where id = %d" % u.id):
    print u
if locking_u and locking_u.name == 'a':
    print 'need to change'
    locking_u.name = 'aa'
else:
    print 'no need to change'
time.sleep(10)
session.commit()
```
测试过程如下：  
1. terminal-1运行该程序，在L14 sleep 10秒，然后释放x锁.  
2. terminal-2运行该程序，在L3-L6等待x锁释放，然后检测锁定的这行的值，如果没有改变则修改。如果已经修改，则只打印。  
预期terminal-1在L15`session.commit()`释放x锁后,terminal-2读到的值是terminal-1修改后的值，不应做改动。  
但实际上**读到的仍然是修改前的值，与mysql预期的行为不符。同时L52-L53的通过connection而不是session的输出可以看到terminal-1修改后的值的，说明修改已经生效了。**  

解决方法：  
1. 在L48-L51加锁时增加一句话:populate\_existing，但该函数是否会导致其他问题，我还没有搞清楚。  
2. 在查找u的时候，即L2，不是查找整行，而是只查找u的id列，即修改为:   
`u_id = session.query(User.id).filter(User.name == 'a').first()`  
对应的L48-L51直接使用u\_id，不再使用u.id。

根据以上两点猜测可能跟sqlalchemy的对象的生效期有关？希望了解的朋友留言告诉我，谢谢。


