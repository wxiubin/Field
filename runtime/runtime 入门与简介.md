### 方法调用 objc_msgSend
调用方法（函数）是语言经常使用的功能，在 Objective-C 中专业一点的叫法是 **传递消息(pass a message)**。Objective-C 的方法调用都是 **动态绑定** ，而C语言中函数调用方式是 **静态绑定**  ( **static binding** )，也就是说，在编译时期就能决定和知道在运行时所调用的函数。

以下面代码为例：

``` C
void sayHello(){
}
void sayGoodBye(){
}
void saySomething(int type){
	if(type == 0){
		sayHello();
	}else{
		sayGoodBye();
	}
}
```

基本上，上面的代码在编译的时候编译器就知道 **sayHello** 和 **sayGoodBye** 两个函数的存在，函数地址是硬编码在指令之中的。但是如果换一种写法：

``` C
void sayHello(){
}
void sayGoodBye(){
}
void saySomething(int type){
	void (*something) ();
	if(type == 0){
		something = sayHello;
	}else{
		something = sayGoodBye;
	}
	something();
}
```

这就得使用 **动态绑定** ，待调用的函数地址需要到运行时才能读取出来。
在 Objective-C 中，对某一个对象传递消息，会用动态绑定机制来决定到底是调用哪个方法。而Objective-C是 C 的超集，底层是由 C语言实现，但是对象接收消息后会调用哪个方法都是在运行期决定。

给对象发送消息可以这么来写：

``` objc
id object = [list objectAtIndex:1];
```

在这行代码中， **list** 称为 **接收者**， **objectAtIndex** 叫做 **选择器**， 选择器和参数合起来称为**消息**。当编译器看到这行代码的时候，会换成标准的C语言函数调用：

``` C
void objc_msgSend(id self, SEL cmd, ...);
id lastObject = objc_msgSend(list, @selector(objectAtIndex:), parameter);
```

**objc_msgSend** 这个函数可以接收两个及两个以上的参数，第一个参数是接收者，第二个参数是选择器，后面的参数是保持顺序的原来消息传递的参数，**objc_msgSend**会依据接收者和选择器来决定调用哪个方法，首先在接收者的方法列表中寻找，如果找不到就会沿着继承体系去向上一层一层的寻找，如果仍旧找不到就会执行**消息转发(message forwarding)** 。
当消息第一次传递之后，objc_msgSend 会将匹配结果进行缓存，下次会直接调用方法。消息传递除了objc_msgSend之外在特殊情况下还会有其他的方法来处理：

- **objc_msgSend_stret** 如果待发送的消息返回一个结构体，就会调用这个函数来处理。
- **objc_msgSend_fpret** 如果消息返回的是浮点数，就会调用这个函数进行处理。
- **objc_msgSendSuper** 如果要传递消息给父类。

**总结：**
-  消息由 接收者、选择器及参数构成，给某对象 **发送消息( invoke a message )** 也就相当于在该对象上调用方法。
-  发送给某对象的全部消息都要有**动态消息派发系统( dynamic message dispatch system )** 来处理。

### 消息转发
在上面介绍了运行时的消息传递机制，但是却没有说对象收到消息却无法解读该怎么办。本篇博客就着重介绍当消息传递时无法解读的时候就会启动的 **消息转发机制( message forwarding )**。


开发可能经常会遇到这种情况：

``` objc
2016-04-20 13:14:07.391 runtime[1096:22076] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[AutoDictionary setDate:]: unrecognized selector sent to instance 0x100302f50'
*** First throw call stack:
(
	0   CoreFoundation                      0x00007fff9f2d94f2 __exceptionPreprocess + 178
	1   libobjc.A.dylib                     0x00007fff90db3f7e objc_exception_throw + 48
	2   CoreFoundation                      0x00007fff9f3431ad -[NSObject(NSObject) doesNotRecognizeSelector:] + 205
	3   CoreFoundation                      0x00007fff9f249571 ___forwarding___ + 1009
	4   CoreFoundation                      0x00007fff9f2490f8 _CF_forwarding_prep_0 + 120
	5   runtime                             0x0000000100001c1c main + 124
	6   libdyld.dylib                       0x00007fff91df85ad start + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
```

这个异常信息是由 **NSObject** 的 **doesNotRecognizeSelector:** 方法抛出来的，本来是给 **AutoDictionary** 的一个实例对象发送消息，但是该对象并没有 **setDate:** 方法，所以消息转发给了 **NSObject** ，最后抛出异常。

先看下消息处理机制流程图：

