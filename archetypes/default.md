---
title: '{{ replaceRE "(\\d{4}-\\d{2}-\\d{2}-|-)" " " .TranslationBaseName | strings.TrimLeft " " | strings.TrimRight " " | title }}'
date: "{{ .Date }}"
description: ""
thumbnail: ""
categories:
  - ""
tags:
  - ""
# url: relative-url
# aliases:
#   - alias-url-1
# draft: true
# menu: main, side, footer
# weight: 0

# theme: PaperMod
# showToc: false
# searchHidden: true
# cover:
#   image: "<image path/url>"
#   alt: "text"
#   caption: "text"
#   relative: false
# author: ["Me", "You"]

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
