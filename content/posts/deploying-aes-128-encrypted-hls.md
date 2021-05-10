---
title: "Deploying an AES-128 Encrypted HTTP Live Stream (HLS)"
date: "2021-05-10T23:07:33+08:00"
description: ""
thumbnail: ""
categories:
  - "streaming"
tags:
  - "streaming"
  - "vod"
  - "hls"
  - "ffmpeg"
# url: relative-url
# aliases:
#   - alias-url-1
# draft: true
# menu: main, side, footer

# theme: mainroad
# thumbnail: images/placeholder.png
# lead: "Lead text"
# comments: true # enable disqus comments for specific page
# authorbox: true # enable authorbox for specifc page
# pager: true # enable pager navigation (prev/next) for specific page
# toc: true # enable Table of Contents for specific page
# mathjax: true # enable MathJax for specific page
# sidebar: "right" # enable sidebar, opts: left, right
# widgets: [ "recent", "categories", "taglist", "social", "languages" ] # enable sidebar widgets in given order
# theme: mainroad
---

**HTTP Live Streaming** (HLS) is an HTTP-based adaptive bitrate streaming communications protocol developed by Apple Inc. and released in 2009.

Some key points of HLS:

0. Created by Apple.
1. Consists of a manifest file (e.g. playlist.m3u8) and chunks of video segment files (e.g. seq0.ts).
2. H264 codec of video + AAC of audio.
3. Use HTTP, easily leveraging CDN to reach the widest audience without worring about the bandwidth and firewalls.
4. Adaptive streaming, enables changing the quality of the video mid-stream.
5. Widely supported across devicies and platforms. PC, mobile, Web, IOS, Andorid, etc.

Now, let's deploy an HLS with [Caddy](https://caddyserver.com/).

## Architecture

![AES-128 Encrypted HLS Architecture](/images/hls-arch.png)

TODO(@ggicci): ...

## References

- [Apple Developer - HTTP Live Streaming](https://developer.apple.com/streaming/)
- [HLS Encryption: How to Encrypt Videos in AES-128 For HTTP Live Streaming [2021 Update]](https://www.dacast.com/blog/hls-encryption-for-video/)
- [Apple FPS - FairPlay Streaming](https://developer.apple.com/streaming/fps/)
