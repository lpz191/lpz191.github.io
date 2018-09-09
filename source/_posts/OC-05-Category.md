title: Objective-C底层原理（五）—— Category
tags:
  - Objective-C
  - 底层原理
categories:
  - iOS开发
date: 2017-01-05 20:15:50
---

<Excerpt in index | 首页摘要>
Category的实现原理是什么？
Category和class extension有什么区别？
Category中的load和initialize方法怎么使用？
Category能否添加成员变量？
<!-- more -->
<The rest of contents | 余下全文>

## 序言
我们平时写OC代码，写的基本也都得心应手。
比如创建一个对象，

``` objc
NSObject *obj = [[NSObject alloc]init];
```
那么这个obj占用多少内存，内存又是如何分配的，对象又是如何产生的？

## Objective-C 类的本质

OC代码的底层都是` C/C++` 实现的,所以OC的面向对象都是基于`C/C++`的数据结构来是实现的，那么现在思考一个问题，OC的类和对象是基于`C/C++`的什么`数据结构`实现的？

现在我们打开Xcode，创建一个Command line项目，我们在`main.m`文件中写入以下代码：
```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *objc = [[NSObject alloc]init];
    }
    return 0;
}
```
下面我们找到`main.m`所在的文件目录，运行`Terminal`,并执行以下命令,将`main.m`文件重写成更底层的`C/C++`代码
#### 不指定架构重写
```bash
$ clang -rewrite-objc main.m -o main.cpp
```
由于我们现在大都是在64位的平台进行开发，或者
#### 直接指定arm64架构重写
```bash
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.cpp
```
这样输出的文件会比不指定架构时要小很多。
这里我选择了指定`arm64`架构命令，执行完毕，我们会在原来的文件目录下看到`main.cpp`文件，打开这个文件，并在`7158行`找到如下代码
```Objc
struct NSObject_IMPL {
	Class isa;
};
```
如果我们不将代码转成`C/C++`底层的代码，我们直接通过代码跳转的方式去看NSObject的实现，我们会看到如下代码
```Objc
@interface NSObject <NSObject> {
    Class isa;
}
```
类似的结构，同样的的实例，它们都有且仅有一个`Class`类型的`isa`,这样我们可以从侧面证明，这个结构体就是我们要找的`NSObject`类的底层实现。通过这个一系列的操作，我们发现，`Objective-C`类的底层数据结构其实就是`C/C++`的结构体。

追根溯源，我们继续点击这个`Class`跳转到`Class`定义的地方。
我们能看到如下代码：
```objc
typedef struct objc_class *Class;
```
原来`Class`是一个指向结构体的指针，那么这个对象占用多少内存呢？

## Objective-C 对象的本质
我们知道，一个指针在`64-bit`系统下占用8个字节的内存，但事实上系统为这个对象仅仅分配了8个字节的内存吗？我们用如下的代码来证实一下：

```objc
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import <malloc/malloc.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *objc = [[NSObject alloc]init];
        //  获得NSObject类的实例对象的大小（runtime方法）
        NSLog(@"%zd", class_getInstanceSize([NSObject class]));
        //  获得objc指针所指向的内存大小（malloc方法）
        NSLog(@"%zd", malloc_size((__bridge const void *)(objc)));
    }
    return 0;
}
```
点击运行，这时我们看到输出结果如下：
```
2018-09-02 14:39:25.626472+0800 Main[26564:3507573] 8
2018-09-02 14:39:25.627123+0800 Main[26564:3507573] 16
Program ended with exit code: 0
```
一个输出8，一个输出16。也就是说NSObject类的实例对象的大小并不是objc对象所占用的内存大小，这到底是怎么回事呢？
基于Apple目前开源的源码，我们打开[苹果源码](https://opensource.apple.com/tarballs/objc4)，并下载[objc4-723.tar.gz](https://opensource.apple.com/tarballs/objc4/objc4-723.tar.gz)解压后打开这个项目。

#### class_getInstanceSize
我们直接全局搜索`class_getInstanceSize`，然后找到`objc-class.mm`文件中的对应的实现代码：
```objc
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}
```
继续查看`alignedInstanceSize()`方法的实现：
```objc
   // Class's ivar size rounded up to a pointer-size boundary.
    uint32_t alignedInstanceSize() {
        return word_align(unalignedInstanceSize());
    }
```
这里有一句注释，翻译成中文就是`类的实例对象的成员变量所占用的内存大小`，这个值就是`8`。

#### allocWithZone
我们继续全局搜索`allocWithZone`，然后找到`NSObject.mm`文件中定义的这个函数，
```c
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone)
```
并且函数内部找到以下这句代码调用，
```objc
obj = class_createInstance(cls, 0);
```
并跳转到`class_createInstance`函数的实现，我们可以看到如下的代码
```c
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}
```
我们继续跳转到`_class_createInstanceFromZone`函数的实现，在这个函数的实现内部，我们发现了有以下代码
```c
    if (!zone  &&  fast) {
        obj = (id)calloc(1, size);
        if (!obj) return nil;
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
```
这里的obj在创建的时候，调用了一个`calloc`方法，并将`size`作为参数传了进去，这里的`size`哪里来的呢？我们跳转到`size`定义的地方，`size_t`类型的`size`变量通过如下代码创建的：
```objc
size_t size = cls->instanceSize(extraBytes);
```
我们再去看`instanceSize()`函数的实现：
```c
    size_t instanceSize(size_t extraBytes) {
        size_t size = alignedInstanceSize() + extraBytes;
        // CF requires all objects be at least 16 bytes.
        if (size < 16) size = 16;
        return size;
    }
```
我们又一次看到了`alignedInstanceSize()`这个函数，我们上面说过，`NSObject`对象在调用这个函数时，分配的内存的大小时8，而`extraBytes`的大小为0。所以这个size就是8；那么通过下面的代码，我们将初始化的对象分配了16个字节的大小。这行代码的意思就是，凡是内存小于16个字节的对象，全部强制分配给16个字节。

也就是说，这个对象在创建的时候，系统给其分配了16个字节的存储空间，但是其成员变量只占用了8个字节。






