# データのプリフェッチと状態

## データストア

SSRでは、基本的にはアプリケーションの「スナップショット」をレンダリングしています。そのため、アプリケーションが非同期データに依存している場合**、レンダリングプロセスを開始する前**に**これらのデータをプリフェッチして解決する必要があります** 。

別の懸念事項は、クライアントサイドアプリケーションをマウントする前に、クライアント側で同じデータを利用できる必要があることです。そうしないと、クライアントアプリケーションが別の状態でレンダリングされ、ハイドレーションが失敗します。

これに対処するには、取得したデータをビューコンポーネント、専用データストア、または「状態コンテナ」の外部に配置する必要があります。サーバー上でレンダリングする前に、データをプリフェッチしてストアに格納することができます。さらに、HTMLの状態をシリアライズしインライン化します。クライアント側のストアは、アプリケーションをマウントする前に直接インライン状態を取得することができます。

我々は、この目的のために公式の州管理ライブラリ[Vuex](https://github.com/vuejs/vuex/)を使用する予定である。 idに基づいてアイテムを取得するためのいくつかの嘲笑されたロジックを持つ`store.js`ファイルを作成しましょう：

```js
// store.js import 'vue'からVueをインポートする 'vuex'からVuexをインポートするVue.use（Vuex）//プロミスを返す汎用APIがあると仮定//実装の詳細を無視するimport {fetchItem} from '/api {fetchItem（{commit}、id）{// store.dispatch（）を介してPromiseを返します。私たちが知っている//データがフェッチされたときfetchItem（id）.then（item => {commit（ 'setItem'、{id、item}）}）}}、突然変異：{setItem（state、{id、item }}）{Vue.set（state.items、id、item）}}}}}
```

`app.js`更新する：

```js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'
export function createApp () {
  // create router and store instances
  const router = createRouter()
  const store = createStore()
  // sync so that route state is available as part of the store
  sync(store, router)
  // create the app instance, injecting both the router and the store
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })
  // expose the app, the router and the store.
  return { app, router, store }
}
```

## コンポーネントによる論理コロケーション

だから、データ取得アクションをディスパッチするコードはどこに置くのですか？

フェッチする必要があるデータは、訪問先のルートによって決定されます。また、どのコンポーネントがレンダリングされるかが決まります。実際、所与のルートに必要なデータは、そのルートでレンダリングされたコンポーネントによって必要とされるデータでもあります。したがって、データ取り出しロジックをルートコンポーネント内に配置するのは当然のことです。

私たちはルートコンポーネントにカスタム静的関数`asyncData`を公​​開します。コンポーネントがインスタンス化される前に、この関数が呼び出されますので注意してください、それはへのアクセス権がありません`this` 。ストアとルートの情報を引数として渡す必要があります。

```html
<！ -  Item.vue  - > <template> <div> {{item.title}} </ div> </ template> <script>デフォルトをエクスポートする{asyncData（{store、route}）{//アクションを返すstore.dispatch（ 'fetchItem'、route.params.id）}、computed：{//アイテムをストア状態から表示します。 item（）{これを返す$ store.state.items [this。$ route.params.id]}}} </ script>
```

## サーバーデータフェッチ

`entry-server.js`では、 `router.getMatchedComponents()`でルートに一致するコンポーネントを取得し、コンポーネントがそのコンポーネントを公開している場合は`asyncData`を呼び出します。次に、解決された状態をレンダリングコンテキストにアタッチする必要があります。

```js
// entry-server.js
import { createApp } from './app'
export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()
    router.push(context.url)
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }
      // call `asyncData()` on all matched route components
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // After all preFetch hooks are resolved, our store is now
        // filled with the state needed to render the app.
        // When we attach the state to the context, and the `template` option
        // is used for the renderer, the state will automatically be
        // serialized and injected into the HTML as `window.__INITIAL_STATE__`.
        context.state = store.state
        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

`template`を使用`template` 、 `context.state`は自動的に最終HTMLに`window.__INITIAL_STATE__`状態として埋め込まれ`window.__INITIAL_STATE__` 。クライアントでは、ストアはアプリケーションをマウントする前に状態を取得する必要があります。

```js
（ウィンドウ.__ INITIAL_STATE__）{store.replaceState（ウィンドウ.__ INITIAL_STATE__）}
```

## クライアントデータフェッチ

クライアントでは、データフェッチを処理する2つの異なる方法があります。

1. **ルートナビゲーションの前にデータを解決する：**

この戦略を使用すると、受信ビューで必要なデータが解決されるまで、アプリは現在のビューにとどまります。メリットは、入力されたビューが準備ができたら完全なコンテンツを直接レンダリングできることですが、データのフェッチに時間がかかる場合、ユーザーは現在のビューに「つまらない」と感じます。したがって、この戦略を使用する場合は、データローディングインジケータを提供することをお勧めします。

一致したコンポーネントをチェックし、グローバルルートフックの中で`asyncData`関数を呼び出すことによって、この戦略をクライアントに実装できます。最初のルートが準備された後にこのフックを登録して、サーバがフェッチしたデータを不必要にフェッチしないようにしてください。

```js
// entry-client.js // ...無関係なコードを省略router.onReady（）=> {// asyncDataを処理するためのルータフックを追加する//初期ルートの後に行うことで、私たちが既に持っているデータをフェッチする//すべての//非同期コンポーネントが解決されるように `router.beforeResolve（）`を使用するrouter.beforeResolve（（to、from、next）=> {const matched = router.getMatchedComponents ）const prevMatched = router.getMatchedComponents（from）//以前にレンダリングされなかったコンポーネントだけを気にします。//したがって、2つの一致リストが異なるまでそれらを比較します。let diffed = false const activated = matched.filter（（c、i if（！activated.length）{return next（）} //ここにはローディングインジケータをトリガする必要がありますPromise.all（activate.map（c => {if（c.asyncData）{return c.asyncData（{store、route：to}）}}））then then（（）=> {//ロードを停止するインジケータnext（）}）catch（next）}）app $ mount（ '＃app'）}）
```

1. **一致したビューがレンダリングされた後にデータをフェッチする：**

この戦略は、クライアント側のデータフェッチロジックをビューコンポーネントの`beforeMount`関数に`beforeMount`ます。これにより、ルートナビゲーションがトリガーされたときにビューが即座に切り替わることができるので、アプリの反応がより速くなります。ただし、レンダリングされたときに、受信したビューには完全なデータはありません。したがって、この戦略を使用するビューコンポーネントごとに条件付きロード状態が必要です。

これはクライアントのみのグローバルmixinで実現できます：

```js
Vue.mixin（{beforeMount（）{const {asyncData} = this。$ options if（asyncData）{// fetchオペレーションを約束するように//コンポーネント内で `this.dataPromise.then（..データが準備完了した後に他のタスクを実行するthis.dataPromise = asyncData（{store：this。$ store、route：this。$ route}）}}}）
```

2つの戦略は最終的にはUXの決定が異なり、作成しているアプリケーションの実際のシナリオに基づいて選択する必要があります。ただし、どの戦略を選択しても、 `asyncData`関数は、ルートコンポーネントが再利用されたときに呼び出される必要があります（同じルートですが、paramsまたはqueryが`user/1`から`user/2`変更されます）。クライアントのみのグローバルミックスインでもこれを処理できます：

```js
（asyncData）{asyncData（{store：this。$ store、route：to}）。then（次の）.catch （next）} else {next（）}}}）
```

## 店舗コードの分割

大規模なアプリケーションでは、Vuexストアが複数のモジュールに分割される可能性があります。もちろん、これらのモジュールを対応するルートコンポーネントチャンクにコード分割することもできます。次のストアモジュールがあるとします。

```js
重要：stateは、モジュールが//複数回インスタンス化できるようにする関数でなければなりません。state（（）=>（count：0））、actions ：{inc：（{コミット}）=>コミット（ 'inc'）}、突然変異：{inc：state => state.count ++}}
```

`store.registerModule`を使用して、ルートモジュールの`asyncData`フックにこのモジュールを遅延登録できます。

```html
// store / index.jsの代わりにここにモジュールをインポートするimport fooStoreModule from '.. （ 'foo'、fooStoreModule）return store.dispatch（ 'foo / inc'）}、//重要：モジュールの重複登録は避けてください。クライアントがルートを複数回訪問したとき。 {fooCount（）{return this。$ store.state.foo.count}}}} </ script>これは、
```

モジュールはルートコンポーネントの依存関係になっているので、webpackによってルートコンポーネントの非同期チャンクに移動されます。

---

Phew、それはたくさんのコードでした！これは、ユニバーサルデータフェッチがサーバーレンダリングされたアプリケーションではおそらく最も複雑な問題であり、私たちは開発を容易にするための基礎を築いているからです。ボイラープレートが設定されると、個々のコンポーネントをオーサリングすることは、実際にはかなり楽しいものになります。
