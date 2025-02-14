---
layout: page
title: About
description: About DadiBit's blog
---

How is this server simple & accessible?

1. limited network usage:
    - 4.4KB for code syntax highliting CSS
    - 1.8KB for main CSS file
2. Dark mode follows system, implemented via alpha colors
3. Minimal elements style (eg: tables)
4. Strict [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) by default via the `http-equiv` meta tag within the head[^1]
5. Focuses on presenting the content in the most beatiful (yet accesible) way possible
6. No builtin JavaScript <noscript>(you are crazy, but in a positive way!)</noscript>

## Honourable mentions

- [ultimatemotherfuckingwebsite.com](https://ultimatemotherfuckingwebsite.com/)
- [no style, please!](https://riggraz.dev/no-style-please/)
- [mold](https://github.com/yree/mold)

[^1]: This can be useful when hosted on Github Pages