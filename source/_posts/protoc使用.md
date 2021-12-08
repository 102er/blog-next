---
title: protoc使用
date: 2021-11-12 17:24:00
Tags: 序列化 GO
---

## 概述

grpc使用protobuf进行数据的序列化和反序列化，在开发中经常需要写proto文件，进行接口声明定义，然后通过protoc工具转成相应的代码文件，protoc工具不支持生成go代码文件，需要按照相应的插件协助；本文主要介绍protoc工具的使用，以及proto文件的定义。

<!-- more -->

## 介绍

### protoc指令

```
protoc	--proto_path=api/v1 \
        -I=api/v2 \
			 --gogo_out=paths=source_relative:. \
       api/v1/hello_world.proto
```

 **-IPATH, --proto_path=PATH 也可以用 -I**：它表示protoc在哪个路径下搜索proto文件，可以重复使用这个参数，表示指定多个目录进行搜索。啥场景需要指定这个参数？

--- 很多情况下，我们会在一个proto文件里面import其他proto文件，就需要指定这个路径，使protoc可以找到依赖。

**--xxx_out=OUT_DIR**：指定使用哪个插件生成相应语言的代码。go需要自己安装插件，一般会选择官方的protoc-gen-go 【--go_out=XXXX】，如果对自定义要求高，可以了解protoc-gen-gogo 【--gogo_out=XXXX】；`--xxx_out`主要的两个参数为`plugins` 和 `paths`，分别表示生成Go代码所使用的插件，以及生成的Go代码的位置。`--go_out`的写法是，参数之间用逗号隔开，最后加上冒号来指定代码的生成位置，比如`--go_out=plugins=grpc,paths=import:.`

- `paths`参数有两个选项，分别是 `import` 和 `source_relative`，默认为 import，表示按照生成的Go代码的包的全路径去创建目录层级，source_relative 表示按照 **proto源文件的目录层级**去创建Go代码的目录层级，如果目录已存在则不用创建。
- ​	`plugins`参数有不带grpc和带grpc两种（应该还有其它的，目前知道的有这两种），两者的区别如下，带grpc的会多一些跟gRPC相关的代码，实现gRPC通信：

`paths`参数有两个选项，分别是 `import` 和 `source_relative`，默认为 import，表示按照生成的Go代码的包的全路径去创建目录层级，source_relative 表示按照 **proto源文件的目录层级**去创建Go代码的目录层级，如果目录已存在则不用创建。

  **  @<filename> ** ：表示要生成的proto文件，可以是一个目录，它会把目录下所有proto文件生成对应的代码。

### proto文件

```
syntax = "proto3";

package helloWorld;

option go_package="api/v1";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
message HelloRequest {
  string name = 1;
}
message HelloReply {
  string message = 1;
}
```

- syntax：指定proto版本，默认是proto2进行编译
- `package`：是proto文件的命名空间，避免我们定义的service、message出现冲突。
- go_package：生成go语言对应的包路径，对应go的包名，它的设置上需要和--go_out==xxx相同，保证go能够被正确引用。（当然，不一样也可以，这样需要保证go_out目录下只有这个包，不然就会出现冲突，所以，我们设置上一般保证相同，减少不必要的问题）
- 注意，不同包之间的 proto 文件不可以循环依赖，这会导致生成的 go 包之间也存在循环依赖，导致 go 代码编译不通过。

