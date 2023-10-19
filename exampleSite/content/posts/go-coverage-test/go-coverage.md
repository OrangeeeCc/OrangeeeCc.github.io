---
weight: 1
title: "浅谈 Golang 代码覆盖率"
date: 2021-06-29T21:40:32+08:00
lastmod: 2021-06-29T21:40:32+08:00
draft: false
author: "Xunzhuo"
authorLink: "https://github.com/xunzhuo"
description: "本文浅谈一下 Golang 代码测试覆盖率的一些细节与原理."

tags: ["Golang", "Toolings"]
categories: ["Golang"]

toc:
  auto: false
---

## 前言

平时开发的时候，大家都会接触到对 Golang 代码编写**单元测试**，其中我们常常会接触到**代码覆盖率**的概念。

通过编写单元测试，并提高代码覆盖率，我们能尽可能减少错误的发生，从而提高代码逻辑的健壮性，本文浅谈一下 Golang 代码覆盖率的一些细节。

## 覆盖率技术

测试覆盖率是一个术语，是指通过运行单元测试，测试逻辑能使多少业务逻辑代码得到执行与验证，计算方式为**已覆盖行数/总行数**。 如果执行单元测试导致50％的语句得到了运行，则测试覆盖率为50％。

传统的计算测试覆盖率的方法是对二进制可执行文件进行埋点，整体思路是在二进制文件中设置断点，当每个分支被执行的时候，断点即被清除，并且目标分支的语句会被标记为已覆盖。

这种方法是成功和广泛使用的。但是有如下几个问题：

1. **实现难度大**。分析二进制文件的执行是困难的，它需要将执行跟踪绑定回源代码的可靠方法，这里的问题包括不正确的调试信息和诸如内联功能的问题等， 使分析变得复杂。 
2. **可移植性差**。 对于每个机器架构需要重新编写，在某种程度上,可能对于每个操作系统都需要重新编写，因为从系统到系统的调试支持差异很大。

Go 的早期测试覆盖工具甚至以相同的方式工作，*Go 1.2* 的发布引入了一个 *test coverage* 的新工具， 它采用插桩源码的形式，整体思路是在编译之前重写测试包的源码以埋点，然后再进行编译和运行，并统计覆盖信息。

这个方案实现起来也较为简单，因为 Go 控制了从测试到执行的全部流程，只需在执行 go 命令时，悄悄的重写测试代码，执行并统计数据即可。

但是同时这个方案的问题也是有的，显然它会修改目标程序的测试源码，即此方式是具有侵入性的，实际执行测试的代码是重写后的源码。

下面会提供一个 demo 进行演示，梳理一下 `go test -cover` 以及 `go tool cover` 的常见使用，并一起来看下它是如何重写测试源码，并统计覆盖率的。

## Go 覆盖率计算

以下面目标代码为例：

```go
package demo

func WhoAmI(name string) string {
	switch name {
	case "xunzhuo":
		return "first name"
	case "liuxunzhuo":
		return "full name"
	case "liu":
		return "last name"
	case "bitliu":
		return "nick name"
	default:
		return "unknown"
	}
}

func HowOldAmI(age string) int {
	switch age {
	case "22":
		return 22
	case "23":
		return 23
	default:
		return 24
	}
}
```

我们编写一个对其的单元测试：

```go
package demo

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestFirstName(t *testing.T) {
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
	}

	for _, testcase := range testcases {
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

func TestFirstAndLastName(t *testing.T) {
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
		{
			description: "who is liu",
			input:       "liu",
			expect:      "last name",
		},
	}

	for _, testcase := range testcases {
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}
```

*go test* 命令接受 `-covermode` 标志将覆盖模式设置为三种设置之一：

- set: 每个语句是否执行？
- count: 每个语句执行了几次？
- atomic: 类似于 *count*, 但表示的是并行程序中的精确计数。

*go test* 通过 `-coverprofile` 指定输出的测试报告文件。

*go test* 不指定 `covermode` 也可以直接传入 `-cover`，默认的是 set 模式。

下面通过命令生成了三种模式下的测试报告：

``` shell
❯ go test -v -covermode=count -coverprofile=count.out ./
=== RUN   TestFirstName
--- PASS: TestFirstName (0.00s)
=== RUN   TestFirstAndLastName
--- PASS: TestFirstAndLastName (0.00s)
PASS
coverage: 30.0% of statements
ok      github.com/xunzhuo/demo/demo    0.484s  coverage: 30.0% of statements
❯ go test -v -covermode=atomic -coverprofile=atomic.out ./
=== RUN   TestFirstName
--- PASS: TestFirstName (0.00s)
=== RUN   TestFirstAndLastName
--- PASS: TestFirstAndLastName (0.00s)
PASS
coverage: 30.0% of statements
ok      github.com/xunzhuo/demo/demo    0.433s  coverage: 30.0% of statements
❯ go test -v -covermode=set -coverprofile=set.out ./
=== RUN   TestFirstName
--- PASS: TestFirstName (0.00s)
=== RUN   TestFirstAndLastName
--- PASS: TestFirstAndLastName (0.00s)
PASS
coverage: 30.0% of statements
ok      github.com/xunzhuo/demo/demo    0.333s  coverage: 30.0% of statements
```

