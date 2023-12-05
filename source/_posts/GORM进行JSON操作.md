---
title: GORM进行JSON操作
date: 2022-04-20 11:24:01
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/golang.png
tags:
    - Go
categories:
    - Go
---
## 在Go中使用GORM进行数据库操作

gorm作为go常见的ORM库，使用起来也是很方面的。
得赖于go简单易用的特性，我们也只需要在启动配置中引入相关的库，进行初始化操作，就能进行对数据库的操作。
```go
func initMysql() *gorm.DB {
	db, err := gorm.Open(mysql.Open("root:123456@(localhost:3306)/yhhy?charset=utf8mb4&parseTime=True"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	sqlDB, err := db.DB()
	if err != nil {
		panic("connect db server failed")
	}
	// 可以进行连接池配置
	sqlDB.SetConnMaxLifetime(600 * time.Second)
	sqlDB.SetMaxOpenConns(50)
	sqlDB.SetMaxIdleConns(10)
	return db
}
```

### 简单操作(主要介绍JSON操作)

一般数据库都有json或类似的字段，需要我们把对象或者集合等规则数据转换为字符串进行存储到数据表中，gorm自带的数据类型操作是没有相关类型，所以需要我们自己对其进行编写。
官网代码如下：我们自定义JSON类型，通过实现gorm的两个接口，来完成数据的转换
```go
type JSON json.RawMessage

// Value 从输入转化为JSON
func (j JSON) Value() (driver.Value, error) {
	if len(j) == 0 {
		return nil, nil
	}
	return json.RawMessage(j).MarshalJSON()
}

// Scan 从数据库JSON字段转换为model对应JSON字段
func (j *JSON) Scan(value interface{}) error {
	bytes, ok := value.([]byte)
	if !ok {
		return errors.New(fmt.Sprint("Failed to unmarshal JSONB value:", value))
	}

	result := json.RawMessage{}
	err := json.Unmarshal(bytes, &result)
	*j = JSON(result)
	return err
}
```
使用如上代码，我发现，如果保存数据，这个流程是正确的，但是，拿取数据，却是有问题的，按照官网的例子，会生成一个加码的字符串。

为了解决这个问题，我重新定义了JSON的类型。代码如下：
我是用map集合来作为新类型的数据结构基础，然后进行数据转换，数据就是正确的.
```go
type JSONB map[string]interface{}

func (j JSONB) Value() (driver.Value, error) {
	marshal, err := json.Marshal(j)
	if err != nil {
		return nil, err
	}
	return string(marshal), nil
}

func (j *JSONB) Scan(value interface{}) error {
	if err := json.Unmarshal(value.([]byte), &j); err != nil {
		return err
	}
	return nil
}
```
