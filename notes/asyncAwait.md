### generator基础用法
```js
function * foo() {
  console.log('start')
  const res = yield 'foo';
  console.log(res);
}

const generator = foo();
const result = generator.next();
console.log(result)

// 通过next传入值，作为yield语句返回值
generator.next('yield value');
// 抛出异常
generator.throw(new Error('generator error'));
```

### 简易async await
```js
function * main(request) {
  try {
    const res = yield request('url');
    console.log(res);
    const res2 = yield request('url2');
    console.log(res2);
  } catch (error) {
    console.log(error)
  }
}

// 生成器函数执行器
function co(generator) {
  const g = generator();
  handlerResult(g.next());

  function handlerResult(result) {
    if (result.done) return; // 生成器函数介绍
    result.value.then((val) => {
      handlerResult(g.next(val));
    }, (error) => {
      g.throw(error);
    });
  }
}

co(main);
```
