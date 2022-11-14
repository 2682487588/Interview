# Go性能优化

- `CPU profile`:报告程序的CPU使用情况
- `Memory Profile(Heap Profile)`：根据程序的内存使用情况
- `Block Profile`:报告goroutine不在运行的情况，用来分析和查找死锁
- `Goroutine Profiling`:报告goroutine使用情况

## 视频（李文周老师的）

https://www.bilibili.com/video/BV1ZJ411k7UA?p=12&spm_id_from=pageDriver&vd_source=3f9e16e9e8bf5419043354c9e52acc13

## 采集性能数据

- `runtime/pprof`：工具性应用分析
- `net/http/pprof`:服务型应用进行分析

> `pprof`开启，每隔一段时间(10ms),就会收集当下堆栈信息，获取各个函数的CPU已经内存资源，形成性能报告

## 工具型应用

### CPU性能分析

**开始cpu性能分析**

> pprof.StartCPUProfile(w io.Writer)

**结束cpu性能分析**

> pprof.StopCPUProfile()

### 内存性能优化

> pprof.WriterHeapProfile(w io.Writer)

使用采样数据之后,使用`go tool pprof`进行内存性能分析

`go tool pprof`使用 `-inuse_space`统计，可以使用`-inuse-objects`查看分配的对象的数量

## 服务型应用

#### Mux

```go
import _ "net/http/pprof"
```

> 需要手动注册路由规则

