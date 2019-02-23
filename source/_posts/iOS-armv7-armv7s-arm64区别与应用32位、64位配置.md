---
title: 'iOS armv7, armv7s, arm64区别与应用32位、64位配置'
catalog: true
date: 2018-08-29 17:18:39
subtitle: 详细了解手机架构
header-img: note_head_img.jpg
catagories:
  - Objective-C
tags:
  - NSObject
  - 底层原理
---
## iOS armv7, armv7s, arm64区别与应用32位、64位配置
### 历程

1. 2015年2月1日不允许不支持arm64的应用的提交；
2. iOS10发布后，如果你的应用中使用到了32位的类、库，没有相应支持64位机型的库，则首次启动后会弹窗式提示：“ xxx可能使iPhone变慢...”；
3. iOS10.3公测版发布:封杀32位应用；
4. 苹果系统（iOS7/iOS8-iOS9）,32位的framework和64位的framework是共存的，所以如果所有的应用都是64位的话，系统就自动不加载32位的framework，这样可以少占用资源，另外对运行的速度是有好处的，也就是说，你会感觉手机运行的快了一点。应用在兼容64位系统后，内存的占用肯定会变多一点，不过性能也有相应的提升。

#### 下面了解下64与32位的区别：

##### 数据类型的变化：

1. 在32位的年代，使用的是IPL32的规范，到了64位之后，改成了LP64规范。
2. 数据类型里面，NSInteger在32位时等同于int，在64位时等同于long，而这个数据结构使用很广，很多不规范的时候会直接和int替换使用，在32位是毫无问题，但在64位时，这就是隐患了。CGFloat也有同样的问题
3. 如果使用了偏移量来访问struct的项，那么需要认真仔细的检查，其余的还算好，当然如果你用了malloc，那么也请检查一下分配的内存大小，建议是多使用sizeof来帮助计算。
4. 方法调用上的变化：在64位系统在运行时调用函数和32位系统是不同的。主要的区别在于传递具有可变参数个数的函数的参数时，我们来看下面的代码：
    ```
    int fixedFunction(int a, int b);  
    int variadicFunction(int a, ...); 
       int main  
    {  
    int value2 = fixedFunction(5,5);  
    int value1 = variadicFunction(5,5);  
    }
    ```
    第一个函数是固定传入2个参数，第二个函数是参数个数不定的，在使用中,也
    传入了2个参数。在32位系统下，这两个函数的参数传递是非常类似的，在64
    位系统下，这两者就是截然不同了。
5. 项目中使用的第三方库肯定需要支持64位系统，否则还是白搭。大家需要检查自己使用的第三方的库，看是否支持64位的版本。苹果系统中（iOS7/iOS8-iOS9）,32位的framework和64位的framework是共存的，所以如果所有的应用都是64位的话，系统就自动不加载32位的framework，这样可以少占用资源，另外对运行的速度是有好处的，也就是说，你会感觉手机运行的快了一点。应用在兼容64位系统后，内存的占用肯定会变多一点，不过性能也有相应的提升。

#### 增加应用对64位的支持：

##### 一、配置前确认工作：

如果你不能确定库是否支持了arm64，可以在cmd模式下用file命令来检查一下库文件：

```
$lipo -info /Users/xxx/NHPushStreamSDK/FFmpeg-iOS/lib/libavcodec.a
//指令说明：lipo -info 静态库(动态库)文件完整路径
```

armv7，armv7s和arm64,i386,x86_64这些都代表什么？
* armv7｜armv7s｜arm64都是ARM处理器的指令集
* i386｜x86_64 是Mac处理器的指令集
  <br>
###### 了解ARM
__Arm处理器，因为其低功耗和小尺寸而闻名，几乎所有的手机处理器都基于arm，其在嵌入式系统中的应用非常广泛，它的性能在同等功耗产品中也很出色。

Armv6、armv7、armv7s、arm64都是arm处理器的指令集，所有指令集原则上都是向下兼容的，如iPhone4S的CPU默认指令集为armv7指令集，但它同时也兼容armv6指令集，只是使用armv6指令集时无法充分发挥其性能，即无法使用armv7指令集中的新特性，同理，iPhone5的处理器标配armv7s指令集，同时也支持armv7指令集，只是无法进行相关的性能优化，从而导致程序的执行效率没那么高。
<br>

