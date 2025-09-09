---
layout:     post
title:      "Private Member Access"
subtitle:   "私有成员访问"
date:       2025-09-09 18:48:00
author:     "Nuoyumi"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - cpp 
    - private member access	
    - private field access	
    - private function access	
    - template programming
    - meta programming
    - explicit template instantiation 
---


在cpp中，有时候在不得已的情况下，需要对第三方库中类的私有数据或者函数进行访问，修改或调用，
从而实现某些特定的需求或者奇效，比如facebook folly中修改string和vector的size（[引用](https://github.com/facebook/folly/blob/f63a5c31680aaabcc0f9c86709fd4a813db292ce/folly/memory/UninitializedMemoryHacks.h)）。
接下来将提供一种通用方便的方式来实现私有成员数据或函数的访问。

## 准备
假设存在以下第三方的类，需要访问或者修改其私有数据成员data，或者调用其私有函数fn。
```
class Private{
	int data;
	int fn(int x) {
		return data + x;
	}
};
```

## 目标
通过一些注册操作:
```
class TagData{};
XXX...TagData...YYY...&Private::data...
class TagFn{};
XXX...TagFn...YYY... &Private::fn...
```
我们可以通过如下方式操作私有成员：
```
Private obj;
auto& data = TagField<TagData>(obj); 
data = 2;
assert( 5 == TagFunctor<TagFn>(obj, 3) );
```

## 方案
在正常情况下，私有成员是无法被访问的。那***不正常***情况下呢？
```
17.7.2 (item 12)
The usual access checking rules do not apply to names used to specify explicit instantiations. 
[Note: In particular, the template arguments and names used in the function declarator 
(including parameter types, return types and exception specifications) may be private types 
or objects which would normally not be accessible and the template may be a member template 
or member function which would not normally be accessible.]
```
简而言之，就是模板实例化的时候，模板参数，返回值等可以使用平时不被允许的私有成员。

### Tag关联私有成员地址
私有成员的地址已经可以在模板实例化中指定了，那怎么拿到这个值从而访问私有成员呢？
总体的思路是，通过在模板实例化中绑定一个Tag到这个地址:
```
template<class Tag, class MemPtr>
struct MemPtrHolder {
	static MemPtr value;

	template<MemPtr memPtr>
	struct Initializer{
		Initializer(){
			value = memPtr;
		}
		static Initializer initializer;
	};	
};

template<class Tag, class MemPtr>
MemPtr MemPtrHolder<Tag, MemPtr>::value;

// initializer必须在value之后定义，从而在value后面初始化来保证value=memPtr
template<class Tag, class MemPtr>
template<MemPtr memPtr>
class MemPtrHolder<Tag,MemPtr>::template Initializer<memPtr> MemPtrHolder<Tag,MemPtr>::Initializer<memPtr>::initializer;
```

### 通过Tag获取私有成员地址使用
然后通过Tag来获取私有成员地址，从而访问私有数据，调用私有函数:
```
template<class>
struct TagMemPtr;

template<class Tag, class T, class... Args>
decltype(auto) TagFunctor(T&& obj, Args&&... args){
	using MemPtr = typename TagMemPtr<Tag>::type;
	auto memPtr = MemPtrHolder<Tag, MemPtr>::value;	
	return (obj.*memPtr) (std::forward<Args>(args)...); 
}

template<class Tag, class T, class... Args>
decltype(auto) TagField(T&& obj, Args&&... args){
	using MemPtr = typename TagMemPtr<Tag>::type;
	auto memPtr = MemPtrHolder<Tag, MemPtr>::value;	
	return obj.*memPtr; 
}
```

### 注册私有成员
在使用之前，需要对私有成员进行注册：
```
#define DECLARE_PRIVATE_MEMBER(Tag, MemPtr, memPtr) 			\
template class MemPtrHolder<Tag,MemPtr>::template Initializer<memPtr>;	\
template<>								\
struct TagMemPtr<Tag>{							\
	using type = MemPtr;						\
};									
```

## 使用示例
```
struct TagData {};
DECLARE_PRIVATE_MEMBER(TagData, int Private::*, &Private::data)
struct TagFn {};
DECLARE_PRIVATE_MEMBER(TagFn, int(Private::*)(int), &Private::fn)

int main() {
	Private obj;
	auto& data = TagField<TagData>(obj); 
	data = 2;
	assert( 5 == TagFunctor<TagFn>(obj, 3) );
}
```

## 总结
本文通过显示模板实例化模板，来捕获私有成员地址，结合Tag来关联私有地址，最终使用私有地址来访问私有成员。
其中一个关键是，通过静态变量initializer来存储私有地址memPtr。
那么有没有更好的方法通过Tag获取私有地址呢？敬请期待下文：friend-injection。
