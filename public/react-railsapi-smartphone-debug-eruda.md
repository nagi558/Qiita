---
title: React + Rails API のスマホ実機デバッグは Eruda が最強だった｜iPhoneだけ起きるバグ調査に必須だった
tags:
  - Rails
  - API
  - フロントエンド
  - React
  - eruda
private: false
updated_at: '2026-05-13T21:11:44+09:00'
id: 979e2bb214d8d9b3d8ad
organization_url_name: null
slide: false
ignorePublish: false
---

# React + Rails API のスマホ実機デバッグは Eruda が最強だった｜iPhoneだけ起きるバグ調査に必須だった

React + Rails API 開発をしていると、

- iPhoneだけレイアウトが崩れる
- スマホだけAPI通信が失敗する
- PCでは正常なのに iPhone Safari だけ 401
- console.log が確認できない

みたいなことが結構ありました。

特に厄介だったのが、

> 「PCでは再現しないので調査が進まない」

状態になることでした。

React + Rails API 構成だと、

- iOS Safari のキャッシュ挙動
- fetch / axios の挙動差
- cookie / token の扱い
- CORS の違い

などで、スマホ実機だけ問題が起きることがあります。

実際、自分の環境では iOS Safari が古いレスポンスを保持していて、Rails API 側で更新した token が反映されないことがありました。

ただ、スマホブラウザには PC の Chrome DevTools のような検証ツールがありません。

Safari Web Inspector を使う方法もありますが、

- Mac が必要
- USB接続が必要
- Safari設定変更が必要

など少し準備が必要でした。

iOS 17 以降では接続が不安定になるケースもあるみたいで、もっと軽くスマホ実機デバッグできる方法を探していました。

そこで便利だったのが **Eruda** です。

Eruda を使うと、スマホ上で DevTools のような機能を表示できます。

特に React + Rails API 開発で、

- console.log の確認
- API通信確認
- LocalStorage の token 確認
- DOM確認

などをスマホ実機だけで確認できるのがかなり便利でした。

---

# Erudaとは？

Eruda は、モバイルブラウザ向けの軽量 DevTools です。

簡単に言うと、

> スマホで使える Chrome DevTools の代替ツール

みたいな感じです。

script タグを追加するだけで使えるので、

- Remote Debugging
- Safari Web Inspector

などより導入がかなり簡単でした。

表示できる主な機能としては、

- Elements
- Console
- Network
- Resources

などがあります。

---

# Erudaの注意点

かなり便利ですが、万能ではありません。

Eruda はブラウザ上で動く JavaScript ツールなので、

- ネイティブレベルの通信ログ
- Service Worker の詳細挙動
- Safari内部の低レベルエラー

などは完全には追えません。

例えば Rails API の cookie-based 認証を使っている場合、

```txt
SameSite=Lax
```

や

```txt
SameSite=None
```

のブラウザ内部挙動までは完全には確認できませんでした。

そのため、

- 軽量にスマホ実機デバッグしたい
- console.log や API通信を確認したい

時にかなり向いているツールだと思いました。

---

# Before / After

## Before

実際に困ったのが、

> iPhone Safari だけ Rails API が 401 を返す

問題でした。

PCブラウザでは正常に 200 が返るのに、

iPhone だけ認証エラーになっていました。

最初は、

- CORS 設定が原因？
- Rails の `before_action`？
- axios の `baseURL` ミス？
- token期限切れ？

などを疑っていました。

ただ、

- console.log が見れない
- token が送れているか分からない
- Network確認ができない

ので、

> 「推測だけで修正する」

状態になっていました。

かなり調査が止まりました。

---

## After

Eruda を導入すると、スマホ上でそのまま確認できました。

Network タブを確認すると、

```http
401 Unauthorized
```

になっていました。

さらに Headers を見ると、

```http
Authorization: Bearer xxx
```

が送られていませんでした。

結果的には、

```ts
axios.interceptors.request.use()
```

の条件分岐ミスで、iPhone Safari 側だけ token が付与されていないのが原因でした。

Eruda がなければ、

- 再現はできる
- でも原因調査ができない

状態だったので、かなり助かりました。

---

# Erudaの導入方法

HTML に script タグを追加するだけで使えます。

```html
<script src="//cdn.jsdelivr.net/npm/eruda"></script>

<script>
  eruda.init();
</script>
```

---

# `//cdn.jsdelivr.net` なのはなぜ？

これは「プロトコル相対URL」と呼ばれる書き方みたいです。

例えば、

- HTTPSサイト → HTTPSで読み込み
- HTTPサイト → HTTPで読み込み

になります。

そのため、

```html
//cdn.jsdelivr.net/npm/eruda
```

と書くだけで両方対応できます。

ただ、Rails の CSP（Content-Security-Policy）で、

```txt
script-src 'self'
```

になっている場合、CDN の script がブロックされることがありました。

---

# React + Vite で導入する場合