```go
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

#### Gin

~~~go
go get github.com/gin-contrib/pprof
~~~

~~~fun
pprof.Register(router)
~~~

![image-20220907153837597](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220907153837597.png)

## go tool pprof命令

~~~bash
go tool pprof [binary] [source]
~~~

- binary数据文件
- source本地来源



## 具体示例

> 先写一段有问题的代码

~~~go
// pprof/main.go
package main

import (
	"flag"
	"fmt"
	"os"
	"runtime/pprof"
	"time"
)

// 一段有问题的代码
func logicCode() {
	var c chan int
	for {
		select {
		case v := <-c:
			fmt.Printf("recv from chan, value:%v\n", v)
		default:

		}
	}
}

func main() {
	var isCPUPprof bool
	var isMemPprof bool

	flag.BoolVar(&isCPUPprof, "cpu", false, "turn cpu pprof on")
	flag.BoolVar(&isMemPprof, "mem", false, "turn mem pprof on")
	flag.Parse()

	if isCPUPprof {
		file, err := os.Create("./cpu.pprof")
		if err != nil {
			fmt.Printf("create cpu pprof failed, err:%v\n", err)
			return
		}
		pprof.StartCPUProfile(file)
		defer pprof.StopCPUProfile()
	}
	for i := 0; i < 8; i++ {
		go logicCode()
	}
	time.Sleep(20 * time.Second)
	if isMemPprof {
		file, err := os.Create("./mem.pprof")
		if err != nil {
			fmt.Printf("create mem pprof failed, err:%v\n", err)
			return
		}
		pprof.WriteHeapProfile(file)
		file.Close()
	}
}
~~~

> 我们加上CPU来进行性能分析`runtime-pprof`可执行文件

~~~go
go build cpuAnalyse.go
//生成cpuAnalyse.exe
cpuAnalyse.exe -cpu=true  //等待二三十秒会生成cpu.pprof文件
~~~

### 命令行交互界面

~~~bash
go tool pprof cpu.pprof
~~~

> 执行上面代码会进行命令行交互界面

~~~cmd
E:\GoStudy\src\Tools\pprof>go tool pprof cpu.pprof
Type: cpu
Time: Sep 7, 2022 at 4:02pm (CST)
Duration: 20.12s, Total samples = 100.64s (500.23%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
~~~

> 我们可以通过 `top3`查看cpu使用前三的函数

~~~cmd
(pprof) top3
Showing nodes accounting for 100.12s, 99.48% of 100.64s total
Dropped 22 nodes (cum <= 0.50s)
      flat  flat%   sum%        cum   cum%
    39.28s 39.03% 39.03%     39.30s 39.05%  runtime.chanrecv
    35.40s 35.17% 74.21%     74.73s 74.25%  runtime.selectnbrecv
    25.44s 25.28% 99.48%    100.46s 99.82%  main.logicCode
(pprof)
~~~

- `flat`：函数占用CPU耗时
- `flat%`:函数占用CPU耗时百分比
- `sum%`：占用CPU累计耗时百分比
- `cum`:当前函数+调用当前函数耗时耗时
- `cum%`:上者的百分比

> `list 函数名`查看对函数详细分析, 执行  `list logicCode`查看对函数的分析

~~~cmd
(pprof) list logicCode
Total: 100.64s
ROUTINE ======================== main.logicCode in E:\GoStudy\src\Tools\pprof\cpuAnalyse.go
    25.44s    100.46s (flat, cum) 99.82% of Total
         .          .      7:   "runtime/pprof"
         .          .      8:   "time"
         .          .      9:)
         .          .     10:
         .          .     11:// 一段有问题的代码
         .       50ms     12:func logicCode() {
         .          .     13:   var c chan int
         .          .     14:   for {
         .          .     15:           select {
    25.44s    100.41s     16:           case v := <-c:
         .          .     17:                   fmt.Printf("recv from chan, value:%v\n", v)
         .          .     18:           default:
         .          .     19:
         .          .     20:           }
         .          .     21:   }
(pprof)
~~~

### 图形化

#### Mac

~~~cmd
brew install graphviz
~~~

#### Windows

下载[graphviz](https://graphviz.gitlab.io/_pages/Download/Download_windows.html) 将`graphviz`安装目录下的bin文件夹添加到Path环境变量中。 在终端输入`dot -version`查看是否安装成功

#### 使用

1.先进入pprof

> go tool pprof cpu.pprof

2.生成图形界面

> web

> 除了CPU性能分析, `pprof`也支持内存性能分析

**图形化界面如下**

![image-20220907165249109](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220907165249109.png)

~~~go
# 查看内存占用数据
go tool pprof -inuse_space http://127.0.0.1:8080/debug/pprof/heap
go tool pprof -inuse_objects http://127.0.0.1:8080/debug/pprof/heap
# 查看临时内存分配数据
go tool pprof -alloc_space http://127.0.0.1:8080/debug/pprof/heap
go tool pprof -alloc_objects http://127.0.0.1:8080/debug/pprof/heap
~~~

## go get

| **参数    | 用法说明                                                     |
| --------- | ------------------------------------------------------------ |
| -d        | 只下载，而不执行创建、安装                                   |
| -t        | 同时下载命令行指定包的测试代码（测试包）                     |
| -u        | 在线下载更新指定的模块（包）及依赖包（默认不更新已安装模块），并创建、安装 |
| -v        | 打印出所下载的包名                                           |
| -insecure | 允许命令在非安全的scheme（如HTTP）下执行get命令              |
| -fix      | 在下载代码包后先执行修正动作，而后再进行编译和安装，根据当前GO版本对所下载的模块（包）代码做语法修正 |
| -f        | 忽略掉对已下载代码包的导入路径的检查                         |
| -x        | 打印输出，get 执行过程中的具体命令                           |

## go-torch和火焰图

> 火焰图是性能分析图标，样子近似火焰得名，我们介绍一个工具`go-torch`

### 安装go-torch

~~~cmd
go get -v github.com/uber/go-torch
~~~

### 特点：

1. 动态的svg
2. 调用顺序从上到下，每个方块代表一个函数（方块大小代表CPU使用长短）

go-torch会先从`http://localhost:8080/debug/pprof/profile`获取profile数据

- -u -url:要访问`URL` 只是主机和端口部分
- -s -suffix:pprof路径，默认为`/debug/pprof/profile`
- -seconds:执行pprof的时间为30s

### 安装 FlameGraph

 











