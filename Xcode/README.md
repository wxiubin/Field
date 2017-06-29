
## Xcode 的配置


#### Code Snippets

CodeSnippets（Xcode的代码片段快捷键）的目录是 `~/Library/Developer/Xcode/UserData/CodeSnippets`

#### Key Bindings


KeyBindings（快捷键映射，可以把页面切换的前进后退和.h与.m切换分别设置成F1,F2和F3）的目录是 `~/Library/Developer/Xcode/UserData/KeyBindings`

#### FontAndColorThemes

FontAndColorThemes（主题字体颜色配置）目录 `~/Library/Developer/Xcode/UserData/FontAndColorThemes`

## Xcode 常见的编译错误

> 收集记录一些Xcode编译报错及解决办法，会持续更新...


### 1. Code signing is required for product type 'Application' in SDK 'iOS 10.0'

真机调试时如果报以下错误：

``` objc
No profiles for 'BundleID' were found:  Xcode couldn't find a provisioning profile matching 'BundleID'.
Code signing is required for product type 'Application' in SDK 'iOS 10.0'
```

<!--more-->

在 `PROJECT` 的设置里 `Build Setting` 下的 `Code Signing Identity` 全部选中 `iOS Developer`。[Stackoverflow解决参考](http://stackoverflow.com/questions/37806538/code-signing-is-required-for-product-type-application-in-sdk-ios-10-0-stic)


### 2. _OBJC_METACLASS_$_(__CLASS__)", referenced from:

该错误一般是因为Objc中的类只写了 `@interface __CLASS__` ，却没有实现 `@implementation __CLASS__`

### 3. Undefined symbols for architecture i386: "_OBJC_CLASS_$_(__CLASS__)", referenced from:

```
Undefined symbols for architecture i386:
  "_OBJC_CLASS_$_HTQuoteDecoder", referenced from:
      objc-class-ref in libHTConnect.a(HTCTestcase.o)
```

![](http://od6bpfkkf.bkt.clouddn.com/Undefined-symbols-for-architecture-i386.png)

如图所示的error，是 `libHTConnect.a`库的 `HTQuoteDecoder` 文件引用问题。

