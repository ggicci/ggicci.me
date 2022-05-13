---
title: "goplay: Embed Go Playground on your Website"
date: "2022-05-11T00:29:20+08:00"
description: "Run go code snippets on your website by embedding a *Go Playground*."
categories:
  - "go"
tags:
  - goplay
  - hugo
  - hugo-shortcode
  - go-playground
  - go
aliases:
  - /goplay
# draft: true
# weight: 0
# menu: main, side, footer

# theme: PaperMod
cover:
  image: goplay-cover.png
---

This post is introducing a way to run Go snippets on websites. Which leveraged the official Go Playground service at https://go.dev/play/.

## Demo

Let's take a quick look at the two demos below. You can click the `Run` button to see the output of the corresponding program. Click `Try it yourself` or `Share` button to open the code in the official Go Playground to further edit/test the code on your hand.

### Demo 1. Hello World

{{% goplay %}}

```go
package main

func main() {
	println("hello world")
}
```

{{% /goplay %}}

### Demo 2. Import third-party package from GitHub

{{% goplay %}}

```go
package main

import (
	"fmt"
	"net/http"
	"net/http/httptest"

	"github.com/ggicci/httpin" // 3rd-party packages: will be auto-downloaded
	"github.com/justinas/alice"
)

type ListUsersInput struct {
	Page    int64  `in:"query=page;default=1"`
	PerPage int64  `in:"query=per_page;default=20"`
	Token   string `in:"header=x-access-token;required"`
}

func ListUsers(rw http.ResponseWriter, r *http.Request) {
	input := r.Context().Value(httpin.Input).(*ListUsersInput)
	fmt.Printf("input: %#v\n", input)
	rw.WriteHeader(204)
}

func main() {
	r, _ := http.NewRequest("GET", "/?page=3", nil)
	r.Header.Set("X-Access-Token", "secret...")
	handler := alice.New(
		httpin.NewInput(ListUsersInput{}),
	).ThenFunc(ListUsers)
	rw := httptest.NewRecorder()
	handler.ServeHTTP(rw, r)
}
```

{{% /goplay %}}

## How it works?

![Goplay Arch](./goplay-arch.png)

Since we were not able to make AJAX requests directly to [`go.dev`](https://go.dev/play) because of [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) issue.
I setted up a **reverse proxy** `goplay.ggicci.me`, which works as a downstream of `go.dev`. It is responsible to make requests on behalf of the clients to `go.dev` and return the response back to the clients.

I also implemented a simple JS SDK [`@ggicci/goplay`](https://github.com/ggicci/goplay) to easily interact with the go playground service. Use it on your page to connect to your own reverse proxy and run go snippets.

### Set up a reverse proxy to `go.dev`

#### with [**Caddy**](https://caddyserver.com/):

```Caddyfile
goplay.ggicci.me {
    header {
      # https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#cors
      access-control-allow-origin      "https://ggicci.me"
      access-control-allow-headers     "content-type"
      access-control-allow-methods     "OPTIONS,HEAD,GET,POST"
    }

    # only proxy the playground API of go.dev/play
    route /_/* {
      reverse_proxy https://go.dev {
        header_up Host go.dev
      }
    }

    respond 404
}
```

### Use `@ggicci/goplay`

Here's a quick view demonstrating how to use `@ggicci/goplay`.

```html
<body>
  <!-- the container where to show the output -->
  <div id="goplay-output"></div>
  <button onclick="run()">Run Hello World</button>

  <script type="module">
    import { GoPlayProxy } from "https://unpkg.com/@ggicci/goplay/dist/index.js";
    const GOPLAY_PROXY_URL = "https://goplay.ggicci.me"; // use your own proxy URL
    const goplay = new GoPlayProxy(GOPLAY_PROXY_URL);

    function run() {
      const goCodeSnippet = `
        package main
        
        func main() {
          println("hello world")
        }
        `;

      goplay.renderCompile(
        document.getElementById("goplay-output"),
        goCodeSnippet
      );
    }

    window.run = run; // make it visible to the Run button
  </script>
</body>
```

The dead simple SDK just implemented a class `GoPlayProxy` with 3 primary APIs:

```js
class GoPlayProxy {
  // CTOR: specify a proxy url to connect
  constructor(public proxyUrl: string = '/goplay') {}
  // compile source code and get JSON response
  public async compile(sourceCode: string, goVersion?: CompilerVersion): Promise<CompilerResponse>
  // same as `compile`, but renders the JSON response as an `HTMLElement` into the specified container
  public async renderCompile(container: HTMLElement, sourceCode: string, goVersion?: CompilerVersion): Promise<void>
  // get the share URL of the source code
  public async share(sourceCode: string, goVersion?: CompilerVersion): Promise<string>
}
```

Visit https://github.com/ggicci/goplay for more details.

## Hugo Shortcode `goplay`

If your are using [hugo](https://gohugo.io/) to generate your website. This section should be helpful to you.

By using the `goplay` shortcode, you can easily turn your Go code block into a Go playground. Take the following example, in your markdown file, just wrap the code block between `{{%/* goplay */%}}` and `{{%/* /goplay */%}}`.

````markdown
## Sample Code

{{%/* goplay */%}}

```go
package main

func main() {
  println("hello world")
}
```

{{%/* /goplay */%}}
````

The latest code of `goplay - hugo shortcode` used by this site can be found [here](https://github.com/ggicci/ggicci.me/blob/master/layouts/shortcodes/goplay.html). Save it under your hugo directory correspondingly. Typically it is `${YourHugoSiteRoot}/layouts/shortcodes/goplay.html`.

## Who is using `@ggicci/goplay`?

- [httpin Documentation](https://ggicci.github.io/httpin/)
- :plus: add yours here by clicking the [**Suggest Changes**](#top) link under the title.

[^hugo-shortcode]: [Hugo Shortcodes](https://gohugo.io/content-management/shortcodes/).