React + Vite の場合は `index.html` に追加すると動きました。

```html
<!-- index.html -->

<body>
  <div id="root"></div>

  <script src="//cdn.jsdelivr.net/npm/eruda"></script>

  <script>
    eruda.init();
  </script>
</body>
```

---

# なぜ index.html に書くの？

React は SPA（Single Page Application）なので、

```html
<div id="root"></div>
```

配下を JavaScript で描画しています。

そのため、

```html
index.html
```

に script を追加すると、アプリ全体で Eruda を有効化できます。

また、

```html
<body>
```

の最後に script を置くことで、

- DOM生成後に script 実行
- hydration 前後のエラーを避けやすい

というメリットもあるみたいでした。

Vite は JavaScript を bundle して読み込むため、script の実行タイミングも少し重要そうでした。

---

# キャッシュで反映されない時

iPhone Safari はキャッシュがかなり強く残ることがありました。

script を追加しても表示されない場合は、

- スーパーリロード
- キャッシュ削除
- シークレットタブ

などを試すと表示されることがありました。

---

# 主な機能

# Elements

Elements では DOM を確認できます。

例えば React + Tailwind 開発で、

```tsx
<div className="md:hidden">
```

が効いていないと思っていたら、

実際には Safari の viewport 幅で条件分岐されていた、みたいな確認ができました。

また、

- className違い
- display:none
- z-index崩れ

なども確認できました。

スマホだけモーダルが崩れる時にかなり便利でした。

---

# Console

Console では、

- console.log
- JavaScriptエラー
- warning
- コマンド実行

などを確認できます。

例えば、

```ts
console.log(user)
```

```ts
console.log(response.data)
```

などをスマホ実機で直接確認できます。

実際に、

> iPhone Safari だけ undefined になる

問題があり、

```ts
props.user.name
```

を参照する前に `user` が undefined になっていることを確認できました。

---

# Network

個人的にはこれがかなり便利でした。

Network タブでは、

- APIリクエスト
- レスポンス
- Status Code
- Headers
- 通信時間

などを確認できます。

React + Rails API 構成だと、

- Rails API の 401
- token送信漏れ
- CORSエラー
- axios の headers 問題

などを確認できました。

また、

```ts
credentials: 'include'
```

が不足していて Cookie が送られていないことも確認できました。

Rails API 側で、

```rb
protect_from_forgery
```

を使っていた時の CSRF 問題調査にも役立ちました。

さらに、iPhone Safari のキャッシュで古い API レスポンスが返っていたことも確認できました。

---

# Resources

Resources では、

- LocalStorage
- SessionStorage
- Cookie

などを確認できます。

JWT認証を使っている場合、

```ts
localStorage.getItem('token')
```

で保存している token が存在するか確認できます。

実際に、

- 古い token が残っていた
- token が null だった

ことが分かり、認証エラー原因の調査に役立ちました。

---

# 開発環境だけで有効化する方法

Eruda は便利ですが、本番環境では OFF の方が安全そうでした。

理由としては、

- LocalStorage が見れる
- APIレスポンスが見れる
- Cookie確認できる
- Console操作できる

ためです。

例えば Rails API の、

```txt
/users/me
```

レスポンスが確認できる状態だと、ユーザー情報が見えてしまう可能性があります。

JWT token を LocalStorage に保存している場合、

```ts
localStorage.getItem('token')
```

などで確認できてしまいます。

そのため、開発用途向けだと思いました。

---

# Viteで開発環境だけ有効化する

Vite の場合は、以下のように条件分岐できます。

```ts
if (import.meta.env.DEV) {
  const script = document.createElement('script')

  script.src = '//cdn.jsdelivr.net/npm/eruda'

  script.onload = () => {
    window.eruda.init()
  }

  document.body.appendChild(script)
}
```

---

# このコードで何をしているの？

まず、

```ts
import.meta.env.DEV
```

で、

> 開発環境かどうか

を判定しています。

これにより、本番環境では Eruda を読み込まなくできます。

また、

```ts
script.onload
```

を使うことで、

> script の読み込み完了後に `eruda.init()`

を実行しています。

script 読み込み前に `eruda.init()` を呼ぶと、

```txt
eruda is not defined
```

になるため、この順番が必要みたいでした。

---

# まとめ

React + Rails API の開発では、

- iPhoneだけエラー
- スマホだけレイアウト崩れ
- token が送れていない
- CORS が怪しい
- PCでは再現しない

など、「スマホ実機だけ問題が起きる」ことが結構ありました。

ただ、PC の Chrome DevTools では再現できず、調査が止まりやすいです。

Eruda を使うと、

- console.log 確認
- API通信確認
- token確認
- DOM確認

などをスマホ実機だけで確認できます。

Safari Web Inspector より導入も簡単だったので、

- とりあえず軽く確認したい
- React のスマホ実機デバッグをしたい
- Rails API の通信エラーを確認したい

時にかなり便利でした。
