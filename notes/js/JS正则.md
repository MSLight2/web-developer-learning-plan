
### 正则表达式字符表：[MDN正则表达式](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Regular_Expressions)
                          
|字符|含义|
|--------|---------|
|` \ `     | 在非特殊字符之前的反斜杠表示下一个字符是特殊字符，不能按照字面理解 |
|` ^ `     | 匹配输入的开始 |
|` $ `     | 匹配输入的结束 |
|` * `     | 匹配前一个表达式 0 次或多次。等价于 `{0,}` |
|` + `     | 匹配前面一个表达式 1 次或者多次。等价于 `{1,}` |
|` ? `     | 匹配前面一个表达式 0 次或者 1 次。等价于 `{0,1}` |
|` . `     | 默认匹配除换行符之外的任何单个字符     |
|` (x) `   | 会匹配 'x' 并且记住匹配项。其中括号被称为捕获括号(即捕获组) 使用`$1、$2、...、$n `访问捕获组 |
|` (?:x) ` | 匹配 'x' 但是不记住匹配项。这种括号叫作非捕获括号 |
|` x(?=y) `| 匹配'x'仅仅当'x'后面跟着'y'.这种叫做先行断言。例如：`/hello (?=js)/` 仅仅匹配`hello`后面跟着的`js`。而`hello gg`不会匹配 |
| `(?<=y)x`| 匹配'x'仅当'x'前面是'y'.这种叫做后行断言。|
| `x(?!y)` | 仅仅当'x'后面不跟着'y'时匹配'x'，这被称为正向否定查找。 |
|` (?<!y)x `| 仅仅当'x'前面不是'y'时匹配'x'，这被称为反向否定查找。 |
|` x|y `   | 匹配'x'或者'y' |
|` {n} `   | n 是一个正整数，匹配了前面一个字符刚好出现了 n 次。 |
|` {n,} `  | n是一个正整数，匹配前一个字符至少出现了n次 |
|` {n,m} ` | n 和 m 都是整数。匹配前面的字符至少n次，最多m次。如果 n 或者 m 的值是0， 这个值被忽略。 |
|` [xyz] ` | 一个字符集合。匹配方括号中的任意字符，包括转义序列 |
|` [^xyz] `| 一个反向字符集。也就是说， 它匹配任何没有包含在方括号中的字符 |
|` [\b] `  | 匹配一个退格(U+0008)。（不要和\b混淆了。） |
|` \b `    | 匹配一个词的边界。一个词的边界就是一个词不被另外一个"字"字符跟随的位置或者前面跟其他"字"字符的位置，例如在字母和空格之间 |
|` \B `    | 匹配一个非单词边界 |
|` \cX `   | 当X是处于A到Z之间的字符的时候，匹配字符串中的一个控制符。 |
|` \d `    | 匹配一个数字。等价于`[0-9]` |
|` \D `    | 匹配一个非数字字符。等价于`[^0-9]`。 |
|` \f `    | 匹配一个换页符(U+000C) |
|` \n `    | 匹配一个换行符(U+000A) |
|` \r `    | 匹配一个回车符(U+000D) |
|` \s `    | 匹配一个空白字符，包括空格、制表符、换页符和换行符。等价于`[\f\n\r\t\v\u00a0\u1680\u180e\u2000-\u200a\u2028\u2029\u202f\u205f\u3000\ufeff]` |
|` \S `    | 匹配一个非空白字符 |
|` \t `    | 匹配一个水平制表符 (U+0009) |
|` \v `    | 匹配一个垂直制表符 (U+000B) |
|` \w `    | 匹配一个单字字符（字母、数字或者下划线）。等价于`[A-Za-z0-9_]` |
|` \W `    | 匹配一个非单字字符。等价于`[^A-Za-z0-9_]` |
|` \0 `    | 匹配`NULL`（U+0000）字符， 不要在这后面跟其它小数，因为 `\0<digits>` 是一个八进制转义序列。 |

#### 常用正则表达式
- 手机号码：`/^1[3456789][0-9]{9}$/g`，`/^1[0-9]{10}/g`
- 手机平台:
  ```
  let UA = window.navigator.userAgent.toLowerCase()
  let isIOS = (/iphone|ipad|ipod|ios/.test(UA))
  let isAndriod = (UA.indexOf('android') > 0)
  ```
- 匹配`img`标签：`/<img.*?(?:>|\/>)/ig`，`/<img.+?src="([^"]*)"([^>]*)?>/ig`，`/<img align="absmiddle" src="([^"]*)"[^>]+>/ig`
- 链接：`/^((https|http):\/\/)+([\da-vx-z-.]+)\.([\w_.-]*)*/g`
- 去除首尾空格：`val.replace(/^\s\s*/, '').replace(/\s\s*$/, '')`
- 判断`url`是否有特定（id）参数：`/(^|&)id([^&]*)(&|$)/`
- 提取`url`特定参数值：`new RegExp('(^|&)' + name + '=([^&]*)(&|$)')`
- 限制输入两位小数：
  ```js
  // PS：这个有点长~
  val.replace(/[^\d.]/g, '')
    .replace(/^\./g, '').replace(/\.{2,}/g, '.')
    .replace('.', '$#$').replace(/\./g, '')
    .replace('$#$', '.').replace(/^(-)*(\d+)\.(\d\d).*$/, '$1$2.$3').(/^0{2,}/, '0')
  ```
  更多...(后续收集填充)
#### Vue源码模板编译模块的正则
- 元素属性: ``` /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/ ```
- Vue指令、事件绑定属性等：``` /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/ ```
- html标签：`/a-zA-Z\u00B7\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u037D\u037F-\u1FFF\u200C-\u200D\u203F-\u2040\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD/`
```js
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
const startTagClose = /^\s*(\/?)>/
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)
const doctype = /^<!DOCTYPE [^>]+>/i
// #7298: escape - to avoid being passed as HTML comment when inlined in page
const comment = /^<!\--/
const conditionalComment = /^<!\[/

// Special Elements (can contain anything)
export const isPlainTextElement = makeMap('script,style,textarea', true)
const reCache = {}

const decodingMap = {
  '&lt;': '<',
  '&gt;': '>',
  '&quot;': '"',
  '&amp;': '&',
  '&#10;': '\n',
  '&#9;': '\t',
  '&#39;': "'"
}
const encodedAttr = /&(?:lt|gt|quot|amp|#39);/g
const encodedAttrWithNewLines = /&(?:lt|gt|quot|amp|#39|#10|#9);/g
```