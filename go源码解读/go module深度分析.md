# Go Modules 全面解析
自从Go11默认支持Module之后，Module已经成为Go官方的依赖管理工具。因此，Go Module的使用是每个Go开发者必须掌握的技术要点。

Go使用go.mod文件描述Module信息，包括Module Path和Module依赖的其他Module。编译时，Go会自动下载go.mod定义的依赖Module，从而完成编译工作。

go.mod文件必须放到项目根目录下(不可放到\$GOPATH/src目录)，下载的依赖Module会自动存在$GOPATH/pkg/mod目录。

## Go Module使用

在根目录中执行`go mod init {module-path}`命令，例如执行`go mod init icode.company.com/company/hello`就可以在当前目录创建go.mod文件。
、、、
cat go.mod
module icode.company.com/company/hello
、、、

一般来讲，`module-path`的格式没有特别要求，好的`module-path`命名应该描述了module的作用以及去哪里下载它，比如`github.com/docker/compose`。

此时go.mod文件中并没有依赖module，如果在package中import了其他module，例如：
、、、
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
、、、
在根目录执行`go build`命令，此时Go会自动下载依赖的Module，并写入到go.mod文件中
、、、
module icode.company.com/company/hello

go 1.12

require rsc.io/quote v1.5.2 // require定义了依赖的module，并指定了版本
、、、
其他操作包括更新依赖的module并删除无须再依赖的module时，执行`go mod tidy`，查看当前module依赖的所有module，执行`go list -m all`，查看module依赖关系可使用`go mod graph | grep {module-path}`，当分析为什么需要这个module时特别有用。


## Module版本管理
Module的版本是不可修改的代码快照。
Module的版本格式如`v1.0.23`，`v0.0.0`，其中`v`是固定的，其它部分别包括：
1. 主版本号：在发布向后不兼容的版本号时递增，同时次版本号和补丁版本号重置为零
2. 次版本号：添加新的功能后，次版本号递增，同时补丁版本号重置为零
3. 补丁版本号：修复bug之后，补丁版本号递增
4. -pre后缀：意味这个版本是预发布，例如`v1.2.22-pre`

一般来讲，如果主版本号是`0`, 那可以认为这个版本是不稳定的，例如`v0.2.0`可能与`v0.1.0`不兼容。当我们管理Module版本时，需要制定合理的主版本号，不能随意发布版本号，需要遵守以上规则。

在更新依赖的module，我们有两种选择：只更新指定的module；更新所有的module。

使用`go get module-path@version`更新到指定的版本，例如`go get golang.org/x/text@v0.3.2`

使用`go get -u golang.org/x/text@v0.3.2`更新到指定的版本，同时其他依赖的module都会更新到最新版本。

我们不能保证最新的module版本都是向后兼容的，所以最好不要使用`go get -u`更新。


### 伪版本号
Module的版本格式并不是强制要求，Module管理者发布版本时，可以不用打版本号。同时，当Module使用方更新版本时，指定的是module的分支而非具体某个版本，此时Go会自动生成一个伪版本号，例如`v0.0.0-20210525053744-b6e0ccf2452d`。

伪版本也分为三个部分：
1. 基础版本前缀，如果之前指定了版本号，则使用之前的版本，否则使用`v0.0.0`。
2. 时间戳，格式为(yyyymmddhhmmss)，即创建版本时的具体时间。
3. 修订标识符，版本提交时的相关信息。

例如，我们更新module时不指定版本，而是指定分支`go get golang.org/x/text@master`，此时go会下载`golang.org/x/text`的master分支的代码。

使用分支会有很多问题，例如我们无法保证分支上的代码一定是向后兼容的，也不确定分支上面的代码到底修改了多少，当代码回退时，我们也不清楚具体要回退到哪个伪版本。

所以，最好的方式还是采用指定版本号的方式管理依赖。

### 多版本依赖
我们先考虑这样一个问题，Module-A依赖Module-B1和Moudle-C1，Module-B1又依赖Module-C2。当我们编译Module-A时，会同时依赖Module-C1和Module-C2，此时会选择哪个版本呢，还是两个版本都会选择？

Go使用[Minimal version selection](https://research.swtch.com/vgo-mvs)算法来选择需要构建的版本。

Go选择版本时，遵守了最小版本选择原则，针对以上例子，如果Module-B同时存在1,2两个版本，但是在Module-A指定了Module-B1，那go只会选择Module-B1，而不是更高版本Module-B2。

如果同时依赖Module-C1, Module-C2，此时G会选择更高版本Module-C2。

也就是编译Module-A时，使用的版本是`Module-A, Module-B1, Module-C2`。

那这里可能会存在一个严重问题，如果Module-C2和Module-C1不兼容怎么办？

当使用Module-C1时，Module-B1无法通过，当使用Module-C2时，Module-A无法通过。

当出现这个问题时，我们可以使用`replace`指令来实现通过依赖Module-C1和Module-C2。

比如下面例子同时依赖`v0.8.1`和`v0.9.1`，使用如下：
、、、
module multiversion

go 1.13

replace github.com/pkg/errors/081 => github.com/pkg/errors v0.8.1
replace github.com/pkg/errors/091 => github.com/pkg/errors v0.9.1
、、、
使用replace过的module-path：
、、、
package main

import (

    "fmt"

    errors081 "github.com/pkg/errors/081"

    errors091 "github.com/pkg/errors/091"

)


func main() {

    err := errors081.New("New error for v0.8.1")

    fmt.Println(err)

    err = errors091.New("New error for v0.9.1")

    fmt.Println(err)

}
、、、

再次总结一下，当依赖多个不同的Module版本时，Go会优先选择高版本。

`replace`指令除了用于处理多版本后，还有一个更实用的场景，例如在代码开发阶段，我们需要测试本地的代码，我们可以使用`replace`指令将编译的module指定为本地代码，例如：
、、、
replace icode.com/compose/seda => /work/vsgo-179/seda
、、、
这样在编译`icode.com/compose/seda` Module时，会使用本地代码`/work/vsgo-179/seda`。

## 其他操作

### go mod download

### go clean -modcache

### go mod edit
