## 网络学习相关笔记
### JSONP
- JSONP
  - JSONP 不是一种数据交换的格式，而是解决跨域问题的一种方案。
- JSONP原理
  - 原理就是利用`script`标签能跨域的特性，实现数据交互
- 如何实现一个JSONP
- JSONP安全问题
- JSONP和CORS的对比
- 其他跨域解决方案
  - Nginx反向代理
  - `postMessage`
  - `document.domain`
### HTTP缓存
### XSS（跨站脚本攻击）
- 什么是XSS攻击：XSS跨站脚本攻击，将一段脚本内容放到目标网站的目标浏览器上解释执行
- XSS防御方案
  - 过滤或转义所有HTTML JS CSS标签
  - 过滤异常字符编码，对异常字符转义
  - VMS 对图片 xss 的防护措施：设置外链图片上传白名单机制，只允许系统信任的图片来源上传。
- 前端防止XSS：使用`innerHtml`。HTML 5 中指定不执行由 innerHTML 插入的 `<script>` 标签
### CSRF/XSRF（跨站请求伪造）
- Cross-site request forgery跨站请求伪造，也被称为"One Click Attack"或者Session Riding，通常缩写为CSRF或者XSRF，是一种对网站的恶意利用。CSRF定义的主语是"请求"，是一种跨站的伪造的请求，指的是跨站伪造用户的请求，模拟用户的操作.
- 防御
  - 操作尽量少使用get
  - 使用验证码
  - 使用token

> 其它安全攻击：越权、短信安全、文件上传安全