![消息处理机制流程图](http://upload-images.jianshu.io/upload_images/1275366-7bcf93059836e5af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

消息转发分为两阶段三步，第一阶段先看接受消息的对象能不能自己处理这个无法解读的消息，这一步可以动态的添加方法去解读接受这个消息；第二阶段是先看看对象自己不能处理这个消息，能不能交给其他对象来进行处理，在这一步如果仍然无法解读消息，那么就会走最后一步：把和消息有关的所有细节封装到一个 **NSInvocation** 中，再询问一次对象是否能解决。
看下三个方法：

``` objc
// 询问对象是否自己处理，是返回YES，一般会在这个方法里面动态添加方法
+ (BOOL)resolveInstanceMethod:(SEL)sel;

// 这一步询问对象把消息交给哪个对象来进行处理
- (id)forwardingTargetForSelector:(SEL)aSelector;

// 如果走到这一步的话，就把消息的所有信息封装成 NSInvocation 对象进行 "最后通牒"
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

来一段代码示例：
新建一个 **AutoDictionary** 类，添加一个 **NSDate** 类型的 **date** 属性，在实现文件里面用 **@dynamic date;** 禁止自动生成存取方法，这样当代码中给 **AutoDictionary** 实例对象的 **date**属性赋值时就会出现消息无法解读的现象。
**.h** 文件：

``` objc
@interface AutoDictionary : NSObject

@property (nonatomic, strong) NSDate *date;

@end
```
**.m** 实现文件代码内容：

``` objc
@interface AutoDictionary()
@property (nonatomic, strong) NSMutableDictionary *backingStore;

/**
 *  该类仅在实现文件 实现了
 *  - (NSDate *)date
 *  - (void)setDate:(NSDate *)date
 *  两个方法，用于处理 AutoDictionary 无法解读的消息
 */
@property (nonatomic, strong) MethodCreator *methodCreator;
@end
@implementation AutoDictionary

@dynamic date;

- (instancetype)init{
    if (self = [super init]) {
        self.backingStore = [NSMutableDictionary dictionary];
        self.methodCreator = [MethodCreator new];
    }
    return self;
}

#pragma mark - 消息转发机制 ：1.动态添加方法 2.后备消息接收者 3.封装NSInvocation，最后通牒
// 3. 封装NSInvocation，最后通牒
- (void)forwardInvocation:(NSInvocation *)anInvocation{
    
}
// 2. 无法接受消息，选择由谁来接受
- (id)forwardingTargetForSelector:(SEL)aSelector{
    return self.methodCreator;
}
// 1. 动态添加方法
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    NSString *selString = NSStringFromSelector(sel);
    
    if ([selString hasPrefix:@"set"]) {
        class_addMethod(self, sel, (IMP)autoDictSetter, "");
    }else{
        class_addMethod(self, sel, (IMP)autoDictGetter, "");
    }
    
    return YES;
}

id autoDictGetter (id self, SEL _cmd){
    
    AutoDictionary *dict = self;
    NSString *key = NSStringFromSelector(_cmd);
    return [dict.backingStore objectForKey:key];
}

void autoDictSetter (id self, SEL _cmd, id value){
    
    AutoDictionary *dict = self;
    
    NSString *selString = NSStringFromSelector(_cmd);
    
    NSString *key = [selString substringWithRange:NSMakeRange(3, selString.length-4)];
    
    key = [key lowercaseStringWithLocale:[NSLocale currentLocale]];
    
    if (value) {
        [dict.backingStore setObject:value forKey:key];
    }else{
        [dict.backingStore removeObjectForKey:key];
    }
}

@end
```
测试代码：

``` objc
AutoDictionary *dict = [AutoDictionary new];
dict.date = [NSDate date];
NSLog(@"dict.date = %@",dict.date);
```

### 给对象、分类添加实例变量

在开发中有时候想给对象实例添加个变量来存储数据，但又无法直接声明，比如说既有类的分类。这个时候我们就可以通过 **关联对象** 在运行时给对象关联一个 **对象** 来存储数据。（注意：并不是真实的添加了一个实例变量）

**关联对象** 可以给某个对象关联其他对象并用**key**来区分其他对象。需要注意的是，存储对象的时候要指明 **存储策略**，用来维护对象的内存管理语义。存储策略是 **objc_AssociationPolicy** 枚举定义，以下是存储策略对应的 **@property**属性：


| 存储策略类型 | 对应的@property属性 |
| ------------ | ------------ |
| OBJC_ASSOCIATION_ASSIGN | weak |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | strong, nonatomic |
| OBJC_ASSOCIATION_COPY_NONATOMIC | copy, nonatomic |
| OBJC_ASSOCIATION_RETAIN | strong |
| OBJC_ASSOCIATION_COPY | copy |

用下面的方法可以管理关联对象：

``` objc
// 这个方法可以根据指定策略给对象关联对象值
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)

// 这个方法可以获取对象关联对象值
id objc_getAssociatedObject(id object, const void *key)

