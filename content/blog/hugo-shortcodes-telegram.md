---
title: "Write a Hugo Shortcodes to embed Telegram post"
date: 2021-09-06T01:13:52+08:00
draft: false
ShowToc: true
tags: ["WeChat"]
categories: []
---

Create your own Shortcodes to Embed [Telegram](telegram.org) post.

<!--more-->

> You can extend Hugo’s built-in shortcodes by creating your own using the same templating syntax as that for single and list pages.
>
> Hugo documents

### File Location

> To create a shortcode, place an HTML template in the `layouts/shortcodes` directory of your [source organization](https://gohugo.io/getting-started/directory-structure/#directory-structure-explained). Consider the file name carefully since the shortcode name will mirror that of the file but without the `.html` extension.

## telegramPost.html

Create `telegramPost.html` inside `layouts/shortcodes`

```html
<div id="tgPost">
  <script type="text/javascript"></script>
  <script
    async
    src="https://telegram.org/js/telegram-widget.js?15"
    data-telegram-post="{{ .Get 0 }}/{{ .Get 1}}"
    data-width="100%"
  ></script>
</div>
```

`layouts/shortcodes/telegramPost.html` will be called with`{{</* telegramPost channelName PostId */>}}`

For instance:

`{{</* telegramPost programmerjokes 2246 */>}}`

{{< telegram programmerjokes 2246>}}

## Reference

[Create Your Own Shortcodes](https://gohugo.io/templates/shortcode-templates/)

[A way to mark plain-text and stop hugo from interpreting?](https://discourse.gohugo.io/t/a-way-to-mark-plain-text-and-stop-hugo-from-interpreting/1325)
