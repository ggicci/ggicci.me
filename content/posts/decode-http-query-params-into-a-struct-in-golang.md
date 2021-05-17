---
title: "Decode HTTP Query Params Into a Struct in Golang"
date: "2021-05-17T11:34:14+08:00"
description: "Use httpin package to decode an HTTP request in the way you like."
thumbnail: ""
categories:
  - "go"
tags:
  - "go"
  - "http"
  - "httpin"
  - "go-reflect"
  - "go-struct-tags"
# url: relative-url # aliases:
#   - alias-url-1
# draft: true
# theme: mainroad
# thumbnail: images/placeholder.png
# lead: "Lead text"
# theme: mainroad
---

Many people use `net/http` package directly in Go to deal with HTTP requests, including reading URL parameters, HTTP headers, and the request body. It's straightforward and efficient, though. We can still get bored writing so much tedious code for just reading and parsing the URL params. Typically when we are maintaining a service with hundres of APIs.

## Parse URL Params with `net/http`

Let's see a piece of code for dealing with HTTP requests by using `net/http` package:

```go
// GET /v1/users?page=1&per_page=20&is_member=true
func ListUsers(rw http.ResponseWriter, r *http.Request) {
	page, err := strconv.ParseInt(r.FormValue("page"), 10, 64)
	if err != nil {
		// Invalid parameter: page.
		return
	}
	perPage, err := strconv.ParseInt(r.FormValue("per_page"), 10, 64)
	if err != nil {
		// Invalid parameter: per_page.
		return
	}
	isMember, err := strconv.ParseBool(r.FormValue("is_member"))
	if err != nil {
		// Invalid parameter: is_member.
		return
	}

	// Read database and return the user list.
}
```

It's okay if there are only a few (usually less than 3) parameters to be parsed. But what if more? Both the number of lots of local variables and the writing of statements of parsing string URL params to target types can kill us. Some "evil" guys even spread these statements all over in an API handler. 🤒

For such cases, what can we do to save clean an tidy code base?

### 1. Reduce the number of local variables.

We can define a custom struct to gather all the parameters together as struct fields to **reduce the number of local variables from N to 1**.

```go
type ListUsersInput struct {
	Page     int  `in:"form=page"`
	PerPage  int  `in:"form=per_page"`
	IsMember bool `in:"form=is_member"`
}

input := &ListUserInput{}
```

At this time, the above three local variables `page`, `perPage` and `isMember` became one variable `input`.

However, even if we put all the parameters together, we still can't get rid of writing the parsing stuff code. Which again challenges the code readability.

### 2. Implement a generic function to do the parsing stuff.

Can we implement a function to parse the URL params automatically for us? Once this is feasible, all the redundant parsing stuff code can be removed. The code base will finally become clean and tidy. Sample code:

```go
func ListUsers(rw http.ResponseWriter, r *http.Request) {
	input := &ListUserInput{}
	if err := ParseURLParams(&input); err != nil {
		return
	}

	if input.IsMember // ...
}
```

Now the above three parsing statements `... := strconv.ParseXXX(...)` became one statement `ParseURLParams(&input)`.

So, how to implement this magic function `ParseURLParams`? It's not that hard. Since we have defined a struct. We can utilize [struct tags](https://flaviocopes.com/go-tags/) to guide the parsing algorithm. And use [go reflect](https://pkg.go.dev/reflect) to adapt this algorithm to different types of structs dynamically.

See the implementation details at [httpin](https://github.com/ggicci/httpin) package by Ggicci.

## Use of `httpin`

> **httpin** - HTTP Input for Go - Decode an HTTP request into a custom struct

The only thing you need to do while using `httpin` is: **well define your input struct** for receiving the URL params. You don't need to write parsing code for each parameter by yourself. You only care about the input struct and in which handler it will be used.

Show case:

```go
// 1. Define you input struct
type ListUsersInput struct {
	Page     int  `in:"form=page"`
	PerPage  int  `in:"form=per_page"`
	IsMember bool `in:"form=is_member"`
}

// 2. Bind this input struct with an HTTP handler
func init() {
	http.Handle("/users", alice.New(
		httpin.NewInput(ListUsersInput{}),
	).ThenFunc(ListUsers))
}

// 3. Use the input in the handler, all the parsing stuff are handled by httpin
func ListUsers(rw http.ResponseWriter, r *http.Request) {
	input := r.Context().Value(httpin.Input).(*ListUsersInput)
}
```

Happy? 😝
