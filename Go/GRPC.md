# GRPC使用

## 1.初始化文件

目录结构如下：

![image-20220714214551381](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220714214551381.png)

### 初始化hello_grpc.proto文件

~~~protobuf
syntax = "proto3";

//需要配置package
option go_package = "./;golang";

package hello_grpc;

message Req {
    string message = 1;
}
message Res {
    string message = 1;
}

service HelloGRPC {
    rpc SayHi(Req)
            returns (Res);
}
~~~

### 初始化导包

~~~bash
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
//需要下载到GOPATH/src下面
~~~

### 初始化build.bat文件

> 这个./hello_grpc.proto是你protobuf文件的名字

~~~bat
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./hello_grpc.proto
~~~

### 下载protoc.exe文件，并且放置到GOPATH/src

~~~go
https://github.com/protocolbuffers/protobuf/releases/tag/v3.9.0
//下载解压，放到GOPATH/src下面
~~~

### 执行build.bat

~~~go
//cd到pb   命令行输入 build.bat
~~~

## 2.服务端四步走

### 1.取出server

~~~go
type server struct {
   hello_grpc.UnimplementedHelloGRPCServer
}
~~~

### 2.挂载方法

~~~go
//定义提供响应的方法
func (s *server) SayHi(ctx context.Context, req *hello_grpc.Req) (res *hello_grpc.Res, err error) {
	//输出获取的请求信息
	fmt.Println(req.GetMessage())
	return &hello_grpc.Res{Message: "我是从服务端返回过来的信息"}, nil
}
~~~

### 3.注册服务

  ~~~go
  	//func main中
  	//创建监听
  	l, _ := net.Listen("tcp", ":8888")
  	//创建一个新server
  	s := grpc.NewServer()
  	//注册server
  	hello_grpc.RegisterHelloGRPCServer(s, &server{})
  ~~~

### 4.创建监听

~~~go
	//注册完之后进行服务
	s.Serve(l)
~~~

## 3.客户端四步走

### 1.创建连接

~~~go
conn, err := grpc.Dial("localhost:8888", grpc.WithInsecure())
~~~

### 2.new一个Client

~~~go
	//注册conn这个接口，返回一个可用的client
	client := hello_grpc.NewHelloGRPCClient(conn)
~~~

### 3.调用client方法

~~~go
//调用sayHi方法
req, err := client.SayHi(context.Background(), &hello_grpc.Req{Message: "我从客户端过来"})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
~~~

### 4.获取返回值

~~~go
	fmt.Println(req.GetMessage())
~~~

# Protobuf的基础使用

## 目录

![image-20220715150237855](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220715150237855.png)

> 定位  pb/person/person.proto

~~~protobuf
syntax = "proto3";  // 1.我们需要使用proto3解读

package person;//2.创建包目录，为了能通过package.xxx引用

//3.当前包路径下的person文件夹(;后面跟的是别名)
//module是github/pixel/pb
option go_package = "github/pixel/pb/person;person";

//引入Home
import "home/home.proto";

//4.定义message结构
message Person {
    string name = 1;
    int32 age = 2;
    //    bool sex = 3;
    //5.声明一个切片
    repeated string test = 4;
    //6.声明一个map
    map <string, string> test_map = 5;
    //使用reserved声明保留字(就是我虽然现在不用，但是我以后可能用，你这个字就先不要用)

    //声明枚举类型
    enum SEX {
        //可能有一些相同的value,这时候我们需要设置alias
        //        option allow_alias = true;
        Male = 0;
        //        Boy = 0;
        FaMale = 1;
        //        Girl = 1;
    }
    SEX sex = 3;

    //Oneof
    oneof TestOneOf {
        string one = 6;
        string two = 7;
        string three = 8;
    }

    //调用package
    home.Home myHome = 9;

}

//6.声明一个切片类型的Person
//message  Home {
//    repeated Person persons = 1;
//    message  V {
//        string name = 1;
//    }
//    //使用这个V
//    V v = 2;
//}

~~~

> build.bat

```bat
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./person/person.proto
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./home/home.proto
```

> home.proto

~~~protobuf
syntax = "proto3";  // 1.我们需要使用proto3解读

package home;//2.创建包目录，为了能通过package.xxx引用

//3.当前包路径下的person文件夹(;后面跟的是别名)
option go_package = "github/pixel/pb/home;home";

message Home {
    string home_num = 1;
}
~~~

## 定义服务

> 定位 person.proto

~~~protobuf
service SearchService {
    rpc Search(Person) returns(Person); //传统的  即可响应
    rpc SearchIn(stream Person) returns(Person); //入参为流
    rpc SearchOut(Person) returns(stream Person); //出参为流
    rpc SearchIO(stream Person) returns(stream Person); //出入为流
}
~~~

