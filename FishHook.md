# fishhook使用及原理探索

[转载原文: iOS逆向 ---- fishhook使用及原理探索](https://juejin.cn/post/6844904175625568270)

## 前言

在上一篇博客 《iOS逆向 ---- Hook方法及原理OC篇》 中，我们探索OC中应用方法交换进行Hook的原理，但是这种方法仅仅对OC方法有效，本篇我们可以一起探索一下如何Hook一个C函数。


## 一、fishhook简介及使用

众所周知，C语言是一门静态语言，而静态语言在编译完成后变量、函数及其参数就已经确定，无法修改。但实际上在运行时我们可以通过动态重新绑定符号来改变函数（能生成符号表的函数）的实现，从而实现Hook C函数的目的。在我们进行动态重绑定符号时，要用到的一个第三方库fishhook，我们一起来探索一下。
 fishhook 是由 Facebook 开源的一个重新绑定符号的库，其主要作用是在程序运行时动态修改绑定符号指针。感兴趣的小伙伴可以 点击下载源码 。 fishhook  源码并不复杂，从  fishhook.h 文件中可以看到，其主要由  rebinding 结构体、 rebind_symbols、 rebind_symbols_image  函数构成。

### 1.1 fishhook方法简介

1、 rebinding结构体 在其源码中的定义如下：

```
/*
 * A structure representing a particular intended rebinding from a symbol
 * name to its replacement
 */
struct rebinding {
  const char *name;  //需要HOOK的函数名称，C字符串
  void *replacement; //新函数的地址
  void **replaced;   //原始函数地址的指针！
};
```

2、 rebind_symbols函数

```
/*
 * For each rebinding in rebindings, rebinds references to external, indirect
 * symbols with the specified name to instead point at replacement for each
 * image in the calling process as well as for all future images that are loaded
 * by the process. If rebind_functions is called more than once, the symbols to
 * rebind are added to the existing list of rebindings, and if a given symbol
 * is rebound more than once, the later rebinding will take precedence.
 */
FISHHOOK_VISIBILITY
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel);
```

该函数用来重新绑定符号所指向的实现地址，包含两个参数，其中 rebindings[] 是一个 rebinding类型数组，用来存储需要hook的函数构建成 rebinding 结构体， rebindings_nel 表示数组的长度。

3、 rebind_symbols_image函数

```
/*
 * Rebinds as above, but only in the specified image. The header should point
 * to the mach-o header, the slide should be the slide offset. Others as above.
 */
FISHHOOK_VISIBILITY
int rebind_symbols_image(void *header,
                         intptr_t slide,
                         struct rebinding rebindings[],
                         size_t rebindings_nel);
```

该函数功能与上一个函数类似，不过是用来重绑定制定镜像文件中的符号，其中 header 表示指定镜像的header， slide 表示偏移量，后两个参数意义与上一个函数一致。

### 1.2 fishhook实际应用--hook系统printf函数

我们通过 rebind_symbols 函数对系统的 printf 函数进行hook，具体代码如下：

```
// hook系统printf函数代码
Rebinding rebind;
rebind.name = "printf";
rebind.replacement = myPrintf; // 将自定义的函数赋值给replacement
rebind.replaced = (void *)&oldPrintf; // 使用自定义的函数指针来接收printf函数原有的实现

Rebinding rebs[1] = {rebind};
rebind_symbols(rebs, 1);

// 定义一个函数指针用来接收并保存系统C函数的实现地址
static int(*oldPrintf)(const char *, ...);

// 定义我们自己的printf函数
int myPrintf(const char * message, ...) {
    char *firstName = "真棒\n";
    char *result = malloc(strlen(message) + strlen(firstName));
    strcpy(result, message);
    strcat(result, firstName);
    
    oldPrintf(result);
    return 1;
}

// 在touchBegan方法中调用printf函数
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    printf("点击了屏幕！！");
}
```

代码运行效果如下：
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172666f5d663e540~tplv-t2oaga2asx-watermark.awebp)

如上所示，在点击屏幕后，本来应该打印 “点击了屏幕！！”，实际打印了 “点击了屏幕！！真棒”，由此证明我们对 printf 函数的Hook是成功的。下面我们看下Hook一个系统C函数有哪些事情要做。

1、构建一个Rebinding结构体

> name： 符号名称，一个C字符串，用来表明我们要hook哪个函数
>
> replacement： 新函数的地址，用来表示要用哪个函数进行替换原函数实现
>
> replaced：二级函数指针，用来保存原函数的实现地址，记录原函数的实现。

2、构建一个Rebinding结构体数组，将要Hook的函数构建结构体完毕后，放在此数组中即可，可以Hook多个函数。

3、调用rebind_symbols函数，进行符号重绑定。

### 1.3 hook一个自定义C函数

以上是Hook一个系统C函数的过程，接下来我们尝试Hook一个自定义函数 func 。

```
// 自定义函数
void func(const char *str) {
    NSLog(@"%s", str);
}

// 
static void(*myFunc)(const char *str);
void newFunc(const char *str) {
    char *subStr = "真棒！勾住了\n";
    char *result = malloc(strlen(str) + strlen(subStr));
    strcpy(result, str);
    strcat(result, subStr);
    
    myFunc(result);
}

// hook过程
func("hello world");
// Custom Function
Rebinding customRebind;
customRebind.name = "func";
customRebind.replacement = newFunc;
customRebind.replaced = (void *)&myFunc;

Rebinding rebs[1] = {customRebind};

rebind_symbols(rebs, 1);
func("hello world");
```

运行结果如下

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/17268d8b451e9928~tplv-t2oaga2asx-watermark.awebp)

