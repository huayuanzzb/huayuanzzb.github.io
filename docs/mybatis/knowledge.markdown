---
layout: post
title:  知识汇总
date:   2019-01-25 17:25:26 +0800
nav_order: 1
parent: mybatis
categories: mybatis
---

### #{} 和 ${} 
### 区别
* ```#{}``` 在 mybatis 初始化完成后会被换成 ?，随后创建 PreparedStatement，可以有效的防止注入
* ```${}``` 是占位符，用于字符串替换，有注入风险。

| mybatis mapper | parameter | mybatis sql | 防注入 |
|----|----|----|----|
|```select id, name as alias_name from User where name = #{name}```| ```name = "google' or true or '1' = '"``` |```select id, name as alias_name from User where name = ?``` | 是 |
|```select id, name as alias_name from User where name = '${name}'```| ```name = "google' or true or '1' = '"```| ```select id, name as alias_name from User where name = 'google' or true or '1' = ''``` | 否 |

### 如何选择
大部分场景应该使用 ```#{}```，但在一些特殊的特殊的场景下，```${}``` 很有用。比如：
1. 有两个表 table_1 和 table_2，需要动态选择表时，可以使用```select * from ${tableName};```
2. 动态指定排序列时，可以使用```select * from table order by ${cols}```

总体来说，如果 sql 中表明，列名甚至整个sql的结构需要动态调整时可以使用 ```${}```，对一个 sql 填充参数时可以使用 ```#{}```