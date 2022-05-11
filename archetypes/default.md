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
# weight: 0
# menu: main, side, footer

# theme: PaperMod
# cover:
#   image: ""
---
