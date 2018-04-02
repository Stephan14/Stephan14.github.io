---
layout:     post
title:      "mysql update使用"
subtitle:   "大量数据update优化"
date:       2018-04-01 15:46:00
author:     "邹盛富"
header-img: "img/update.jpeg"
tags:
    - mysql
---

### 场景

最近使用Python语言基于SQLAlchemy编写一些查数据的操作，要向数据库中大量更新数据（大约50000行的数据），本来是通过modle的方式更新数据库中的相应的数据，这样开发成本比较低，比较方便，代码如下：

```

if reserved_empty:
    if len(reserved_empty) < len(codes):
        replace_codes = codes[:len(reserved_empty)]
        for i in range(0, len(replace_codes)):
            reserved_empty[i].code = replace_codes[i]
        session.GameWriteSession.commit()
        codes = codes[len(reserved_empty):]
    else:
        replace_codes = codes
        for i in range(0, len(replace_codes)):
            reserved_empty[i].code = replace_codes[i]
        session.GameWriteSession.commit()
        return
else:
    logging.info('')

```
但是，在实际的测试中却发现这种使用方式的性能很低，更新50000行数据大约需要2分钟左右，因此还需要进行相应的优化

### 解决方法

通过查找资料发现，对于SQLAlchemy中这些优化，大多数的方法是使用拼接mysql语句的方式来进行优化。因此只能先尝试使用拼接sql语句的方式来解决这个问题，因为这些数据已经存在，所以尝试使用update语句进行更新，通常解决上述问题的方式通过循环的方式多次更新其中的值，代码如下：

```
if reserved_empty:
    if len(reserved_empty) < len(codes):
        replace_codes = codes[:len(reserved_empty)]
        for i in range(0, len(replace_codes)):
            sql = 'UPDATE gift_code SET code=' + replace_codes[i] + 'WHRER id=' + reserved_empty.id
            session.GameWriteSession.execute(sql)
            session.GameWriteSession.commit()
    else:
        replace_codes = codes
        for i in range(0, len(replace_codes)):
            sql = 'UPDATE gift_code SET code=' + replace_codes[i] + 'WHRER id=' + reserved_empty.id
            session.GameWriteSession.execute(sql)
            session.GameWriteSession.commit()
        return
```
上述这种方法并没有什么问题，但是循环中不止执行了一次SQL查询，系统优化的时候，总是想尽可能的减少数据库查询的次数，以减少资源占用，同时可以提高系统速度，因此可以尝试另一种更好的方法

### 优化方法

优化的代码如下：

```
if reserved_empty:
    pre_sql = 'UPDATE gift_code SET code CASE id '
    ids = []
    replace_codes = []
    if len(reserved_empty) < len(codes):
        replace_codes = codes[:len(reserved_empty)]
        codes = codes[len(reserved_empty):]
    else:
        replace_codes = codes
        codes = codes[len(reserved_empty):]

    for i in range(0, len(replace_codes)):
        pre_sql += ' WHEN ' + reserved_empty.id + ' THEN ' +  replace_codes[i]
        ids = append(ids, reserved_empty.id)
    pre_sql += ' ELSE code'
    suffix_sql = ' WHERE id IN(' + ','.join(ids) + ')'
    sql = pre_sql + suffix_sql
    session.GameWriteSession.execute(sql)
    session.GameWriteSession.commit()

    if len(code) <= 0:
        return
else:
    logging.info('')

```

在上述代码中，通过更复杂的只与数据库进行了一次交互，提高了程序的性能,如果想要修改一行中的多列可以使用更复杂的SQL语句

参考连接

[MySQL 更新多条记录的不同值](https://crispgm.com/page/mysql-update-multirows.html)

[Multiple Updates in MySQL](https://stackoverflow.com/questions/3432/multiple-updates-in-mysql)
