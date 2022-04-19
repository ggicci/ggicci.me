---
title: "Go 自动提取 HTTP 请求参数到结构体中"
date: "2021-05-17T11:34:14+08:00"
description: "使用 httpin 包自动提取/映射/绑定 HTTP 请求参数到自定义的结构体中。"
thumbnail: ""
aliases:
  - "decode-http-query-params-into-a-struct-in-golang"
  - "httpin"
categories:
  - "go"
tags:
  - "go"
  - "http"
  - "httpin"
  - "go-reflect"
  - "go-struct-tags"

# PaperMod
cover:
  image: httpin-cover.png
---

## 引入

在 Go 中，我们可以直接使用 Go 自带的 `net/http` 包对 HTTP 请求参数进行解析。包括读取 URL 参数、读取 HTTP 头、读取 HTTP 请求体。比如下面的一个简单的例子：

```go
// GET /v1/users?page=1&per_page=20&is_member=true
func ListUsers(rw http.ResponseWriter, r *http.Request) {
    page, err := strconv.ParseInt(r.FormValue("page"), 10, 64)
    if err != nil {
        // 处理参数错误: page.
        return
    }
    perPage, err := strconv.ParseInt(r.FormValue("per_page"), 10, 64)
    if err != nil {
        // 处理参数错误: per_page.
        return
    }
    isMember, err := strconv.ParseBool(r.FormValue("is_member"))
    if err != nil {
        // 处理参数错误: is_member.
        return
    }

    // 读取数据库并返回给客户端
}
```

这段代码看上去没什么毛病，但是隐藏了几个问题需要我们思考：

- **Q1**. 这个 API 处理了 3 个参数，同时**带来了 3 个局部变量**。如果我们的 API 需要处理 7、8 个参数呢？
- **Q2**. 除了这个 API，假设我们同时需要**开发或者维护上百个 API** 呢？

## 避免局部变量过多

对于 **Q1**，我们能确定的一点是，局部变量的增多会导致程序的可读性和可维护性变差。不过这个问题也容易解决。我们可以定义一个结构体（struct），把所有需要处理的参数定义成这个结构体中的字段即可。这样我们只需要实例化一个结构体就可以了，这样就 **避免了过多局部变量的产生**。比如：

```go
type ListUsersInput struct {
	Page     int
	PerPage  int
	IsMember bool
}

input := &ListUsersInput{} // 只剩下 1 个局部变量
```

## 参数解析代码的提炼与复用

对于 **Q2**，就算我们使用了一个局部变量的结构体来存储参数，也逃不过 **从 HTTP 请求中解析每个参数（即结构体中每个字段）** 的过程。如下：

```go
// GET /v1/users?page=1&per_page=20&is_member=true
func ListUsers(rw http.ResponseWriter, r *http.Request) {
    var err error
    input := &ListUsersInput{} // 只剩下 1 个局部变量
    input.Page, err = strconv.ParseInt(r.FormValue("page"), 10, 64)
    if err != nil {
        // 处理参数错误: page.
        return
    }
    input.PerPage, err = strconv.ParseInt(r.FormValue("per_page"), 10, 64)
    if err != nil {
        // 处理参数错误: per_page.
        return
    }
    input.IsMember, err = strconv.ParseBool(r.FormValue("is_member"))
    if err != nil {
        // 处理参数错误: is_member.
        return
    }

    // 读取数据库并返回给客户端
}
```

上面的代码，我们处理 3 个参数，但是我们依旧需要提取每个参数的值，并且把它们放到结构体中。**如果这部分代码不能够复用，将会是个很严重的问题。** 假设我们解决了参数解析代码复用的这个问题，那么在开发 API 的时候，我们就不需要重复写这个代码了。毕竟给每个参数都写一段解析和错误处理的代码会把代码的可读性和可维护性都降低。同时也能给宝宝们心灵上产生沉重的打击，开始对自己作为一个天才程序员的存在产生怀疑 :monocle_face:。

## Go Reflection 解决参数解析代码无法复用的问题

如果大家有在 Go 里面使用 `json` 包处理过 JSON 的编码解码问题。那么我们肯定熟悉下面的这段代码：

