## bazel 学习记录
bazel 是一款类似于 make、makefile 的开源构建和测试工具，他支持多种编程语言编写的项目，可以为多个平台构建。

需要构建工具的原因主要有以下几点：
- 项目的不同部分由不同的编程语言编写
- 项目存在外部依赖，这些依赖位于本地不同位置或远程代码仓库
- 项目构建存在耗时，希望构建时仅对更改的部分重建

### WORKSPACE file
是 Bazel 的工作区，用于存储项目源文件和 Bazel 构建输出的目录，将该目录及其内容标识为Bazel工作区的文件。他需要位于项目目录的根目录下，可以为空，但是通常会包含从网络或本地提取的其他依赖项的外部代码库声明

### BUILD file
一个项目中有一个或多个，主要用于告知 Bazel 如何构建项目，工作区包含一个 BUILD 文件的目录就是一个软件包。

### Bazel 的 c++ 事例
example 由 Bazel 官方提供的，github地址为 github.com/bazelbuild/…

#### 单个目标单个软件包
刚开始文件目录如下：
```css
examples
└── cpp-tutorial
    └──stage1
       ├── main
       │   ├── BUILD
       │   └── hello-world.cc
       └── WORKSPACE
```
构建：
```shell
bazel build //main:hello-world
```

执行完构建之后目录结构变成：
```css
examples
└── cpp-tutorial
    └──stage1
       ├── bazel-bin
       ├── bazel-out
       ├── bazel-stage1
       ├── bazel-testlogs
       ├── main
       │   ├── BUILD
       │   └── hello-world.cc
       └── WORKSPACE
```

测试：
```shell
bazel-bin/main/hello-world
```
会打印 Hello world

#### 单个软件包多个目标

下面的BUILD文件告诉Bazel先构建hello-greet库（使用Bazel内置的cc_library）然后构建hello-world二进制文件，其中的deps属性告诉Bazel构建hello-world需要hello-greet库。
```css
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
    ],
)
```

构建方式如上

如果我们现在修改 hello-greet.cc 并重新构建项目，Bazel 只会重新编译该文件。

#### 多个软件包多个目标
结构如下：
```css
└──stage3
   ├── main
   │   ├── BUILD
   │   ├── hello-world.cc
   │   ├── hello-greet.cc
   │   └── hello-greet.h
   ├── lib
   │   ├── BUILD
   │   ├── hello-time.cc
   │   └── hello-time.h
   └── WORKSPACE
```
其中，lib目录下的BUILD文件如下
```css
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "hello-time",
    srcs = ["hello-time.cc"],
    hdrs = ["hello-time.h"],
    visibility = ["//main:__pkg__"],
)
```

main目录下的BUILD文件如下
```css
load("@rules_cc//cc:defs.bzl", "cc_binary", "cc_library")

cc_library(
    name = "hello-greet",
    srcs = ["hello-greet.cc"],
    hdrs = ["hello-greet.h"],
)

cc_binary(
    name = "hello-world",
    srcs = ["hello-world.cc"],
    deps = [
        ":hello-greet",
        "//lib:hello-time",
    ],
)
```

主软件包中的 hello-world 目标依赖于 lib 软件包中的 hello-time 目标（因此是目标标签 //lib:hello-time）- Bazel 通过 deps 属性知道这一点。

@rules_cc//cc:defs.bzl 是 Bazel 中 C/C++ 项目的构建基石，通过加载它可以使用官方预定义的规则，实现高效、可复用的编译流程。其设计体现了 Bazel 的模块化、沙盒化和跨平台特性

为了顺利构建，请使用可见性属性使 lib/BUILD 中的 //lib:hello-time 目标明确对 main/BUILD 中的目标可见。这是因为默认情况下，目标仅对同一 BUILD 文件中的其他目标可见。Bazel 使用目标可见性来防止出现包含实现细节的库泄露到公共 API 等问题。
