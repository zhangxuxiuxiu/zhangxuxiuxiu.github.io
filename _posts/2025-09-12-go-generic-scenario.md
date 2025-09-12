---
layout:     post
title:      "Golang Generic Scenarios"
subtitle:   "泛型的有趣应用"
date:       2025-09-12 11:38:00
author:     "Nuoyumi"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - golang 
    - generic programming 
---

在golang中，泛型有很多有趣的应用。


## 反序列化返回一个接口的实现
比如函数 `func Unmarshal[T any](body []byte, val T) (Hello,error)`
此时要求T是一个指针类型，且实现了接口Hello。参考如下：
```
func Unmarshal[T any, U interface {
	Hello
	*T
}](body []byte, val U) (error) {
	return json.Unmarshal(body, val)
}
```
或者实现如下：
```
func Unmarshal[T any, U interface {
	Hello
	*T
}](body []byte) (Hello, error) {
	var val T
	if err := json.Unmarshal(body, &val); err != nil {
		return nil, err
	}
	return U(&val), nil
}
```

## 实现单例模式
```
func DoOnce[T any](fn func() T) func() T {
	var once sync.Once
	var val T
	return func() T {
		once.Do(func() {
			val = fn()
		})
		return val
	}
}
```