// 这个方法可以删除指定对象的全部关联对象值
void objc_removeAssociatedObjects(id object)
```

对于关联对象这个OC特性，我们可以把对象想象成一个 NSDictionary，关联对象需要一个 **key**( 类型是 opaque pointer，无类型的指针 ) 来区分，我们可以把要添加的变量名作为 **key** ，把变量的值作为关联的对象来存储到 ”对象“ 这个 NSDictionary 中。
所以，关联对象的

``` objc
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```

方法类似于字典的

``` objc
[dict setObject: forKey:]
```
方法。

在存储和获取关联对象时需要用一个相等的 **key** ，因为是给 Class 的实例对象关联对象，所以一般用静态变量来做 **key** 。

说的再多，不如上段代码！

比如说，我们给 **NSString** 实例加上个 **NSDate** 类型的 **date** 变量。什么？给字符串加个日期变量是要干袅？我要给字符串过个生日不行吗！ 别闹，举个栗子嘛！（捂脸逃跑~~~）

首先，我们先给 **NSString** 新建个名为 **RT** 的 category。
在头文件中有个 NSDate 类型的 date 属性：

``` objc
//  NSString+RT.h
//  runtime
#import <Foundation/Foundation.h>

@interface NSString (RT)

@property (nonatomic, strong) NSDate *date;

@end
```

在分类中的属性只会生成 **get** 和 **set** 方法，并不会生成变量。
所以我们需要重写 **get** 和 **set** 方法，关联对象以变相实现添加变量，在现实文件中：

``` objc
//  NSString+RT.m
//  runtime
#import <objc/runtime.h>
#import "NSString+RT.h"

@implementation NSString (RT)

static void *runtime_date_key = "date";
- (NSDate *)date{
    return objc_getAssociatedObject(self, runtime_date_key);
}

- (void)setDate:(NSDate *)date{
    objc_setAssociatedObject(self, runtime_date_key, date, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
```
需要注意的是，关联对象用到的 **key** 是个无类型的指针，一般来说是静态来修饰。
另外，给对象关联的只能是对象，如果是 **int**、 **float** 等类型需要 **NSNumber** 进行包装。
因为 **date** 是强引用和非原子属性，所以关联策略用  **OBJC_ASSOCIATION_RETAIN_NONATOMIC**

然后执行代码：

``` objc
NSString *string = @"runtimeTestString";
string.date = [NSDate date];
NSLog(@"string.date = %@",string.date);
```

输出结果：

``` objc
2016-04-12 21:27:31.099 runtime[2837:103727] string.date = 2016-04-12 13:27:31 +0000
```

**注意：**
- 定义关联对象时需要指定内存管理语义，用来模拟对象对变量的拥有关系
- 尽量避免使用关联对象，因为如果出现bug不易于问题排查

#### iOS 开发中的 AOP
在 **Objective-C** 中，类的方法列表会把选择器的名称映射到方法的实现上，这样 **动态消息转发系统** 就可以以此找到需要调用的方法。这些方法是以函数指针的形式来表示，这种指针叫做 **IMP**。
如下：

![](http://upload-images.jianshu.io/upload_images/1275366-25408cf5248e6ee7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

``` objc
id (*IMP) (id, SEL, ...)
```

**Objective-C** 的 runtime 机制以此提供了获取和交换映射**IMP**的的接口：

``` objc
// 获取方法
Method class_getInstanceMethod(Class cls, SEL name)；

// 交换两个方法
void method_exchangeImplementations(Method m1, Method m2)
```

我们可以通过上面两个方法来进行选择器和所映射的**IMP**进行交换：

![](http://upload-images.jianshu.io/upload_images/1275366-d2e83dd7333b21b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

来，直接上代码示例，比如我们的要实现功能是在每个控制器的**viewDidLoad**方法里面log一下，一般有三种实现方式：

1. 直接修改每个页面的 **view controller** 代码，简单粗暴；
2. 子类化 **view controller** ，并让我们的 **view controller** 都继承这些子类；
3. 使用 **Method Swizzling** 进行 hook，以达到 **AOP** 编程的思想

第一种实现的代码是在每个类的里面都这么写：

``` objc
- (void)viewDidLoad {
    [super viewDidLoad];
    DDLog();
}
```

第二种是只在基类里面写。然后所有的控制器都继承这个基类。
最后一种是最佳的解决方案：

``` objc
@implementation UIViewController (Log)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL originalSelector = @selector(viewDidLoad);
        SEL swizzledSelector = @selector(log_viewDidLoad);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        BOOL success = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (success) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)log_viewDidLoad{
    [self log_viewDidLoad];
    DDLog(...);
}

@end
```

**注意：** 

- 为什么使用 **+ (void)load** ？因为父类、子类和分类的该方法是分别调用，互不影响，而且是在类被加载的时候必定会调用的方法。

