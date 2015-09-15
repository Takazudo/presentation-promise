<h1 style="line-height:1.4;font-size:3em">オレはあんまよく<br>Promise<br>わかってなかった</h1>

<p style="margin-bottom:0;padding-bottom:0"><a href="https://twitter.com/Takazudo">@Takazudo</a></p>

----

## Promiseとは？

* ES6のアレ
* 似た実装
  * jQueryのDeferredとか
  * AngularJSの$qとか

----

## 基本

```js
var doAsync = function() {
  return new Promise(function(resolve, reject) {
    doAjax({
      success: function(data) {
        resolve(data);
      },
      fail: function() {
        reject('だめだった');
      },
    });
  });
};
```

---

```js
doAsync()
  .then(function(data) {
    console.log(data); // { prop: 'gotVal' }
  }, function(message) {
    alert(message); // だめだった
  });
```

`promise.then(onFulfilled, onRejected)`

----

## チェーンについて

```js
var doAsync1 = function() {
  return new Promise(...);
};

doAsync1()
  .then(doAsync2)
  .then(doAsync3)
  .then(doAsync4)
  .then(doAsync5)
  .then(doSomethingFinally);
```

これってどうなってんの

----

## こういうことだった

```js
var p1 = doAsync1()
var p2 = p1.then(doAsync2)
var p3 = p2.then(doAsync3)
var p4 = p3.then(doAsync4)
var p5 = p4.then(doAsync5)
var p6 = p5.then(doSomethingFinally);
```
ハンドラでPromiseが返されたら  
続きはそのPromiseのthenになる


----

## Promise以外が返されたら？

```js
var normalizeResult = function(data) {
  data.normalized = escapeHtml(data.text);
  return data; // Promiseオブジェクトじゃない
};

doAsync1()
  .then(normalizeResult) // こいつはPromise返さないよ？
  .then(doSomethingFinally);
```

これってどうなってんの

----

## こういうことだった

```js
doAsync1()
  .then(function(result) {
    return new Promise(function(resolve) {
      resolve(normalizeResult(result));
    });
  })
  .then(doSomethingFinally);
```

新しいPromiseが作られて返され、  
渡ってきた値で即resolveされてる

だから同期処理も混ぜられる…！

----

## ちなみに

```js
var dump = function(data) {
  console.log(data); // 出力するだけ
};

doAsync1()
  .then(dump) // 何も返さない
  .then(doSomethingFinally); // でも実行されます
```

これも

----

## こういうこと

```js
var dump = function(data) {
  console.log(data); // 出力するだけ
};

doAsync1()
  .then(function(data) {
    return new Promise(function(resolve) {
      dump(data);
      resolve(); // 引数無しでresolveされる
    });
  })
  .then(doSomethingFinally);
```

これと同じ。だから続く。

----

## エラーハンドラ

```js
doAsync1()
  .then(doAsync2) // ここでrejectされたら
  .then(doAsync3) // スキップ
  .then(doAsync4) // スキップ
  .then(doAsync5) // スキップ
  .then(doSomethingFinally) // スキップ
  .catch(handleError); // キャッチ
```

どうしてこうなる？

----

## こういうことだった

```js
doAsync1()
  .then(doAsync2) // ここでrejectされたら
  .then(doAsync3) // スキップ
  .then(doAsync4) // スキップ
  .then(doAsync5) // スキップ
  .then(doSomethingFinally) // スキップ
  .then(null, handleError); // キャッチ
```

`reject`されると次のonFulfilledハンドラ実行されない。しかし次の`then`には渡される。結果キャッチされるまで渡され続ける。

----

## catchの続き

```js
var handleError = function() {
  console.log('エラーだった!!!');
};

doAsync1()
  .then(doAsync2) // ここでrejectされたら
  .catch(handleError); // キャッチ
  .then(doAsync3) // 実行される…！
  .then(doAsync4) // 実行される…！
  .then(doAsync5) // 実行される…！
  .then(doSomethingFinally) // 実行される…！
```

なんで続き実行されんの

----

## こういうことだった

```js
doAsync1()
  .then(doAsync2) // ここでrejectされたら
  .catch(function() {
    return new Promise(function(resolve) {
      handleError(); // エラーだった!!!
      resolve(); // resolveしてるから
    });
  })
  .then(doAsync3) // 実行される…！
  .then(doAsync4) // 実行される…！
  .then(doAsync5) // 実行される…！
  .then(doSomethingFinally) // 実行される…！
```

onFulfilledハンドラ同様、onRejectedハンドラでも新しいPromiseが作られて即resolveされるのと同じ

なので後続する`then`のonFulfilledハンドラが呼ばれる

----

## もし続けたくないならこう

```js
var handleError = function() {
  console.log('エラーだった!!!');
  return new Promise(function(resolve, reject) {
    reject(); // ここで終わり
  });
};

doAsync1()
  .then(doAsync2) // ここでrejectされたら
  .catch(handleError); // キャッチ
  .then(doAsync3) // 実行されない
  .then(doAsync4) // 実行されない
  .then(doAsync5) // 実行されない
  .then(doSomethingFinally) // 実行されない
```

----

## あとこれ同じ

```js
return new Promise(function(resolve, reject) {
  resolve(1000);
});
```

```js
return Promise.resolve(1000);
```

---

```js
return new Promise(function(resolve, reject) {
  reject(1000);
});
```

```js
return Promise.reject(1000);
```

----

# おわり