> 这些指令集在哪些设备中有用到呢？

| arm64     | armv7s    | armv7               |
| ------------------------------------------ | ------------------------------- | ------------------- |
| iPhone6s        | iPhone5      | iPhone4             |
| iphone6s plus          | iPhone5C                        | iPhone4S            |
| iphone6           | iPad4(iPad with Retina Display) | iPad                |
| iphone6 plus       |       | iPad2               |
| iphone5s       |        | iPad3(The New iPad) |
| iPad Air         |      | iPad mini        |
| iPad mini2 (iPad mini with Retina Display) |         | iPod Touch 3G       |
|           |            | iPod Touch4         |
>i386是针对intel通用微处理器32位处理器
>x86_64是针对x86架构的64位处理器

>模拟器32位处理器测试需要i386架构，
>模拟器64位处理器测试需要x86_64架构，
>真机32位处理器需要armv7,或者armv7s架构，
>真机64位处理器需要arm64架构。


###### Xcode Build Setting中指令集相关选项释义
1. Architectures
   指定工程被编译成可支持哪些指令集类型，而支持的指令集越多，就会编译出包含多个指令集代码的数据包，对应生成二进制包就越大，也就是ipa包会变大(Space-separated list of identifiers. Specifies the architectures (ABIs, processor models) to which the binary is targeted. When this build setting specifies more than one architecture, the generated binary may contain object code for each of the specified architectures. )。

2. Valid Architectures
   限制可能被支持的指令集的范围，也就是Xcode编译出来的二进制包类型最终从这些类型产生，而编译出哪种指令集的包，将由Architectures与Valid Architectures（因此这个不能为空）的交集来确定(Space-separated list of identifiers. Specifies the architectures for which the binary may be built. During the build, this list is intersected with the value of ARCHS build setting; the resulting list specifies the architectures the binary can run on. If the resulting architecture list is empty, the target generates no binary.)。

3. Build Active Architecture Only
   指定是否只对当前连接设备所支持的指令集编译
    当其值设置为YES，这个属性设置为yes，是为了debug的时候编译速度更快，它只编译当前的architecture版本，而设置为no时，会编译所有的版本。 所以，一般debug的时候可以选择设置为yes，release的时候要改为no，以适应不同设备。
    <br>
    <br>
##### 二、工程配置，增加64位的支持
> note：在Xcode6.1.1及以上 Valid Architectures  设置里， 默认为 Standard architectures(armv7,arm64),如果你想改的话，自己在other中更改。

* 使用 standard architectures (including 64-bit)(armv7,arm64) 参数，则打的包里面有32位、64位两份代码，在iPhone5s（ iPhone5s的cpu是64位的 ）下，会首选运行64位代码包， 其余的iPhone（ 其余iPhone都是32位的,iPhone5c也是32位 ），只能运行32位包，但是包含两种架构的代码包，只有运行在ios6，ios7系统上。 这也就是说，这种打包方式，对手机几乎没要求，但是对系统有要求，即ios6以上。 
* 使用 standard architectures (armv7,armv7s) 参数， 则打的包里只有32位代码， iPhone5s的cpu是64位，但是可以兼容32位代码，即可以运行32位代码。但是这会降低iPhone5s的性能。 其余的iPhone对32位代码包更没问题， 而32位代码包，对系统也几乎也没什么限制。 
* 要发挥iPhone5s及以上的64位机型性能，就要包含64位包，那么系统最低要求为ios6。 如果要兼容ios5以及更低的系统，只能打32位的包，系统都能通用，但是会丧失iPhone5s的性能。
  <br>
