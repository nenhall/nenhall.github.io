---
title: Runtime
catalog: true
date: 2019-02-22 22:50:41
subtitle: __forwarding_ 底层实现解剖
header-img: note_head_img.jpg
catagories:
  - NSObject
tags:
 - 底层原理
 - Objective-C
---

# runtime

- 什么是Runtime？平时项目中有用过么？
  - OC是一门动态性比较强的编程语言，允许很多操作推迟到程序运行时再进行
  - OC的动态性就是由Runtime来支撑和实现的，Runtime是一套C语言的API，封装了很多动态性相关的函数
  - 平时编写的OC代码，底层都是转换成了Runtime API进行调用
- 具体应用
  - 利用关联对象（AssociatedObject）给分类添加属性
  - 遍历类的所有成员变量（修改textfield的占位文字颜色、字典转模型、自动归档解档）
  - 交换方法实现（交换系统的方法）
  - 利用消息转发机制解决方法找不到的异常问题

### 方法缓存

**缓存策略：方法调用后会存入方法缓存列表，以键值对形式存储，方法地址作为key，具体方法作为值，如果调用的是父类的方法，本类中没有缓存，调用时会缓存到自己的缓存方法列表中；**

**缓存方法查找：先从自己的缓存方法列表中查找，如果没有找到则去一层层的往父类查找，查找到同时会缓存到自己的缓存方法列表里面；**

## objc_msgSend执行流程

> objc_msgSend执行流程可以分为3大阶段:
>
> - 消息发送阶段
> - 动态方法解析阶段
> - 消息转发阶段

### 一、消息发送阶段：

> OC中的方法调用，其实都是转换为objc_msgSend函数的调用
>
> 消息发送阶段：消息接收者通过isa指针找到接收者的类对象，再从接收者的类对象中找到方法实现，找到方法后，还会缓存到方法缓存列表，方便下次调用
>
> **注：**如果消息接收者是类对象（调用的是类方法），那么就是：类对象通过父类的isa找到superClass(即元类对象)；类对象的类对象 == 元类对象

![](https://blogimage-1257063273.cos.ap-guangzhou.myqcloud.com/20190209184255.png)

### 二、动态方法解析阶段：

> 下图中进入消息发送流程时，如果在动态解析方法做了处理，找到了方法实现，那在调用方法实现的同时还会做方法缓存处理。

![](https://blogimage-1257063273.cos.ap-guangzhou.myqcloud.com/20190219225443.png)

```objective-c
// Method可以理解为等价于struct method_t *
// struct method_t *otherMethod = (struct method_t *)class_getInstanceMethod(self, @selector(other));
struct method_t {
    SEL sel;
    char *types;
    IMP imp;
}; 

void c_other(id self, SEL _cmd) {
    NSLog(@"c_other - %@ - %@", self, NSStringFromSelector(_cmd));
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(test)) {
        // 动态添加test方法的实现
        /** 写法一：*/
        // "v16@0:8" 解读：返回值为void，16为这个方法共占用16字节，
        // @代表第一个参数类型为id类型，0代表@参数从内存的第0个位置开始，:代表第二个参数类型为SEL，:号后面8代表:参数是从内存中的第8个位置开始
        class_addMethod(self, sel, (IMP)c_other, "v16@0:8");
        
        /** 写法二：*/
        // 获取其他方法
        Method method = class_getInstanceMethod(self, @selector(other));
        // 动态添加test方法的实现
        class_addMethod(self, sel,
                        method_getImplementation(method),
                        method_getTypeEncoding(method));
        
        // 返回YES代表有动态添加方法
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

如下图objc源码中可以看出：
会先判定方法是否动态解析过(triedResolver)，如果没有，则进入动态解析，如果解析过，则进入消息转发流程(_objc_msgForward_impcache)，并缓存方法到缓存方法列表中，便于下次调用；

![](https://blogimage-1257063273.cos.ap-guangzhou.myqcloud.com/20190219232606.png)

### 三、消息转发：

动态方法解析阶段没有做处理的时候，会进入消息转发阶段，

- 消息转发

  ```objective-c
  - (id)forwardingTargetForSelector:(SEL)aSelector
  {
      if (aSelector == @selector(test)) {
          /* 从本质上讲，其实也就是在调用这句代码：
          objc_msgSend([[Cat alloc] init], aSelector)
          **/
          return [[Cat alloc] init];
      }
      return [super forwardingTargetForSelector:aSelector];
  }
  ```

  如果在- (id)forwardingTargetForSelector:方法还是没有处理，那接下会来到下面的返回方法签名方法

  

- 返回方法签名：- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector

  只有在返回方法签名的方法中返回了正确的方法签名，否则不会进入下面的方法，一样崩溃

  ```objective-c
  // 方法签名：返回值类型、参数类型
  - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
  {
      if (aSelector == @selector(test)) {
          //以下两句代码是等价的，参数的占用位置可以省略不写
          return [NSMethodSignature signatureWithObjCTypes:"v20@0:8i16"];
          return [NSMethodSignature signatureWithObjCTypes:"v@:i"];
          //补充：使用这种方法直接返回也是可以，但前提是Cat对象需要实现对的方法
          return [[[Cat alloc] init] methodSignatureForSelector:aSelector];
      }
      return [super methodSignatureForSelector:aSelector];
  }
  ```

  返回了正常方法签名后，必须实现- (void)forwardInvocation:方法

  

- 方法转发处理：- (void)forwardInvocation:(NSInvocation *)anInvocation

  在这个方法中，我们就可以随意处理了，随意处理即：你可以什么事都不做，只实现这个方法就可以了

  ```objective-c
  /**
   NSInvocation封装了一个方法调用，包括：方法调用者、方法名、方法参数
   anInvocation.target 方法调用者
   anInvocation.selector 方法名
   [anInvocation getArgument:NULL atIndex:0]
   */
  - (void)forwardInvocation:(NSInvocation *)anInvocation
  {
  //    转发给Cat对象去处理，当然Cat需要实现了对应的方法，不然还是崩溃
  //    anInvocation.target = [[Cat alloc] init];
  //    [anInvocation invoke];
      //等价上面的代码
      [anInvocation invokeWithTarget:[[Cat alloc] init]];
      
      /* 获取第一个参数
       atIndex:2 因为第0位是消息接收者，第1位当前的selector，但也不能超出下标，否则崩溃
       */
      int value;
      [anInvocation getArgument:&value atIndex:2];
      NSLog(@"argument：%d", value);
  }
  ```

## 类对象的消息转发阶段

- 因为是类方法，所以是+号开头，但直接靠代码自动提示，是没有的，敲不出来的，方法名与实例方法名一样

```objective-c
+ (id)forwardingTargetForSelector:(SEL)aSelector {
    // objc_msgSend([[Cat alloc] init], @selector(test))
    if (aSelector == @selector(test)) {
        return [Cat class];
    }
    return [[Cat alloc] init];
    return [super forwardingTargetForSelector:aSelector];
}

+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(test)) return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    
    return [super methodSignatureForSelector:aSelector];
}

