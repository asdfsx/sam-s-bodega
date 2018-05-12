+++
date = "2018-05-12T00:01:07+08:00"
title = "Golang单元测试"
description = "Golang单元测试"
topics = [
  "topic 1",
]
keywords = [
  "golang"
]
tags = [
  "golang"
]
author = "asdfsx"
type = "post"
draft = false

+++

参考几篇文章后，简单的进行了实践。感觉可以作为以后使用 golang 开发时，单元测试的固定套路。  
首先介绍下这几个库  

* goconvey  
  可以作为单元测试框架。提供web界面管理测试，不仅可以看到单元测试的成功失败，还可以看到测试的覆盖率。而且启动goconvey以后，可以在文件修改后，自动进行测试。
  ![](http://ohrdj7osp.bkt.clouddn.com/21AB18E1-5AC0-43E8-8722-A8F98871AC3E-crop1.png)
* gostub
  在测试过程中，根据需求动态修改全局变量的值（比如，使用测试专用的配置文件），根据需求指定某个函数的返回值。（专业术语叫打桩？？？）
* gomock
  专门用来测试接口的工具。可以根据接口定义来生成一个实现了接口的mock结构体。不过这个mock结构体上接口的每个函数返回值都需要根据测试来指定，这个mock结构体，可以在测试中使用。

### goconvey
安装
```
go get github.com/smartystreets/goconvey
```

启动测试框架
```
cd $GOPATH/src/github.com/asdfsx/codility
goconvey
```

通过浏览器访问 `http://127.0.0.1:8080/` 可以看到测试结果。更改上面的地址可以对不同目录下的代码进行测试。

goconvey 可以直接支持 golang 的 testing 模块。但是为了获得更好的输出，可以使用goconvey的api对测试进行包装
```
import(
  . "github.com/smartystreets/goconvey/convey"
  "testing"
)

func TestDB2(t *testing.T){
    ...
    Convey("CreateConnection", t, func(){
        dbConn, err = CreateConnection()
        So(err, ShouldEqual, nil)
    })
    ...
)

```

### gostub
安装
```
go get github.com/prashantv/gostub
```

在测试的过程中，需要根据情况调整配置和一些全局变量，gostub 就是用来做这个的（打桩？）。
```
import (
	"testing"
	. "github.com/prashantv/gostub"
)

var (
	MYSQLUSER string
	MYSQLPASSWORD string
	MYSQLADDR string
	MYSQLPORT int
	DATABASENAME string
)

func Test1(t *testing.T){
	stubs := New()
	stubs.Stub(&MYSQLUSER, "root")
	stubs.Stub(&MYSQLPASSWORD, "root")
	stubs.Stub(&MYSQLADDR, "127.0.0.1")
	stubs.Stub(&MYSQLPORT, 3306)
	stubs.Stub(&DATABASENAME, "mysql")
	defer stubs.Reset()
	connstr := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s", MYSQLUSER, MYSQLPASSWORD, MYSQLADDR, MYSQLPORT, DATABASENAME)
	...
)
```

### gomock
gomock 主要用来进行接口的测试。通过它提供的 mockgen 工具，会根据接口定义来生成接口测试用的文件。配合 gostub 对mock程序的行为进行打桩，然后就可以用来测试了。

安装
```
go get github.com/golang/mock/gomock
go install github.com/golang/mock/mockgen
```

使用 mockgen

```
mockgen -source db.go > mock_db.go 
```

mock_db.go 就是未来接口测试用到的mock类

```
import (
	"testing"
	. "github.com/smartystreets/goconvey/convey"
	. "github.com/prashantv/gostub"
	. "github.com/golang/mock/gomock"
)

func TestMock(t *testing.T){
	var newRedisRepo2 = newRedisRepo
	Convey("test obj demo", t, func() {
		Convey("create obj", func() {
			ctrl := NewController(t)
			defer ctrl.Finish()
			mockRepo := NewMockRepository(ctrl)
			mockRepo.EXPECT().Retrieve(Any()).Return(nil, nil)
			mockRepo.EXPECT().Create(Any(), Any()).Return(nil)
			mockRepo.EXPECT().Retrieve(Any()).Return(nil, nil)
			stubs := StubFunc(&newRedisRepo2, mockRepo)
			defer stubs.Reset()
			So(repo.Create("", nil), ShouldBeNil)
		})
	})
}


```

### 总结
灵活使用以上3个工具应该能满足绝大多数的测试场景。接下来就是多用多看相关文档了。


# 参考文章
[GoConvey框架使用指南](https://www.jianshu.com/p/e3b2b1194830)   
[GoStub框架使用指南](https://www.jianshu.com/p/70a93a9ed186)  
[GoMock框架使用指南](https://www.jianshu.com/p/f4e773a1b11f)  


