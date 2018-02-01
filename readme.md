# Name

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "1.0.0"

[github source](https://github.com/developit/greenlet)

~~[english](./README.en.md)~~

---

我们来说明一下，`worker`的信息传递顺序

 `1.myWorker.postMessage` ->
 
  `worker.js中,  2. onmessage`--> 
 
 `worker.js中, 3. postMessage` --> 
 
 `4. myWorker.onmessage`

[onmessage- 更多](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/onmessage)

``` js
/** Move an async function into its own thread.
 *  @param {Function} fn	The (async) function to run in a Worker.
 */
export default function greenlet(fn) {
	let w = new Worker(URL.createObjectURL(new Blob([
            'onmessage='+( //     ------ 2
                // 👌
				f => ({ data }) => Promise.resolve().then(
					() => f.apply(f, data[1]) // 带着从 -- 1，获得 a-输入变量
				).then(
					d => { postMessage([data[0], null, d]); }, // --- 3
					e => { postMessage([data[0], ''+e]); } //  --- 3
				)
			)+'('+fn+')' // 👌 把定义的函数放入 f，且运行返回-({ data }) 函数
		]))),
		c = 0, // 递增
		p = {}; 
	w.onmessage = ({ data: [c,e,d] }) => { //  --- 4
        p[c][e?1:0](e||d); 
        // 当-e-具有值时，说明具有错误
        // e || d 都代表 f.apply(f, data[1])
        // reject(e)

        // 当 -e- 不具有值时说明
        // 正常运行
        // resolve(f.apply(f, data[1])

		delete p[c]; // 作用完成
	};
	return (...a) => new Promise( (y, n) => {
		p[++c] = [y, n]; // resolve reject
		w.postMessage([c, a]);  //      ---- 1
	});
}
```

结合一下，示例

``` js
import greenlet from 'greenlet'

let get = greenlet(async url => {
	let res = await fetch(url)
	return await res.json()
})

console.log(await get('/foo'))
```

先从定义开始 `fn`

``` js
greenlet(fn) { 
    // fn == 「 async url => {
    //       	let res = await fetch(url)
    //       	return await res.json()
    //         } 」

// 其他不管先
```

`await get('/foo')` 使用

``` js
	return (...a) => new Promise( (y, n) => {
        // a == '/foo'
		p[++c] = [y, n]; // resolve reject
        w.postMessage([c, a]);  //   发送 c 递增顺序与 输入变量值
        // 到 worker 定义的函数-onmesssage
	});
```

进入 `worker-onmessage`

``` js
	let w = new Worker(URL.createObjectURL(new Blob([
            'onmessage='+( //     ------ 2
                // 👌
				f => ({ data }) => Promise.resolve().then(
					() => f.apply(f, data[1]) // 带着从 -- 1，获得 a-输入变量
				).then(
					d => { postMessage([data[0], null, d]); }, // --- 3
					e => { postMessage([data[0], ''+e]); } //  --- 3
				)
			)+'('+fn+')' // 👌 把定义的函数放入 f，且运行返回-({ data }) 函数
		]))),
```

`('+fn+')` 我们先看看 这个括号

`(f => //...)(fn)` 就是带入 `f` == `fn` 并运行

``` js
// 所以
            'onmessage='+( //     ------ 2
                // 👌
				f => ({ data }) => Promise.resolve().then(
					() => f.apply(f, data[1]) // 带着从 -- 1，获得 a-输入变量
				).then(
					d => { postMessage([data[0], null, d]); }, // --- 3
					e => { postMessage([data[0], ''+e]); } //  --- 3
				)
            )+'('+fn+')'
            
// 变

            'onmessage='({ data }) => Promise.resolve().then(
					() => f.apply(f, data[1]) // f == fn
				).then(
					d => { postMessage([data[0], null, d]); },  // --- 3
                    e => { postMessage([data[0], ''+e]); }  //  --- 3
				)
			)
```

下一步，来到 `--- 3`

有两个-3，

`d =>` 和 `e =>` 就像 `Promise(resolve,reject)` 只会运行一个

- 默认正确

> 运行 `d => { postMessage([data[0], null, d]); }`

- 错误❌

> 运行 `e => { postMessage([data[0], ''+e]); }`

---

不管运行哪个 `e||d` === `f.apply(f, data[1])`

并来到 `w.onmessage`

``` js
// c 是递增顺序 , e 是 错误, d 是 默认

// p的值来源于  p[++c] = [y, n];  
// y == resolve
// n == reject

// p[c][1] == reject
// p[c][0] == resolve

w.onmessage = ({ data: [c,e,d] }) => { //  --- 4
        p[c][e?1:0](e||d); 

        // e?1:0
        // 如果-e-具有值时，说明具有错误
        // e || d 都代表 f.apply(f, data[1])
        // reject(e)

        // 当 -e- 不具有值时说明
        // 正常运行
        // resolve(f.apply(f, data[1])

		delete p[c]; // 作用完成, 删除
	};
```

因为 `y,n` 是来自`new Promise( (y, n) =>`

``` js
	return (...a) => new Promise( (y, n) => {
        // a == '/foo'
		p[++c] = [y, n]; // resolve reject
        w.postMessage([c, a]);  //   发送 c 递增顺序与 输入变量值
        // 到 worker 定义的函数-onmesssage
	});
```

`=>` 绑定了 `this`, 所以 `p[c][e?1:0](e||d); `

会返回-示例中结果

``` js
let get = greenlet(async url => {
	let res = await fetch(url)
	return await res.json() // <----
})

console.log(await get('/foo'))
```

像

``` js
	return (...a) => new Promise( (y, n) => {
        y(async url => {
	let res = await fetch(url)
	return await res.json() 
        })
        // or 
        // n(async url => {
        // 	let res = await fetch(url)
        // 	return await res.json() // <----
        // })
	});
```






