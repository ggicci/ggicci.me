---
title: "Reduce Docker Image Size by Using Multi-stage Builds"
date: 2018-10-13 22:40:20 +0800
# summary: "If you don't take any optimization measures, docker images can easily get large. And in most cases we just wrapped too many inessential things into the images. So, we should take actions to get rid of it."
categories:
  - docker
tags:
  - docker
  - dockerfile
# mainroad
# thumbnail: images/placeholder.png
---

If you don't take any optimization measures, docker images can easily get large. And in most cases we just wrapped too many inessential things into the images. So, we should take actions to get rid of it.

Let's take a look at a common example Dockerfile for building an application written in [Go](https://golang.org/).

```dockerfile
FROM golang:1.10.2-stretch

WORKDIR /go/src/github.com/ggicci/docker-example-server
COPY . .

RUN go install

LABEL \
 me.ggicci.appdemo.image.created="2006-01-02T15:04:05+08:00" \
 me.ggicci.appdemo.image.version="1.0.0"

# (and more)

WORKDIR /app
ENTRYPOINT [ "docker-example-server" ]
EXPOSE 8080
```

After building the image by running command `docker build -t example:1.0.0 .`, we can see the size of the image we get is 800+ MB.

```text
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
example             1.0.0               9daf905d097d        12 seconds ago      801MB
```

However as we know, go programs are statically compiled and linked. We don't need many things in the image to run the final binary:

1. Source code.
2. Go compiling tools (the `go` command).
3. Big linux image.

Then, let's cut them off.

## Use Multi-stage Builds

```dockerfile
# Stage: builder
FROM golang:1.10.2-stretch as builder

WORKDIR /go/src/github.com/ggicci/docker-example-server
COPY . .

RUN go install

# Stage: runner
FROM alpine:3.7

WORKDIR /app
COPY --from=builder /go/bin/docker-example-server /app

LABEL \
  me.ggicci.appdemo.image.created="2006-01-02T15:04:05+08:00" \
  me.ggicci.appdemo.image.version="1.0.0"
  # (and more)

ENTRYPOINT [ "docker-example-server" ]
EXPOSE 8080
```

There are two stages defined in the Dockerfile above:

- `builder`: based on a linux image with go tools installed, we build our application;
- `runner`: based on a tiny linux image [alpine](https://alpinelinux.org/), we copy the application binary from the builder image and discard the builder image.

Now, the image we get is small enough. Which is only 10+ MB.

```text
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
example-2           1.0.0               77488a7264a1        9 seconds ago       11MB
```

Cheers 🎉
