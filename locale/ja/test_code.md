# タイトル

```js
const Nuxt = require('nuxt')
const nuxt = new Nuxt()
//保ゲホゲホゲホゲホゲテストテスト
nuxt.renderAndGetWindow('http://localhost:3000')
.then((window) => {
  // ヘッド<title>を表示
  console.log(window.document.title)
})</title>
```