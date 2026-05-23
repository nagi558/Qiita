---
title: Next.jsのmiddlewareとschema(Zod)の違いをRailsのbefore_action / validatesで理解する
tags:
  - Rails
  - TypeScript
  - Next.js
  - zod
private: false
updated_at: '2026-05-23T20:18:44+09:00'
id: 7fb69f72d426a3b82692
organization_url_name: null
slide: false
ignorePublish: false
---

# Next.jsのmiddlewareとschema(Zod)の違いをRailsのbefore_action / validatesで理解する

Next.jsを学習していると、

- middleware
- schema（Zod）

という言葉がよく出てきます。

ただ、Rails経験者だと

- 「結局何が違うの？」
- 「Railsでいうとどれ？」
- 「どの層で動いてるの？」

となりやすいです。

この記事では、

- Rails経験者が Next.js の概念を最速で理解できる
- middleware と schema(Zod) の違いが整理できる
- Rails のどの概念に近いか分かる

ことを目標に、Railsと比較しながら解説します。

---

# まず結論：middleware と schema(Zod) の違い

最初に比較表を見ると理解しやすいです。

| 項目 | middleware | schema / Zod |
|---|---|---|
| 役割 | アクセス制御 | データ検証 |
| 何を守る？ | ページ・API | フォーム・入力値 |
| 実行タイミング | ルーティング前 | データ送信時 |
| Railsで近いもの | before_action / Rack middleware | validates / ActiveModel::Validations |
| 主な用途 | 認証・リダイレクト | 入力チェック |
| よく使う技術 | middleware.ts | Zod |

ざっくり言うと、

- middleware → 「この人アクセスしてOK？」
- schema → 「このデータ正しい？」

をチェックしています。

---

# どの層で動く？

図で見るとこんなイメージです。

```txt
Client
  ↓
middleware
  ↓
Route Handler / Page
  ↓
schema / Zod
  ↓
DB
```

役割としては、

- middleware → 認証・アクセス制御
- schema → 入力データ検証

を担当しています。

---

# middlewareとは？

middleware は、

> Request後、ページやAPIへ到達する前に割り込んで処理するもの

です。

イメージはこんな感じです。

```txt
Request
 ↓
middlewareでチェック
 ↓
Page / API Route
```

---

# Railsでいう before_action に近い

Rails経験者だと、かなり近いのがこれです。

```rb
before_action :authenticate_user!
```

例えば、

- ログインしているか確認
- 管理者だけ許可
- 特定ページへリダイレクト

などを行います。

Next.js の middleware も役割としてかなり近いです。

---

# middleware は Edge Runtime で動く

ただし、Rails の before_action と完全に同じではありません。

Next.js の middleware は、

> ルーティング前に Edge Runtime 上で実行される

という特徴があります。

つまり、

- Controller到達前
- Page描画前
- API到達前

に実行されます。

さらに Edge Runtime は、

- Node.js ではなく V8 Isolate 上で動く
- fetch が高速
- fs など一部 Node API が使えない

という特徴があります。

Railsでいうと、

- Rack middleware
- before_action

の中間くらいのイメージが近いです。

---

# Next.js の middleware の例

例えば、
ログインしていない場合に `/login` へ飛ばす処理です。

```ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('token')

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}
```

やっていることは Rails の

```rb
before_action :authenticate_user!
```

にかなり近いです。

---

# schema(Zod)とは？

一方 schema は、

> データの構造や型をチェックするもの

です。

例えば、

- title は文字列？
- 空文字じゃない？
- 文字数長すぎない？
- 数値が入っている？

などを確認します。

---

# Railsでいう validates に近い

Railsでいうとこれです。

```rb
validates :title, presence: true
```

つまり、

> 「正しいデータだけ受け取りたい」

という役割です。

さらに厳密にいうと、
Rails の `ActiveModel::Validations` に近い考え方です。

---

# Next.jsでは Zod を使うことが多い

Next.js では `Zod` を使うケースがかなり多いです。

例えばこんな感じです。

```ts
import { z } from 'zod'

export const postSchema = z.object({
  title: z.string().min(1, 'タイトルは必須です'),
})
```

これは Rails の

```rb
validates :title, presence: true
```

に近いです。

---

# Zod の強みは「型 + バリデーション」をまとめられること

Railsでは、

- Model に validations を書く
- 型は DB や Ruby 側で管理

することが多いです。

一方 Next.js + TypeScript + Zod では、

> 型定義とバリデーションを同時に定義できます。

例えば、

```ts
type Post = z.infer<typeof postSchema>
```

のように、
schema から TypeScript の型推論もできます。

これが Zod のかなり強いポイントです。

---

# schema(Zod) はどこで使う？

実際の Next.js では、
schema はかなり色々な場所で使われます。

例えば、

- API Route の入力チェック
- Server Actions のバリデーション
- フォーム送信時の型安全性
- React Hook Form と組み合わせた入力検証

などです。

特に TypeScript と組み合わせることで、

> 「型安全 + バリデーション」

を同時に実現できるのが大きなメリットです。

---

# 実際は middleware と schema を両方使う

例えば投稿機能なら、

## middleware
- 未ログインなら `/login` へリダイレクト

## schema(Zod)
- title が空じゃないか確認
- body の文字数確認

## API Route / Server Actions
- DBへ保存

という形で組み合わせます。

つまり、

> 「誰がアクセスできるか」
> 「どんなデータを受け取るか」

を別々に管理しています。

---

# まとめ

Rails経験者向けにまとめると、

| Next.js | Rails |
|---|---|
| middleware | before_action / Rack middleware |
| schema / Zod | validates / ActiveModel::Validations |

と考えるとかなり理解しやすいです。

Next.js は最初用語が多く感じますが、
Rails経験者だと「役割」が似ているものもかなり多いです。

Rails から Next.js に移行中の方で、

- 「これって Rails でいうと何？」
- 「この概念の対応関係も知りたい」

などあれば、ぜひコメントで教えてください！

次回は、
「Next.js の Server Actions を Rails で例えると？」
も書く予定です。
