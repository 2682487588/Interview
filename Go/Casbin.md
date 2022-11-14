# PERM元模型



## Policy策略

### 构成

- subject(sub访问的实体)

- object(访问的资源)

- action(act访问的方法)

- eft(策略结果，一般为空，为空就是allow的情况)(eft只有两种情况，allow或者deny)

### 书写方式

~~~mysql
[policy_definition]
p = sub,obj,act,(选填eft)
~~~

## Request请求规则

### 构成

- subject(sub访问的实体)
- object(访问的资源)
- action(act访问的方法)

### 书写方式：

~~~mysql
[request_definition]
r = sub, obj, act
//和policy类似，只不过少了eft
~~~

## Matchers匹配规则

### 作用：Request和Policy的匹配规则

### 书写方式:

~~~mysql
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
//满足匹配条件的eft会被返回到effect表达式中和effect表达式进行匹配，看返回的是true还是false
~~~

## Effect影响

### 书写规则：

> 固定为以下几种

| Policy effect                                                | 意义             | 示例                                                         |
| ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ |
| some(where (p.eft == allow))                                 | allow-override   | [ACL, RBAC, etc.](https://casbin.org/docs/en/supported-models#examples) |
| !some(where (p.eft == deny))                                 | deny-override    | [Deny-override](https://casbin.org/docs/en/supported-models#examples) |
| some(where (p.eft == allow)) && !some(where (p.eft == deny)) | allow-and-deny   | [Allow-and-deny](https://casbin.org/docs/en/supported-models#examples) |
| priority(p.eft) \|\| deny                                    | priority         | [Priority](https://casbin.org/docs/en/supported-models#examples) |
| subjectPriority(p.eft)                                       | 基于角色的优先级 | [主题优先级](https://casbin.org/docs/en/supported-models#examples) |

##   role_definition 角色域



~~~mysql
g = _,_ 以角色为基础
g = _,_,_ 以域为基础

[role definition]
g = _,_  //第一个_代表用户,第二个_代表角色
g = 用户,角色
~~~

### 用户模型

> 在RBAC模型中，使用g关联的两个角色，他们变得通用起来
>
> 比如郭g关联alice和data2_admin,可以在policy的sub写alice/data2_admin这样都会通过



![image-20220713144118738](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220713144118738.png)

  

### 多租户模型

~~~mysql

[role_definition] g = _,_,_
域匹配规则:g会接收三个参数
[matchers] m = g(r.sub,p.sub,r.dom)&&r.dom==p.dom&&r.obj==p.obj&&r.act==p.act
~~~

~~~
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
~~~

Policy

~~~bash
p, admin, domain1, data1, read
p, admin, domain1, data1, write
p, admin, domain1, data2, read
p, admin, domain2, data2, read
p, admin, domain2, data2, write

g, alice, admin, domain1
g, bob, admin, domain2
~~~

Request

~~~
alice, domain1, data2, read
~~~

Result

~~~
true
~~~

![image-20220713162535078](https://secondlife.oss-cn-qingdao.aliyuncs.com/img/image-20220713162535078.png)

#### 实例流程：

> 根据规则r.sub和r.dom会先找p.sub，我们在Policy中根据`g`找到alice,admin,domain1中的admin  **也就是说alice和admin是等价的**, 然后我们查policy规则里面，我们发现  根据后面匹配规则 data2 = data2 ,read = read2可以托导出
>
> p,admin,domain1,data2,read这条记录，是存在的，所以返回true



# 实际操作

### 将配置规则存到数据库

~~~bash
go get github.com/casbin/gorm-adapter/v3
~~~

~~~go

func main() {

	a, _ := gormadapter.NewAdapter("mysql", "root:123456@tcp(127.0.0.1:3306)/casbin", true) // Your driver and data source.
	e, _ := casbin.NewEnforcer("./model.conf", a)
	sub := "alice" // 想要访问资源的用户
	obj := "data1" // 将要被访问的资源
	act := "read"  // 用户对资源实施的操作
	added, err := e.AddPolicy("alice", "data1", "read")
	fmt.Println(added)
	fmt.Println(err)
	ok, err := e.Enforce(sub, obj, act)
	if err != nil {
		fmt.Println("食不食油饼")
		fmt.Printf("%s", err)
		return 
	}
	if ok == true {
		fmt.Println("荔枝")
	} else {
		fmt.Println("苏珊")
	}
}
~~~

### 对Policy的增删改查

~~~go
	filterPolicy := e.GetFilteredPolicy(0, "alice")//查询index为0中,名称为alice的元素
	fmt.Println(filterPolicy)
//[[alice data1 read] [alice data2 read]]
//荔枝

~~~

### 添加RBAC的映射



> policy.csv

~~~csv
[role_definition]
g = _,_
[matchers]
m = g(r.sub,p.sub) && r.obj == p.obj && r.act == p.act
//需要向匹配组添加上g(r.sub, p.sub)的关系
~~~

> main.go

~~~go
~~~