#### Valid Architectures与Architectures指令集交集表：
| Valid Architectures设置 | Architectures设置    | 生成二进制包支持的指令集(取交集) |
| ----------------------- | -------------------- | ------------------------------------------- |
| armv7, armv7s, arm64  |  armv7s   | armv7s  |
| armv7, armv7s  | armv7s,arm64  |  armv7s   |
| armv7, armv7s, arm64  |  armv6, armv7s, arm64 | armv7s, arm64                                      |
| armv6, armv7, armv7s   |  armv6, armv7s, arm64 | armv6, armv7s                               |
| armv7, arm64 ,armv7s   | armv7, arm64 ,armv7s  | armv7, arm64 ,armv7s |
| armv7, armv7s, arm64  |  armv7，armv7s  | armv7s, armv7 |

>1. 比如 Valid Architectures设置的支持arm指令集版本有：armv7/armv7s/arm64，对应的Architectures设置的支持arm指令集版本有：armv7s，这时Xcode只会生成一个armv7s指令集的二进制包。
> 2. xcode默认的设置：
        **Architectures** : `Strandard architectures - $(ARCHS_STANDARD)` //相当于arm64 armv7s armv7
       **Valid Architectures** : `arm64 armv7s armv7`
>3. 如果你对ipa安装包大小有要求，可以减少安装包的指令集的数量，这样就可以尽可能的减少包的大小。

![自定设置](https://upload-images.jianshu.io/upload_images/2443108-7013ad449e239282.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![系统默认设置](https://upload-images.jianshu.io/upload_images/2443108-67cf1a85da78e2d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br>
#### 制作静态库.a是指令集选择
现在回归到正题，如何制作一个“没有问题”的.a静态库，通过以上信息了解到，当我们做App的时候，为了追求高效率，并且减小包的大小，Build Active Architecture Only设置成YES，Architectures按Xcode默认配置就可以，因为arm64向前兼容。但制作.a静态库就不同了，因为要保证兼容性，包括不同iOS设备以及模拟器运行不出错，所以结合当前行业情况，要做到最大的兼容性。
ValidArchitectures设置为：armv7｜armv7s｜arm64｜i386｜x86_64 
Architectures设置不变（或根据你需要）:  armv7｜arm64
然后分别选择iOS设备和模拟器进行编译，最后找到相关的.a进行合包:
* 合并两个不同构架的库文件
```
$ lipo -create /Users/xxxxxx/lib/libv7.a  /Users/xxxxxx/lib/lib64.a  -output /Users/xxxxxx/lib/lib.a
//指令说明： lipo -create 库文件1一完整路径  库文件2完整路径  -output 输出的文件名路径
```
* 从某个库文件中提起出指定构架的库文件
```
Desktop$ lipo -thin arm64 debugserver -output debugserver2
//指令说明：lipo -thin 对应构架指令 对应文件 -output 输出的文件名路径
```
使用lipo -create 真机库.a的路径 模拟器库.a的的路径 －output 合成库的名字.a ，这样就制作了一个通用的静态库.a。
制作动态、静态库详情可以参考[【链接】](http://blog.csdn.NET/lizhongfu2013/article/details/12648633)


<br>
#### 其它注意点：
1. 因此如果一些程序中使用的静态库不支持armv7s，而你的工程支持armv7s时，就会出现`“xxxx does not contain a(n) armv7s slice:xxxxx for architecture armv7s"`的编译错误，想要解决这个问题，有两个方法：
    1. 如果是开源的，能够找到源代码，则可以用源代码重新打一个支持armv7s的libaray, 或者在工程中直接使用源代码，而不是静态库。
    2. 如果不是开源的，要么就坐等第三方库的支持，要么就暂时让你的工程不支持armv7s。
2. 此外，模拟器并不运行arm代码，软件会被编译成x86可以运行的指令。所以生成静态库时都是会先生成两个.a，一个是i386的用于在模拟器运行，另一个是在真实设备上运行的，然后再用命令将两个.a合并成一个。
3. 需要注意的是iOS模拟器没有运行arm指令集，编译运行的是x86指令集，所以，只有在iOS设备上，才会执行设备对应的arm指令集。
4. 连接的手机指令集匹配是由高到低（arm64 > armv7s > armv7）依次匹配的。


[简书链接](https://www.jianshu.com/p/567d3b730608)

部份资源[链接](
http://blog.csdn.net/sakulafly/article/details/41089411)