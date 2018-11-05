# タイトル3

```js
const Nuxt = require('nuxt')
const nuxt = new Nuxt()
//保ゲホゲホゲホゲホゲテストテストテストホゲゲ
nuxt.renderAndGetWindow('http://localhost:3000')
.then((window) => {
  // ヘッド<title>を表示
  console.log(window.document.title)
})</title>
```

- #### `runInNewContext`

    - 2.3.0+
    - `createBundleRenderer`内のみで使用
    - Expects: `boolean | 'once'` (`'once'` は2.3.1以上でのみサポート)
