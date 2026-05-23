---
title: Rails × ReactでuseEffect無限ループが起きやすい原因まとめ
tags:
  - Rails
  - API
  - フロントエンド
  - React
  - useEffect
private: false
updated_at: '2026-05-12T23:16:55+09:00'
id: 0f75aee74d04e254ecf4
organization_url_name: null
slide: false
ignorePublish: false
---

# Rails × ReactでuseEffect無限ループが起きやすい原因まとめ

Rails API + React で開発していると、

- 無限レンダリング
- API の連続リクエスト
- Loading が終わらない
- 無限リダイレクト

などの「無限ループ系のバグ」にかなり遭遇しました。

特に React の `useEffect` と Rails API の組み合わせでは、

> 「値が変わる → API 再取得 → また値が変わる」

という循環が発生しやすいみたいです。

最初は React 側だけの問題だと思っていたのですが、
調べてみると Rails API 側のレスポンス設計もかなり関係していました。

今回、実際にハマった内容をきっかけに、

- なぜ無限ループが起きるのか
- React は何を見て再実行しているのか
- Rails API の何が不安定さを生むのか

を整理してみます。

---

# まず前提：React は「値が変わった」と判断すると再実行する

`useEffect` は、

> 「依存配列の値が変わったら再実行される」

仕組みです。

例えば：

```tsx
useEffect(() => {
  fetchPosts()
}, [posts])
```

この場合、

```txt
posts が変わる
↓
useEffect 再実行
↓
fetchPosts 実行
↓
posts 更新
↓
また posts が変わる
```

という循環になります。

---

# React はどうやって「変化」を判定しているのか？

調べてみると、React は依存配列を `Object.is` ベースで比較しているみたいです。

つまり、

- primitive（文字列・数値）
- object
- array
- function

で挙動がかなり違います。

---

# primitive は値比較

```tsx
Object.is(1, 1) // true
Object.is("a", "a") // true
```

同じ値なら「変化なし」と判定されます。

---

# object / array / function は参照比較

```tsx
Object.is(
  { sort: "desc" },
  { sort: "desc" }
)
```

これは `false` になります。

つまり React は、

> 「中身が同じか」

ではなく、

> 「同じ参照か」

を見ているみたいです。

これが無限ループの原因になることがかなり多いです。

---

# 1. useEffect の依存配列ミス

React 側で最も多い原因です。

## ❌ NG例

```tsx
useEffect(() => {
  fetchPosts()
}, [posts])
```

これだと、

1. fetchPosts 実行
2. posts 更新
3. useEffect 再実行
4. 再fetch

のループになります。

---

# ✔ 対策

初回取得だけなら：

```tsx
useEffect(() => {
  fetchPosts()
}, [])
```

にします。

依存配列に state を入れる場合は、

> 「その state の変更時に本当に再実行したいか」

をかなり意識した方が良さそうでした。

---

# 2. Rails API が毎回「別データ」を返している

最初は React 側だけ疑っていたのですが、
Rails API 側が原因になっているケースもかなりありました。

---

# ❌ 例：order が不安定

```rb
Post.all
```

これだけだと、
DBによっては順番が保証されないことがあるみたいです。

例えば：

```json
[
  { "id": 1 },
  { "id": 2 }
]
```

だったものが、

次のリクエストでは：

```json
[
  { "id": 2 },
  { "id": 1 }
]
```

になる可能性があります。

---

# なぜ React 側で問題になるのか？

React 側では、

> 「前回取得したデータと違う」

と判断されるためです。

つまり、

```txt
データ取得
↓
順番が変わる
↓
state更新
↓
再render
↓
再fetch
```

という流れになります。

---

# updated_at もかなり危険

例えば API が毎回：

```json
{
  "updated_at": "2026-05-12T12:00:01"
}
```

↓

```json
{
  "updated_at": "2026-05-12T12:00:02"
}
```

のように変わると、

