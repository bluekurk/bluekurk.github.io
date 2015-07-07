# Effective Objective-C 2.0

### 1. 了解历史
兼容c，c++
### 2. 尽量避免头文件包含头文件

前置声明；@import module; pch 文件

### 3. 多用literal
@[@1, @2, @3],  @{@"Matt":@"firstName"}

### 4. 少用预定义，多用类型常量
预定义#define无类型，副作用问题多；尽量用 static const，external等方式取代

### 5. 用枚举
可以继承类型了

	enum EnumType : NSInteger;

NS_ENUM 传统的，累加型

NS_OPTIONS  位运算类型

这两个 Macro 向后兼容旧式写法（C++）

### 6. 理解property
* 自动合成的成员变量，影响到内存布局，类大小，二进制兼容
* atomic/nonatomic
* readonly/readwrite
* assign/strong/weak/copy/unsafe_unretained
* getter/setter

### 7. 内部尽量直接访问实例变量(存疑)
考虑要点

* 直接访问ivar速度快，但普通情况下可以忽略（除非密集计算等）
* copy语意以及MRC下的内存语意被忽略
* KVO不会被触发
* 调试时无法在@property处或者setter处下断点

当然，在init和dealloc里应该尽量直接访问ivar，避免子类重载以及类信息被破坏self不完整等问题。

而在延迟初始化/Lazy Initialization里必须使用getter方法。否则无法初始化。

### 8. 理解对象相等
* isEqual 的 前提是 hash 相等
* 不同对象如果 hash 一样，在collection里会出问题
* hash最好是轻量级函数，否则在collection里会有性能问题。
* isEqualToString 针对 isEqual 优化，其他类似。

### 9. class cluster
 NSArray 跟 NSMutableArray 的class对象是一样的

### 10. Associated Object
为一个已有的class添加成员变量

### 11. objc_msgSend
消息传递
函数原型

	id objc_msgSend(id self, SEL cmd, ...)
	
同时还有辅助函数 objc_msgSend_stret(返回结构体), objc_msgSend_fpret(返回浮点数)

objc_msgSendSuper(以及另两个返回值版本)可以和超类发消息

所有的 Objective-C 对象的每个方法都可以类比为

	<return_type> Class_selector(id self, SEL _cmd, ...)

这样的结构方便尾递归优化，避免栈溢。

### 12. 理解消息转发机制
message forwarding

1) dynamic method resolution

2) full forwarding mechanism

第一步寻找有没有指定的类，它能否添加新的方法来响应。如果失败则走第二步。具体方法有

	+(BOOL) resolveInstanceMethod:(SEL) selector
    
和类似的 resolveClassMethod, 在里面添加支持函数

第二步，先让接收者查看是否有其他对象能处理这条信息；如果还是没有，则启动full forwarding mechanism, 把详细的信息都封装到 NSInovaction 里，再给接收者最后一次机会，让它来处理这个message。

在下面的方法里检查有没有 replacement receiver

	-(id) forwardingTargetForSelector:(SEL)selector

最后的机会

	-(id) forwardInvocation:(NSInvocation*)invocation

这三步，越往后代价越高

### 13. Method Swizzling

动态插入自己想要执行的函数。一般在 +(void)load 或者 + (void)initialize 里做实现，交换系统或者库函数和自己实现的函数。各种实例，不赘述。

### 14. 类对象
注意*isa 和 *super_class, 分别指向meta class 和 super class。当然最后会在 NSObject 上打圈圈。

* isKindOfClass 是否为某个类或者派生类
* isMemberOfClass 是否为某个特定类的实例

注意 class cluster的问题：

	    NSArray * e1 = [NSArray new];
	    NSArray * e2 = [NSMutableArray new];
    	BOOL b1 = [e1 isKindOfClass:NSArray.class];
	    BOOL b2 = [e2 isKindOfClass:NSMutableArray.class];
	    BOOL b3 = [e1 isMemberOfClass:NSArray.class];
	    BOOL b4 = [e2 isMemberOfClass:NSMutableArray.class];

结果是(测试环境Xcode 6.3.2, iOS simulator iPhone 5, iOS 8.3)

	    (__NSArrayI *) e1 = 0x7986d6a0 @"0 objects"
	    (__NSArrayM *) e2 = 0x7986e930 @"0 objects"
    	(BOOL) b1 = YES
	    (BOOL) b2 = YES
	    (BOOL) b3 = NO
	    (BOOL) b4 = NO

NSDictionary 和 NSMutableDictionary一样。

另外注意 NSProxy 的实例调用class方法和调用动态查询返回的结果不一致，后者能返回"真正"的干活的类型；前者返回 NSProxy。

### 15. 用前缀避免命名空间污染
* Apple 保留了两字前缀的权利，所以尽量三个(大写)字前缀
* 小心 C 类全局函数
* 小心第三方库的相互包含以及冲突(libA 包含v1的libC， libB包含v2的libC)

### 16. 提供 designated initializer
* 一个类里的多个初始化函数应当指定一个 designated initializer, 其他初始化函数调用它来实现
* 子类重载实现新的 designated initializer， 必须重载父类里的对应方法。
* 如果父类的初始化方法不适用于子类，则应该重载并在里面抛出异常。

### 17. description方法
NSLog 用 %@ 输出的实例，是 [NSObject description]的结果，所以适当地实现它。可以用NSDictionary格式方便整理格式。
而 po 的结果，是 [NSObject debugDescription]，往往就是简单调用[NSObject description]