```go
type ListUsersInput struct {
	Page     int  `json:"page"`
	PerPage  int  `json:"per_page"`
	IsMember bool `json:"is_member"`
}
```

Go 自带的 `json` 包，通过结构体中的 [**结构体标签 (struct tag)**](https://pkg.go.dev/reflect#StructTag) 的内容来决定如何解析 JSON 字符串。这部分逻辑是依赖 [Go Reflection](https://go.dev/blog/laws-of-reflection) 来实现的。

通过反射，我们可以获取结构体的字段信息，比如 `ListUsersInput` 中的 `Page` 字段，我们可以获取：

- 字段名称：`Page`
- 字段类型：`int`
- 字段标签：`json:"page"`

对于 `json` 包来讲，这些内容已经足够让程序知道如何去解析 JSON 字符串了。

同样的，我们也可以借助这些信息来写出一个算法，**实现从一个 HTTP 请求中解析出结构体中每个字段的值**。

```go
package main

import (
	"fmt"
	"reflect"
)

type ListUsersInput struct {
	Page     int  `json:"page"`
	PerPage  int  `json:"per_page"`
	IsMember bool `json:"is_member"`
}

func main() {
	input := &ListUsersInput{}
	rt := reflect.TypeOf(*input)
	for i := 0; i < rt.NumField(); i++ {
		field := rt.Field(i)
		fmt.Printf("%d, Name: %s, Tag: %q\n", i, field.Name, field.Tag)
	}
}
```

## 使用 `ggicci/httpin`

> [**httpin**](https://github.com/ggicci/httpin) - 🍡 HTTP Input for Go - Decode an HTTP request into a custom struct

**ggicci/httpin** 是一个被 [awesome](https://github.com/ggicci/awesome-go#forms) 项目提及的项目。**httpin** 可以帮助你从 HTTP 请求中自动地提取各类参数：

- 请求参数 (URL 参数)，也就是 URL 问号后面带的参数，如 `?name=john&is_member=true`
- 请求头参数，比如 `Authorization: xxx`
- 表格数据，如 `login=john&password=*****`
- JSON/XML 数据包， 如 `POST {"name": "john", "is_member": true}`
- 路径变量参数，如 `/users/{username}`
- 上传文件，如图片、视频等

你不需要自己写任何解析的代码，只需要关心两件事情：

1. 定义一个结构体，并标明每个字段从哪里提取参数，参数名是什么
2. 在哪个服务器处理请求的方法中需要用到这个结构体

让我们看一个例子（和 `net/http` 包配合使用）：

```go
// 1. 定义你的结构体
type ListUsersInput struct {
	Page     int  `in:"form=page"`
	PerPage  int  `in:"form=per_page"`
	IsMember bool `in:"form=is_member"`
}

// 2. 绑定这个结构体到你的请求处理函数 (handler)
func init() {
	http.Handle("/users", alice.New(
		httpin.NewInput(ListUsersInput{}),
	).ThenFunc(ListUsers))
}

// 3. 直接使用一行语句取得请求数据，httpin 已经帮你把结构体自动填充好了
func ListUsers(rw http.ResponseWriter, r *http.Request) {
	input := r.Context().Value(httpin.Input).(*ListUsersInput)
}
```

**httpin** 的现状:

- **完善的文档（目前只有英文，欢迎翻译）**：https://ggicci.github.io/httpin/
- **超高的测试覆盖率**：[98% 以上](https://codecov.io/gh/ggicci/httpin)
- **开放集成**：集成了 [net/http](https://ggicci.github.io/httpin/integrations/http)，[go-chi/chi](https://ggicci.github.io/httpin/integrations/gochi)，[gorilla/mux](https://ggicci.github.io/httpin/integrations/gorilla)，[gin-gonic/gin](https://ggicci.github.io/httpin/integrations/gin) 等。
- **可扩展**（高级功能）：通过添加自定义的指令（directive）可以实现。 详情请阅读 [httpin - custom directives](https://ggicci.github.io/httpin/directives/custom)。

你会发现使用了 **httpin** 之后，你：

- ⌛️ 节省了很多开发时间
- ♻️ 降低了代码重复率
- 📖 代码库可读性更好了
- 🔨 代码库可维护性更好了

❤️ 请放心食用 ❤️

爱我就给我一个小星星嗯