React 側では毎回「別データ」と判定されます。

---

# なぜ「毎回別データ」が危険なのか？

React の state 比較では、

```txt
前回と同じデータか
```

がかなり重要だからです。

つまり Rails API 側で、

- 毎回違う順番
- 毎回違う timestamp
- 毎回違う object 構造

を返すと、

React 側の再renderトリガーになります。

---

# ✔ Rails API 側の対策

## order を固定する

```rb
Post.order(created_at: :desc)
```

---

# 不要な値を返さない

例えば：

- updated_at
- generated_at
- ランダム値

など。

serializer / Jbuilder では、

> 「本当に必要なデータだけ返す」

ことが重要みたいです。

---

# API の「安定性」がかなり大事だった

今回調べてみて、

Rails API は、

> 「毎回同じ入力なら、毎回同じ出力を返す」

のがかなり重要だと分かりました。

この安定性が崩れると、
React 側では「毎回変化した」と判断されやすくなります。

---

# 3. state 更新と副作用が相互依存している

認証処理でかなりハマりました。

例えば：

```txt
token 更新
↓
user 取得
↓
token 検証
↓
また token 更新
```

みたいな循環です。

---

# なぜ危険？

`useEffect` が複数存在すると、

```txt
A の更新が B を呼ぶ
↓
B の更新が A を呼ぶ
```

という依存循環が起きやすくなるみたいです。

---

# ✔ 対策

調べてみると、

- AuthContext
- custom hook

などに認証処理を集約する構成がよく紹介されていました。

また、

> 「どの state がどの副作用を引き起こすか」

を整理すると原因を見つけやすかったです。

---

# 4. Rails の before_action で無限リダイレクト

Rails 側ではこれもかなり起きやすかったです。

## ❌ NG例

```rb
before_action :require_login
```

ログインページにも適用されると：

```txt
ログインページ
↓
ログイン必須
↓
ログインページへ redirect
↓
また before_action
```

というループになります。

---

# ✔ 対策

```rb
before_action :require_login, except: [:new, :create]
```

ログイン不要ページを除外します。

---

# React 側にも影響する

Rails 側でリダイレクトループが起きると、

React 側では：

- API エラー
- loading 固定
- 再fetch

などが起きることもありました。

最初は React の問題だと思っていたのですが、
Rails 側の認証設計が原因のことも多いみたいです。

---

# 5. useEffect 内で毎回「新しい object」を生成している

これもかなりハマりました。

## ❌ NG例

```tsx
useEffect(() => {
  setFilters({ sort: "desc" })
}, [filters])
```

最初は、

```txt
中身同じだから大丈夫では？
```

と思っていました。

---

# なぜループするのか？

React は object を、

> 「参照が同じか」

で判定しているみたいです。

つまり：

```tsx
{ sort: "desc" } !== { sort: "desc" }
```

として扱われます。

---

# ✔ 対策

```tsx
const defaultFilters = useMemo(() => ({
  sort: "desc"
}), [])

useEffect(() => {
  setFilters(defaultFilters)
}, [defaultFilters])
```

`useMemo` / `useCallback` で参照を安定化すると、
無限ループを防ぎやすいみたいです。

---

# まとめ

| 問題 | 原因 |
|---|---|
| useEffect ループ | 依存配列ミス |
| API 連打 | Rails が毎回別データを返す |
| 認証ループ | state 依存循環 |
| 無限redirect | before_action 適用範囲ミス |
| 再render地獄 | object参照変化 |

---

# 最後に

今回調べてみて、

ReactでもRailsでも、

> 「変更が別の変更を呼ぶ構造」

になると、無限ループが発生しやすいことが分かりました。

特に：

- useEffect
- object参照
- APIレスポンス
- before_action
- state依存

あたりは依存関係が複雑になりやすいみたいです。

まだ学習中ですが、
同じようにハマった方の参考になれば嬉しいです。
