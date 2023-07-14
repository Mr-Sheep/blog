---
title: "LoveIt 主題無法渲染修復"
date: 2020-10-05T23:56:04+08:00
draft: false
---

LoveIt是一個非常漂亮 + 實用的Hugo主題，但是作者長期沒有更新，導致在hugo v0.75.0以上版本渲染時報錯
<!--more-->

錯誤內容：
```bash
❯ hugo
Start building sites …
ERROR 2020/10/05 23:28:46 render of "page" failed: execute of template failed: template: _default/single.html:18:124: executing "content" at <partial "function/content.html">: error calling partial: "/Users/peterlu/Documents/Dev/blog/themes/LoveIt/layouts/partials/function/content.html:4:19": execute of template failed: template: partials/function/content.html:4:19: executing "partials/function/content.html" at <partial "function/ruby.html" $content>: error calling partial: partial that returns a value needs a non-zero argument.
ERROR 2020/10/05 23:28:46 render of "page" failed: execute of template failed: template: _default/single.html:18:124: executing "content" at <partial "function/content.html">: error calling partial: "/Users/peterlu/Documents/Dev/blog/themes/LoveIt/layouts/partials/function/content.html:4:19": execute of template failed: template: partials/function/content.html:4:19: executing "partials/function/content.html" at <partial "function/ruby.html" $content>: error calling partial: partial that returns a value needs a non-zero argument.
Total in 418 ms
Error: Error building site: failed to render pages: render of "page" failed: execute of template failed: template: _default/single.html:18:124: executing "content" at <partial "function/content.html">: error calling partial: "/Users/peterlu/Documents/Dev/blog/themes/LoveIt/layouts/partials/function/content.html:4:19": execute of template failed: template: partials/function/content.html:4:19: executing "partials/function/content.html" at <partial "function/ruby.html" $content>: error calling partial: partial that returns a value needs a non-zero argument.
```

<br>需要修改的文件：`layouts/partials/function/content.html`
在源文件第一行後和最後一行之前加入相應內容<br>

```html
{{- $content := .Content -}}

加入的代碼 =>
{{- if ne "" $content -}}

{{- if .Ruby -}}
    {{- $content = partial "function/ruby.html" $content -}}
{{- end -}}
@@ -16,4 +18,6 @@

{{- $content = partial "function/escape.html" $content -}}

加入的代碼 =>
{{- end -}}

{{- return $content -}}
```
也可以直接將submodule項目地址更換爲 `https://github.com/Mr-Sheep/LoveIt/`

參考：
https://github.com/dillonzq/LoveIt/issues/518