+ (void)forwardInvocation:(NSInvocation *)anInvocation {
    NSLog(@"abc");
}
```

### 补充内容：从objc的本质去看消息转发实现

- 先来写一段测试代码：

  - 条件1：方法调用者是类对象，调用类方法；
  - 条件2：在`+forwardingTargetForSelector`中返回一个实例对象(正常情况应该是返回一个类对象);
  - 条件3：在这个类里面对应的类方法和实例方法都实现，看运行后程序级否调用成功；

  分别有两个类：Cat、Person，然后Person在main函数中调用test方法，Person实现中进行转发

  ```objective-c
  // Cat
  @interface Cat : NSObject
  + (void)test;
  - (void)test;
  @end
  
  @implementation Cat
  + (void)test {
      NSLog(@"%s", __func__);
  }
  - (void)test {
      NSLog(@"%s", __func__);
  }
  @end
      
  // Person
  @interface Person : NSObject
  + (void)test;
  @end
  
  @implementation Person
  + (id)forwardingTargetForSelector:(SEL)aSelector {
      if (aSelector == @selector(test)) return [[Cat alloc] init];
      return [super forwardingTargetForSelector:aSelector];
  }
  @end
  
  // main.m
  int main(int argc, const char * argv[]) {
      @autoreleasepool {
         //调用Person类对象的test方法
          [Person test];
      }
      return 0;
  }
  ```

  运行后的打印结果是：

  注意：没有崩溃，并成功的调用了-开头的test方法，为什么会这样，我们继续往下看，等下解答

  ```
  2018-02-21 22:33:51.863918+0800 消息转发[4731:536691] -[Cat test]
  ```

- `__forwarding__`这一部份的实现是不开源的，只能我们自己去摸索猜测，但有国外的开发者已经按苹果objc源码分析出了`__forwarding__`的内部实现，虽然不能保证是一模一样的，但大致的流程是一致的，如下图是来自国外开发者对其实现的分析的**伪代码**：

  - 在调用`(id)forwardingTargetForSelector:(SEL)aSelector`方法的不管是实例对象对还是类对象，最终都是在调用`objc_msgSend(…)`，所以刚在的现象也就很好的解释了：

    **它的内部不会关心是类还是实例，只关心是谁是消息接收者，消息接收者有没有实现对应的方法。**

  ![](https://blogimage-1257063273.cos.ap-guangzhou.myqcloud.com/20190221214800.png)

###  讲一下 OC 的消息机制

- OC中的方法调用其实都是转成了objc_msgSend函数的调用，给receiver（方法调用者）发送了一条消息（selector方法名）
- objc_msgSend底层有3大阶段
  - 消息发送（缓存列表查找、方法数组查找、父类查找）
  - 动态方法解析
  - 消息转发