我们预想的结果应该是，后一次打印会拼接 “真棒！勾住了”，实际前后两次调用结果是一样的，证明我们的hook并未成功。

由此我们引出一下几个问题：

> 问题1：replaced 为何使用二级指针？
>
> 问题2：为何系统函数可以Hook成功，自定义函数无法Hook成功？
> 
> 问题3：fishhook对函数进行Hook的原理是什么？

## 二、fishhook原理探索

上诉三个问题我们先从第三个问题开始，因为这个问题解决了，前两个问题的答案就显而易见了。

### 2.1 静态链接与动态链接

在之前的博客 dyld流程分析 中介绍过，build一个工程主要包含了 预编译、编译、汇编、链接 几个过程

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/17268fbb8b31357e~tplv-t2oaga2asx-watermark.awebp)

在实际的开发过程中，我们的程序代码不可能都放在一个文件里，因此我们需要通过符号将不同的文件链接起来，链接又分为静态链接(编译时)和动态链接(运行时)。如图所示在build一个工程后生成可执行文件，发生的链接是静态链接，而在dyld加载程序到内存中发生的链接为动态链接。

1、静态链接的特点是将一个进程中用到的库都加载进内存，而不管内存中是否加载过。

2、动态链接发生在加载可执行文件到内存时，系统先检测应用程序依赖的库中有没有已经加载的，若有则直接去内存取，不会重复加载。

### 2.2 fishhook原理--符号重新绑定

符号就是不同文件之间相互引用的函数名和变量名，比如A、B两个文件，B中调用A中的函数func()，我们称A定义了一个函数func()，B引用了A中的函数func()，这里func就是一个符号，我们可以将符号看作链接的粘合剂，整个链接过程基于符号才能够正确的完成。而我们自定义的函数是放在我们自己的文件中的，不会生成符号，所以无法Hook成功，fishhook只能Hook能生成符号表的C函数。

在程序编译完成生成可执行文件时，会先进行静态链接，此时会给系统的C函数的符号指定一个无意义的地址，等到加载完共享缓存之后，会讲共享缓存库中的函数地址赋值给这个符号。此时我们对符号地址进行重新绑定，使其指向我们自己的函数，就完成了系统C函数的符号重新绑定，实现了Hook的目的。如果我们希望函数保持原有的功能，则需要定义一个二级指针，用于在加载完共享缓存后，保存原有函数的实现地址。比如printf示例中的 oldPrintf。

我们通过分析printf示例中printf的符号指向来验证这一过程，我们分别在 printf第一次调用前、printf 第一次调用后、重绑定后 打三个断点，观察符号指向的内容。首先通过 image list 指令在控制台中看下当前demo的Ma ch-O文件在内存中的地址为 0x00000001042b8000 （不同demo该值不一样），然后加上printf 符号在文件中偏移地址 80C8 ，该值可在Mach-O文件中查看，如图：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172694d5e6bb0a91~tplv-t2oaga2asx-watermark.awebp)

根据文件的首地址加上符号的偏移地址，我们可以得出符号在内存中的地址为 0x1042C00C8 ，通过x/4gx 和 dis -s 命令查看其内容，三次结果分别为：

第一次调用前：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/17269584a5e21823~tplv-t2oaga2asx-watermark.awebp)
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/1726958a1eae3a8a~tplv-t2oaga2asx-watermark.awebp)

第一次调用后，即共享缓存中printf的地址：
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172695b2748a5dd2~tplv-t2oaga2asx-watermark.awebp)
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172695b45534e25f~tplv-t2oaga2asx-watermark.awebp)

重绑定后：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172695d3e2515177~tplv-t2oaga2asx-watermark.awebp)

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172695dd266304fc~tplv-t2oaga2asx-watermark.awebp)

通过对比三次的结果，我们发现在第一次调用前，printf的内容即没有说明是哪个文件下的，也没有为函数开辟空间（没有sub指令），所以其并不是一个函数地址；而第一次调用后和重绑定后，printf分别指向一块函数地址，且分别属于  libsystem_c.dylib  和 当前demo的Mach-O文件  FishhookDemo ，符号名称也由  printf  变为  myPrintf 。

### 2.3 fishhook如何通过符号找到函数

fishhook通过符号找到函数的过程有一下几步，如图所示：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/1726967bb64ff1d0~tplv-t2oaga2asx-watermark.awebp)

1、在_DATA_段的懒加载表 `_la_symbol_ptr` 找到对应的符号
如果没有则符号未定义

2、如果有则在 Dynamic Symbol Table 中找到对应符号的记录
根据记录中的Data值

3、将根据Data值在 Symbol Table 中找到对应下标的记录

4、根据记录中的 String Table Index 在 String Table中找到对应的
函数名或变量名

我们依旧通过printf函数来分析这一过程，首先在Mach-O文件中的懒加载表/非懒加载表找到符号

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172694d5e6bb0a91~tplv-t2oaga2asx-watermark.awebp)

接下来我们在 Dynamic Symbol Table 中找到对应的符号

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/172696d82bd283a7~tplv-t2oaga2asx-watermark.awebp)

将Data值117换算成十进制为279，在Symbol Table找到对应下标的记录

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/1726971f2f1eb7e6~tplv-t2oaga2asx-watermark.awebp)

根据String Table Index的值加上 String Table的首地址即可找到printf

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/31/17269736eac7dff0~tplv-t2oaga2asx-watermark.awebp)

## 三、总结

fishhook是基于对符号的重新绑定进行Hook的，可以Hook有能生成符号的函数，对于同一个文件中的函数无法Hook，因为其不会生成符号。以上即为小编对fishhook 原理的探索，希望对有需要的朋友有所帮助，也欢迎大家进行交流和指正。