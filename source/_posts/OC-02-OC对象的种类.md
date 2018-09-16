title: Objective-C底层原理（二）—— OC对象的种类
tags:
  - Objective-C
  - 底层原理
categories:
  - iOS开发
date: 2017-01-02 20:35:50
---

<Excerpt in index | 首页摘要>
Objective-C中的对象共可以分为三种：
    instance对象（实例对象）
    class对象（类对象）
    meta-class对象（元类对象）
<!-- more -->
<The rest of contents | 余下全文>


## instance对象
### 定义：
通过类`alloc`出来的对象，每次调用`alloc`方法都会产生新的instance对象。
### 对象内存中存储的信息：
`instance`对象在内存中存储的信息包括：
1. `isa`指针
2. 其它的成员变量

## Class对象
### 定义：
先看一组OC代码：
```objc
        NSObject *objc = [[NSObject alloc]init];
        Class obj1 = [objc class];
        Class obj2 = [NSObject class];
        Class obj3 = object_getClass(objc);
```
其中obj1、obj2、obj3它们都是`Class`类型的对象
，都是`NSObject`的类对象。
### 对象内存中存储的信息：
class对象在内存中存储的信息主要包括：
1. `isa`指针；
2. `superClass`指针；
3. 类的属性信息(`property`)；
4. 类的对象方法信息(`instance method`)；
5. 类的协议信息(`protocol`)；
6. 类的成员变量信息(`ivar`)；
    * 注意：这里不包含类方法信息(`class method`)，那么类方法存在了什么地方呢？
    
## meta-class对象
### 定义：
什么是元类对象？我们看下面一行代码：
```objc
        Class metaClassObj = object_getClass([NSObject class]);
```
`metaClassObj`是我们通过运行时获取的`NSObject`类对象的类对象，其本质上依然是一个类对象。那么这个`metaClassObj`就是`NSObject`的元类对象(`meta-class`)，并且每个类在内存中有且只有一个元类对象。

### 对象内存中存储的信息：
`meta-class`和`Class`的内存结构是一样的，但是用途不一样，在内存中存储的信息主要包括：
1. `isa`指针
2. `superclass`指针
3. 类的类方法信息(`class method`)
4. ...

## 比较获取类对象的方法
 
##### `Class objc_getClass(const char *aClassName)`

 1> 传入字符串类名
 2> 返回对应的类对象
 
##### `Class object_getClass(id obj)`
 
 1> 传入的obj可能是instance对象、class对象、meta-class对象
 2> 返回值
 a) 如果是instance对象，返回class对象
 b) 如果是class对象，返回meta-class对象
 c) 如果是meta-class对象，返回NSObject（基类）的meta-class对象
 
##### `- (Class)class、+ (Class)class`
返回值就是类对象
```objc 
 - (Class) {
     return self->isa;
 }
 ```
 ```objc
 + (Class) {
     return self;
 }
 ```
## 三种对象之间的关系（方法调用原理）
instance的isa指向class

class的isa指向meta-class

meta-class的isa指向基类的meta-class

class的superclass指向父类的class
如果没有父类，superclass指针为nil

meta-class的superclass指向父类的meta-class
基类的meta-class的superclass指向基类的class

instance调用对象方法的轨迹
isa找到class，方法不存在，就通过superclass找父类

class调用类方法的轨迹
isa找meta-class，方法不存在，就通过superclass找父类

如下图所示：
![](http://peqakd06c.bkt.clouddn.com/15364882268573.jpg)




