---
title: 优化Mysql大数据量Update
date: 2021-11-16 09:58:26
categories:
- Mysql
tags:
- Mysql调优
---

### 引言

最近**废话文学**非常流行，比如什么*听君一席话，如听一席话*，*我家门口有两棵树，一颗是枣树，另一颗也是枣树*。**废话文学**虽然很有趣，但是却大大降低了传递信息的速度。

### 项目中的实际问题

实际项目中经常会遇到这样一类问题：Mysql需要定时修改某张表的状态字段，比如这里需要将一批物联网终端的状态置为unbind(未绑定)。通常情况我们会写成这样一段代码

```go
const MACHINE_COUNT = 10

func update(db *sql.DB, id int) {
	r, err := db.Exec("update machine_state set state = 'unbind' where machine_id = ?", id)
	if err != nil {
		fmt.Println("exec failed, ", r, err)
	}
}

r.GET("/update", func(c *gin.Context) {
		startTime := time.Now().Unix()
		fmt.Println("start time:", startTime)
		for i := 1; i < MACHINE_COUNT; i++ {
			update(db, i)
		}
		endTime := time.Now().Unix()
		fmt.Println("end time:", endTime, "consume:", (endTime - startTime))

		c.JSON(200, gin.H{
			"message": "success",
		})
})
// MACHINE_COUNT = 10000 
// start time: 1637027706
// end time: 1637027784 consume: 78(秒)
```

当我们需要更新的终端数量不是很多时并没有什么问题，但是如果一次需要更新一万台或者更多数量的终端时，更新速度就会变得难以接受，在更新阶段Mysql的负载也会疯狂飙升。

那么联想到我们上诉所说的废话问题，那我们的上段代码翻译成白话文可能是这样的效果：**我有一万台终端的数量需要更新，第一台是1号机械，把它的状态更新为未绑定，第二台是2号机械，把它的状态更新为未绑定，第三台是2号机械，把它的状态更新为未绑定。。。**这段话一说Mysql可就倒了霉了，需要不停的建立TCP链接，解析区别不大的SQL，扫描差不多位置的磁盘。

### 不说废话的replace into

下面就让我们用人狠话不多的**replace into**，来优化一下上面的代码吧

```go
func quicklyUpdate(db *sql.DB, updateMap map[int]string) {
	updateSql := "replace into machine_state(machine_id, state) values"

	for id, state := range updateMap {
		updateSql += fmt.Sprintf("(%d,'%s'),", id, state)
	}
	updateSql = updateSql[:len(updateSql)-1]
	r, err := db.Exec(updateSql)
	if err != nil {
		fmt.Println("exec failed, ", r, err)
	}
}

r.GET("/quickly-update", func(c *gin.Context) {
		startTime := time.Now().Unix()
		fmt.Println("start time:", startTime)
		updateMap := make(map[int]string)
		for i := 1; i < MACHINE_COUNT; i++ {
			updateMap[i] = "online"
		}
		quicklyUpdate(db, updateMap)

		endTime := time.Now().Unix()
		fmt.Println("end time:", endTime, "consume:", (endTime - startTime))

		c.JSON(200, gin.H{
			"message": "success",
		})
})
// MACHINE_COUNT = 10000
// start time: 1637027839
// end time: 1637027840 consume: 1(秒)
```

### replace into原理

replace into 跟 insert 功能类似，不同点在于：replace into 首先尝试插入数据到表中， 如果发现表中已经有此行数据(根据主键或者唯一索引判断)则先删除此行数据，然后插入新的数据。 否则，直接插入新数据。

要注意的是：

1. 插入数据的表必须有主键或者是唯一索引！否则的话 replace into 会直接插入数据，这将导致表中出现重复的数据
2. 因为需要先执行delete操作再执行insert操作，所以如果存在自增长主键的话会导致主键不断变大。