测试报告可读性较差，如生成的 `atomic.out`:

```shell
mode: atomic
github.com/xunzhuo/demo/demo/about.go:3.33,4.14 1 3
github.com/xunzhuo/demo/demo/about.go:5.17,6.22 1 2
github.com/xunzhuo/demo/demo/about.go:7.20,8.21 1 0
github.com/xunzhuo/demo/demo/about.go:9.13,10.21 1 1
github.com/xunzhuo/demo/demo/about.go:11.16,12.21 1 0
github.com/xunzhuo/demo/demo/about.go:13.10,14.19 1 0
github.com/xunzhuo/demo/demo/about.go:18.32,19.13 1 0
github.com/xunzhuo/demo/demo/about.go:20.12,21.12 1 0
github.com/xunzhuo/demo/demo/about.go:22.12,23.12 1 0
github.com/xunzhuo/demo/demo/about.go:24.10,25.12 1 0
```

`go tool cover` 提供命令去展示上述的覆盖率报告，通过 `-func` 以命令行方式展示：

```shell
❯ go tool cover -func=atomic.out
github.com/xunzhuo/demo/demo/about.go:3:        WhoAmI          50.0%
github.com/xunzhuo/demo/demo/about.go:18:       HowOldAmI       0.0%
total:                                          (statements)    30.0%
```

通过 `-html` 以可视化方式展示：

```shell
❯ go tool cover -html=atomic.out
```

在浏览器中可以看到 atomic 模式的覆盖率报告，提供三种颜色也区分：

+ 绿色代表已覆盖，深浅代表覆盖的次数
+ 灰色代表已覆盖，覆盖次数较少
+ 红色代表未被覆盖

