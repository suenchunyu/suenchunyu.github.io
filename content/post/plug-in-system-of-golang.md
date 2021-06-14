+++
author = "Suen ChunYu"
title = "Plug-in System of Golang"
date = "2021-02-01"
description = "Go的插件机制"
tags = [
    "golang",
    "cgo",
]
categories = [
    "Go"
]

+++

> Go语言的开发者一般关注点更多的会放在其比较热门的特性上，例如：Goroutine、GPM和Channel等，但Go提供的远远不止这些，其中的插件机制和CGO相对来说都是比较少关注的Features，通过插件机制可以实现构建松散耦合的模块化程序。

## 1. 设计原理
Go的插件机制基于的是C语言的动态库机制实现的，所以本身Go直接继承了C语言动态库的优缺点；在Linux中一般会碰**静态链接库**和**动态库**两种，它们各自的特点和优势大不相同，我们一般要结合实际场景酌情选择：

- 静态库（静态链接库）：一般是由开发者自定的变量和函数构成，其构成简单，不像动态链接库那么复杂，在编译期间一般由编译器或者链接器将它们集成到目标程序内，并打包制作成可独立运作的可执行文件。
- [动态库（共享对象）](https://zh.wikipedia.org/wiki/%E5%87%BD%E5%BC%8F%E5%BA%AB#%E5%8A%A8%E6%80%81%E5%BA%93)：和Windows的[动态链接库](https://zh.wikipedia.org/wiki/Dynamic_Link_Library)不同，在Unix或者Linux环境下是共享对象（共享库），预先编译链接好的可执行文件，程序在使用这些库时是从共享对象中加载，而不是在编译期间打包进程序。

由于特性的不同，动态库和静态库的优缺点也就非常明显；只依赖静态库并通过静态链接的二进制程序可以独立执行，但是由于编译后的结果包含了全部的依赖，这就导致编译后的结果比较大；而动态库往往是在多个可执行文件之间共享，加载动态库的过程也是在装载时（Load-Time）加载，大多数情况下同一时间多个应用可以使用一个库的同一份拷贝，系统并不需要加载这个库的多个实例，极大的降低了内存的占用。

![Libraries](https://suenchunyu.dev/images/libraries.png#center)

使用静态链接编译二进制文件在部署上有非常明显的优势，最终的编译结果也可以直接运行在大多数的机器上，静态链接带来的部署优势远比更低的内存占用显得重要，所以很多编程语言包括 Go 都将静态链接作为默认的链接方式。

## 2. 插件机制
通过在主程序和共享库之间定义一系列的接口，我们可以通过Go提供的`plugin`库来加载其他人编译的共享库对象，这样做的好处是主程序和共享库的开发者不需要彼此共享代码，只要在双方约定的接口不变的前提下，修改了共享库之后也不需要重新编译一遍主程序：

```go
type Extension interface {
  Name() string
}

func main() {
  if plug, err := plugin.Open("extension.so"); err != nil {
    panic(err)
  } else {
    initSymbol, err := plug.Lookup("Name")
    if err != nil {
      panic(err)
    }
    nameFunc := initSymbol.(func() Extension)
    name := nameFunc()
    fmt.Println(name)
  }
}
```

上述代码定义了`Extension`接口，并认为共享库中一定包含了`func Name() string`的函数，当我们通过[`plugin.Open`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L31)读取包含Go插件的共享库之后，获取文件中的`Name`符号并且将其转换为正确的函数类型，就可以通过此函数拿到它的名字了。

## 3. 不同操作系统下的不同实现
不同的操作系统下对应都有不同的动态链接机制和共享库格式，Linux中的共享对象会使用[ELF](https://zh.wikipedia.org/wiki/%E5%8F%AF%E5%9F%B7%E8%A1%8C%E8%88%87%E5%8F%AF%E9%8F%88%E6%8E%A5%E6%A0%BC%E5%BC%8F)格式并提供了一组操作动态链接器的接口：
```c
void *dlopen(const char *filename, int flag);
char *dlerror(void);
void *dlsym(void *handle, const char *symbol);
int dlclose(void *handle);
```

`dlopen`会根据传入的文件名加载对应的共享库并返回一个句柄（Handle）；`dlsym`函数可以在该句柄中搜索特定的符号，这个符号可以是函数也可以是变量，它会返回该符号被加载到内存中的地址；由于待查找的符号可能不存在于目标共享库中，所以每次查找后都需要调用`dlerror`查看当前查找的结果。

## 4. Go之中的共享库

Go语言的插件机制的全部实现都在[plugin](https://pkg.go.dev/plugin)包中，这个包实现了Go插件的加载和符号解析。  
由于Go插件是一个带有公开函数和变量的包，需要使用如下命令来将代码编译成Go插件：

```reStructuredText
go build -buildmode=plugin ...
```

该命令会生成一个共享对象`.so`文件，当该文件被加载到Go程序时会使用如下结构体[`plugin.Plugin`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L21)表示：

```go
// Plugin is a loaded Go plugin.
type Plugin struct {
	pluginpath string
	err        string        // set if plugin failed to load
	loaded     chan struct{} // closed when loaded
	syms       map[string]interface{}
}
```

与插件机制相关的两个方法分别是用于加载Go插件的[`plugin.Open`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L31)和在插件中查找符号的[`plugin.Plugin.Lookup`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L39)。

## 5. 插件机制之中的CGO

在Go的插件机制之中，使用到了CGO来完成对于动态链接器接口的操作，分别是[`plugin.pluginOpen`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L18)和[`plugin.pluginLookup`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L19)两个C的函数，这两个函数的实现非常简单，只是简单的对`dlopen`和`dlsym`做了简单的包装，[`plugin.pluginOpen`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L18)简单包装了下标准库中的`dlopen`和`dlerror`函数并在共享库加载成功之后返回指向共享库的句柄：

```c
static uintptr_t pluginOpen(const char* path, char** err) {
	void* h = dlopen(path, RTLD_NOW|RTLD_GLOBAL);
	if (h == NULL) {
		*err = (char*)dlerror();
	}
	return (uintptr_t)h;
}
```

[`plugin.pluginLookup`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L19)使用了标准库中的`dlsym`和`dlerror`用来获取共享库句柄中的特定的符号：

```c
static void* pluginLookup(uintptr_t h, const char* name, char** err) {
	void* r = dlsym((void*)h, name);
	if (r == NULL) {
		*err = (char*)dlerror();
	}
	return r;
}
```
这么包装的目的是让他们的函数签名看起来更像是Go中的函数签名，方便调用。

## 6. 加载过程
用于加载共享库的函数[`plugin.pluginOpen`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L18)会将共享库文件的路径作为参数并返回[`plugin.Plugin`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L21)结构：

```go
func Open(path string) (*Plugin, error) {
	return open(path)
}
```

这个函数会调用`plugin`包中的一个私有函数[`plugin.open`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L42)加载插件，这个函数是插件加载过程的核心函数，大体分为如下几个步骤：

1. 准备C语言函数[`plugin.pluginOpen`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L18)的参数；
2. 通过CGO机制调用[`plugin.pluginOpen`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin_dlopen.go#L18)并初始化加载的插件模块；
3. 查找加载模块中的`init`函数并调用此函数执行插件模块包的初始化过程；
4. 通过插件的文件名和符号列表构建[`plugin.Plugin`](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/plugin/plugin.go#L21)结构。