### 18. 尽量使用不可变对象
如果把可变对象放入collection再作修改，则会破坏里面的结构(比如set会包含两个相同元素)

如果希望某个属性只允许内部修改，可以用 class extension 重新声明为 readwrite

不要吧可变的 collection 作为属性公开，应该对它封装。

### 19. 使用清晰的命名
就是有时会冗长点
### 20. 为私有方法添加前缀
* 区分公开方法和私有方法。修改公开方法的代价很高。
* 不要简单地在前面添加下划线，可能 Apple 会占用

### 21. 理解 Objective-C 错误类型
* ARC 不保证异常安全。如果需要，使用 -fobjc-arc-exceptions 标志编译。总的来说，并不推荐使用异常
* 更偏向于用 NSError， error domain/error code/user info来报告，类似c
* 注意传入的NSError ** 的内存语意是  * __autoreleasing *

### 22. 理解 NSCopying 协议
留意 - copy, - mutableCopy, -immutableCopy 的区别；注意浅拷贝和深拷贝的使用场景。


### 23. delegate 和 datasource
理解 protocol

### 24. 将类的实现代码分部到便于管理的多个category里去
* 调试时方法的 symbol name 包含 category 名字
* 需要隐藏细节时创建 private category

### 25. 总是为第三方category 的名称添加前缀
### 26. category里不要声明property
即使用 associated object 实现，也得留意内存语意等细节。不建议这么做。
### 27. 用class extention 隐藏实现细节
* 比如引入c++时，避免了一堆文件都要声明class 关键字然后统统用.mm文件实现
* 可以声明类变量和 property
* 可以覆盖 property 的 部分声明(readonly 改成 readwrite)
* 可以声明遵从某些 protocol
### 28. 通过 protocol 提供 anomyous object
用protocol避免继承基类。这种方式类似于 c++ 的抽象基类，轻量。

比如 NSDictionary的set方法，key部分要求是 id<NSCopying> , 支持copy。
### 29. 理解引用计数
### 30. ARC
### 31. (ARC下)dealloc只释放引用并解除监听
建议实现 -cleanup 函数，在调用 dealloc之前就释放大部分资源。
### 32. 异常安全时注意内存管理
典型的是在finally里保证释放资源。然而ARC下并不手动释放，所以问题更大
### 33. 用 weak 避免 retain cycle
delegate一般声明为weak

以及weakself(某些情况下可以使用__block变量)
### 34. 用autoreleasing pool降低内存峰值
### 35. 用Zombie Object调试内存问题
* 内存多释放一次: crash 用zombie调试。
* 内存少释放一次: leak

### 36. 不要使用retainCount
### 37. 理解block
* 捕获
### 38. 为常用的block创建typedef
### 39. 用handler block降低代码分散程度
比如网络请求，需要处理 didReceiveData， didError， didFinish等事件，可以分别挂block，也可以挂一个block整体都做处理。
### 40. block里小心形成循环引用
* weakself
### 41. 多用dispatch queue， 少用同步锁
@synchronization, NSLock,等等减弱性能
合理安排dispatch queue避免竞态条件，死锁。
比如dispatch_barrier_sync用于写同步
### 42. 多用GCD，少用performSelector
* performSelector 参数受限，特别时延迟和在主线程上的操作
* 函数命名时的alloc等内存语义会被忽略
* 返回对象只能是 id
### 43. 比较GCD很NSOperationQueue的使用时机
NSOperationQueue 除了重一些，优势有:
* 取消某个操作
* 指定操作的依赖关系
* 通过 KVO 监控 NSOperation 对象的属性
* 指定操作优先级
* 重用 NSOperation 对象，继承之
### 44. Dispatch Group
任务分组，可以并发也可以序列操作，完成后可以带个notify block操作

dispatch_apply 展开循环
### 45. dispatch_once执行只需运行一次的线程安全代码
典型的如实现 singleton
### 46. 不要使用dispatch_get_current_queue
单独判断这个避免不了死锁。比如queueA在queueB里，串行依赖，这俩会互相死锁，在queueB里判断不在queueA里没有作用。解决办法是dispatch_queue_set_specific设置标志后检查。
### 47. 熟悉系统框架
* Foundation

  基础数据结构， NS开头
* CoreFoundation

  配合Foundation，对象桥接出来。大量有用的库AVFoundation, CoreGraphics, CoreData, 等等

### 48. 多用块枚举，少用for循环
快速枚举， 特别是NSDictionary同时给出key和value
### 49. 对自定义其内存语义的collection使用无缝桥接
* bridge, bridge retain, bridge transfer
* CFRelease, CFRetain
### 50. 构建时使用NSCache而不是NSDictionary
* NSCache自动在lowmemory warning时释放资源
* NSCache线程安全，而且不会拷贝key
* NSCache的上限只是建议
* NSPurgableData 和 NSCache搭配使用可实现自动清除功能
* 挑选那些值得缓存的数据
### 51. 精简initialize和load的实现
* 注意不要产生循环依赖
* load是在第一次把class(subclass)或者category加载到runtime时调用(细节见文档)
* initialize是class(或subclass)第一次接收到message时调用的。仅调用一次，但如果子类没有实现，会调用父类的(实质上的多次)。线程安全。(细节见文档)
### 52. NSTimer会retain target
如果使用了self作为target，一定要合理地调用invalidate，否则循环引用导致dealloc无法调用，泄露。




