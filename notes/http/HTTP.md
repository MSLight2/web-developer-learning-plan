## 网络学习相关笔记
### JSONP
- JSONP
  - JSONP 不是一种数据交换的格式，而是解决跨域问题的一种方案。
- JSONP原理
  - 原理就是利用`<script>`标签能跨域的特性，通过函数调用的方式实现数据交互。核心就是动态添加`<script>`标签来调用服务器提供的js脚本
- 如何实现一个JSONP
  ```js
  // ------------------JSONP的简单实现-------------
  // 客户端代码
  function getremotedata(data) {
    console.log(data)
  }
  var div = document.getElementsByTagName('div')

  div[0].onclick = function(){
    // 远程js。此处实际一般是个接口url，访问后由服务端根据callback名称生成一个对回调函数的调用，传入查询结果，
    // 然后通过 <script> 加载到客户端执行。客户端本地定义该callback就可以获取数据了
    var url = "/getdata.js"
    var script = document.createElement('script')
    script.setAttribute('src', url)
    document.getElementsByTagName('head')[0].appendChild(script)
  }

  // 远程getdata.js
  getremotedata({
    code:0,
    result:'success'
  })
  ```
  jsonp请求的流程图

  ![](https://github.com/MSLight2/web-developer-learning-plan/blob/master/notes/img/jsonp.png)
  [图片来源](https://segmentfault.com/a/1190000009773724)
- JSONP安全问题
  - 因为callback后接的是字符串，是拼接的，所以callback参数可能会恶意添加标签（如script），造成XSS漏洞
  - JSON劫持
  - 解决方式：
    - 严格定义Content-Type: application/json
    - 过滤callback函数敏感字符，单词
    - 服务端做一次urlencode，使用`edecodeURIComponent`处理。处理不符合的字符：如`<` `>`
    - 严格限制对JSONP输出callback函数名的长度
    - 添加回调函数白名单
- JSONP和CORS的对比
  - JSONP只支持GET请求不支持POST请求
  - JSONP没有同源限制问题
  - JSONP兼容性更好
  - JSOPN没有请求的状态码返回
  - 与CROS的主要区别在于JSON的核心是通过HTTP来动态添加 `<script>` 标签来调用服务器提供的js脚本。
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