![image-20230130152328738](https://liuxunzhuo.oss-cn-chengdu.aliyuncs.com/blog/image-20230130152328738.png)

前文也提到多次 golang 覆盖率技术是基于重写代码并埋点，我们一起来看看具体是如何重写代码并生成报告的。

`go tool cover` 提供命令去展示重写后的代码，`-mode` 通过 mode 去指定目标代码的覆盖模式，与 go test 一致，支持 set、count、atomic。

### Count 模式埋点代码

以下为例，count 模式下重写后的代码：

```shell
❯ go tool cover -mode=count about_test.go > count_test.go
```

生成的代码为：

```go
//line about_test.go:1
package demo

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestFirstName(t *testing.T) {GoCover.Count[0]++;
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
	}

	for _, testcase := range testcases {GoCover.Count[1]++;
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

func TestFirstAndLastName(t *testing.T) {GoCover.Count[2]++;
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
		{
			description: "who is liu",
			input:       "liu",
			expect:      "last name",
		},
	}

	for _, testcase := range testcases {GoCover.Count[3]++;
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

var GoCover = struct {
	Count     [4]uint32
	Pos       [3 * 4]uint32
	NumStmt   [4]uint16
} {
	Pos: [3 * 4]uint32{
		9, 22, 0x250022, // [0]
		22, 25, 0x30025, // [1]
		28, 46, 0x250029, // [2]
		46, 49, 0x30025, // [3]
	},
	NumStmt: [4]uint16{
		2, // 0
		2, // 1
		2, // 2
		2, // 3
	},
}

```

可以看到，执行完之后，源码里多了个`GoCover`变量，其有三个比较关键的属性:

- `Count` uint32 数组，数组中每个元素代表相应基本块 (basic block) 被执行到的次数
- `Pos` 代表的各个基本块在源码文件中的位置，三个为一组。比如这里的`21`代表该基本块的起始行数，`23`代表结束行数,`0x2000d`比较有趣，其前 16 位代表结束列数，后 16 位代表起始列数。通过行和列能唯一确定一个点，而通过起始点和结束点，就能精确表达某基本块在源码文件中的物理范围
- `NumStmt` 代表相应基本块范围内有多少语句 (statement)

变量会在每个执行逻辑单元设置个计数器，比如 `GoCover.Count[0]++`, 而这就是所谓插桩了。通过这个计数器能很方便的计算出这块代码是否被执行到，以及执行了多少次。相信大家一定见过表示 go 覆盖率结果的 coverprofile 数据，类似下面:

`github.com/xunzhuo/demo/demo/about.go:3.33,4.14 1 3`

这里的内容就是通过类似上面的变量`GoCover`得到。其基本语义为
"**文件:起始行.起始列,结束行.结束列 该基本块中的语句数量 该基本块被执行到的次数**"


可以看到覆盖的次数，每次执行都会去 +1，可以看出它不仅关注了代码是否被覆盖，同时关注了代码被覆盖的次数，这能看出重要代码是否被多次覆盖，对我们单测编写有指导作用。

### Set 模式埋点代码

我们再来看看 set 模式下重写后的代码以及它的区别：

```shell
go tool cover -mode=set about_test.go > set_test.go
```

代码如下：

```go
//line about_test.go:1
package demo

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestFirstName(t *testing.T) {GoCover.Count[0] = 1;
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
	}

	for _, testcase := range testcases {GoCover.Count[1] = 1;
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

func TestFirstAndLastName(t *testing.T) {GoCover.Count[2] = 1;
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
		{
			description: "who is liu",
			input:       "liu",
			expect:      "last name",
		},
	}

	for _, testcase := range testcases {GoCover.Count[3] = 1;
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

var GoCover = struct {
	Count     [4]uint32
	Pos       [3 * 4]uint32
	NumStmt   [4]uint16
} {
	Pos: [3 * 4]uint32{
		9, 22, 0x250022, // [0]
		22, 25, 0x30025, // [1]
		28, 46, 0x250029, // [2]
		46, 49, 0x30025, // [3]
	},
	NumStmt: [4]uint16{
		2, // 0
		2, // 1
		2, // 2
		2, // 3
	},
}

```

可以看到覆盖的次数始终为 1，可以理解为 set 模式，只关注是否被覆盖，并不关注热点代码被覆盖的次数。

### Atomic 模式埋点代码

我们最后来看看 atomic 模式下重写后的代码以及它的区别：

```shell
go tool cover -mode=atomic about_test.go > atomic_test.go
```

代码如下：

```go
//line about_test.go:1
package demo; import _cover_atomic_ "sync/atomic"

import (
	"testing"

	"github.com/stretchr/testify/require"
)

func TestFirstName(t *testing.T) {_cover_atomic_.AddUint32(&GoCover.Count[0], 1);
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
	}

	for _, testcase := range testcases {_cover_atomic_.AddUint32(&GoCover.Count[1], 1);
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

func TestFirstAndLastName(t *testing.T) {_cover_atomic_.AddUint32(&GoCover.Count[2], 1);
	testcases := []struct {
		description string
		input       string
		expect      string
	}{
		{
			description: "who is xunzhuo",
			input:       "xunzhuo",
			expect:      "first name",
		},
		{
			description: "who is liu",
			input:       "liu",
			expect:      "last name",
		},
	}

	for _, testcase := range testcases {_cover_atomic_.AddUint32(&GoCover.Count[3], 1);
		whoami := WhoAmI(testcase.input)
		require.Equal(t, testcase.expect, whoami)
	}
}

var GoCover = struct {
	Count     [4]uint32
	Pos       [3 * 4]uint32
	NumStmt   [4]uint16
} {
	Pos: [3 * 4]uint32{
		9, 22, 0x250022, // [0]
		22, 25, 0x30025, // [1]
		28, 46, 0x250029, // [2]
		46, 49, 0x30025, // [3]
	},
	NumStmt: [4]uint16{
		2, // 0
		2, // 1
		2, // 2
		2, // 3
	},
}
var _ = _cover_atomic_.LoadUint32

```

可以看到和 count 类似，只是覆盖的次数是通过 `"sync/atomic"` 去安全的增加，在并发场景下，覆盖次数会更加精确。

## 结束语

计算机科学家 Edsger Dijkstra 曾说过：**Program testing can be used to show the presence of bugs, but never to show their absence!** （测试能证明缺陷存在，而无法证明没有缺陷）所以测试不可能是完整的，实现 100% 的测试覆盖率听起来很美，但是在具体实践中通常是不可行的，也不是值得推荐的做法。应该对更需要测试的地方添加测试代码，而不是一味的为每个方法都加入测试代码，经验表明：

+ 高代码覆盖率并不能保证高产品质量，但低代码覆盖率一定说明大部分逻辑没有被自动化测到。后者通常会增加问题遗留到线上的风险，当引起注意。
+ 没有普适的针对所有产品的严格覆盖率标准。实际上这更应该是业务或技术负责人基于自己的领域知识，代码模块的重要程度，修改频率等等因素，自行在团队中确定标准，并推动成为团队共识。
+ 低代码覆盖率并不可怕，能够主动去分析未被覆盖到的部分，并评估风险是否可接受，会更加有意义。

贴一篇 Google 发布的代码覆盖的[最佳实践](https://testing.googleblog.com/2020/08/code-coverage-best-practices.html)，这篇博客提到，其经验表明，重视代码覆盖率的团队通常会更加容易培养卓越工程师文化，因为这些团队在设计产品之初就会考虑可测性问题，以便能更轻松的实现测试目标。而这些措施反过来会促使工程师编写更高质量的代码，更注重模块化.
