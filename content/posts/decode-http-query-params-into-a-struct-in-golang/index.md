---
title: "Decode HTTP Query Params into a Struct in Go"
date: "2021-05-17T11:34:14+08:00"
description: "Use httpin package to decode HTTP request params into a struct in Go."
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

Many people use `net/http` package directly in Go to deal with HTTP requests, including reading URL parameters, HTTP headers, and the request body. It's straightforward and efficient, though. We can still get bored writing so much tedious code for just reading and parsing the URL params. Especially when we were maintaining a service with hundres of APIs.

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

It's okay if there are only a few (usually less than 3) parameters to be parsed. But what if more? Both the number of lots of local variables and the writing of statements of parsing string URL params to target types can kill us. Someone even spreads these statements all over in an API handler. 🤒

For such cases, what can we do to save a clean and tidy code base?

## Using `ggicci/httpin`

> [**httpin**](https://github.com/ggicci/httpin) - 🍡 HTTP Input for Go - Decode an HTTP request into a custom struct

**ggicci/httpin** is an [awesome](https://github.com/ggicci/awesome-go#forms) package which helps you easily decoding HTTP reqeusts from:

- Query string (URL parameters), e.g. `?name=john&is_member=true`
- Headers, e.g. `Authorization: xxx`
- Form data, e.g. `login=john&password=*****`
- JSON/XML body, e.g. `POST {"name": "john", "is_member": true}`
- Path variables, e.g. `/users/{username}`
- File uploads

You don't need to write parsing code for any parameter by yourself. You only care about the input struct and in which handler it will be used.

Let's see an example (working with `net/http`):

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

**httpin** is:

- **well documentated**: at https://ggicci.github.io/httpin/.
- **well tested**: [over 98% in coverage](https://codecov.io/gh/ggicci/httpin)
- **open integrated**: with [net/http](https://ggicci.github.io/httpin/integrations/http), [go-chi/chi](https://ggicci.github.io/httpin/integrations/gochi), [gorilla/mux](https://ggicci.github.io/httpin/integrations/gorilla), [gin-gonic/gin](https://ggicci.github.io/httpin/integrations/gin), etc.
- **extensible** (advanced feature): by adding your custom directives. Read [httpin - custom directives](https://ggicci.github.io/httpin/directives/custom) for more details.

You will find that with the help of **httpin**, you can get the following benefits:

- ⌛️ Saving developer time
- ♻️ Lower code repetition rate
- 📖 Higher readability
- 🔨 Higher maintainability

❤️ Have a try with it :)
