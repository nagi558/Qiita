---
title: RuboCopで学んだRails設計の基本
tags:
  - Rails
  - RSpec
  - RuboCop
private: false
updated_at: '2026-05-10T20:00:31+09:00'
id: ee220fb8789413314165
organization_url_name: null
slide: false
ignorePublish: false
---


# RuboCopで学んだRails設計の基本  
## `RSpec/MessageSpies` `Rails/I18nLocaleTexts` `Metrics/AbcSize` から理解する「壊れにくいRailsコード」の考え方

Rails開発をしていると、RuboCopにかなり細かく怒られます。

最初は、

> 「動いているんだから良くない？」

と思っていました。

しかし調べていくと、RuboCopは単なるLintツールではなく、

- brittle（壊れやすい）なテストを防ぐ
- Fat Controller を防ぐ
- 責務分離を促す
- Railsらしい設計へ導く

ための「Rails設計ガイド」に近い存在だと分かりました。

特に以下3つのCopは、単なる書き方ルールではなく、Railsの思想そのものに深く関係しています。

- `RSpec/MessageSpies`
- `Rails/I18nLocaleTexts`
- `Metrics/AbcSize`

この記事では、

- なぜそのCopが存在するのか
- 何が問題なのか
- なぜRailsコミュニティで推奨されるのか
- どう直すべきなのか
- 直すと何が嬉しいのか
- どう再発防止するのか

まで整理します。

---

# RuboCop設定例（まずはここ）

```yml
# .rubocop.yml

RSpec/MessageSpies:
  Enabled: true

Rails/I18nLocaleTexts:
  Enabled: true

Metrics/AbcSize:
  Max: 20
```

Cop名を知っておくと、

- issue調査しやすい
- 検索しやすい
- チーム会話しやすい

というメリットがあります。

QiitaやGoogle検索でも、

- `RuboCop RSpec/MessageSpies とは`
- `Rails I18nLocaleTexts エラー`
- `Metrics/AbcSize 対処法`

などで流入しやすくなります。

---

# 1. `RSpec/MessageSpies` が推奨される理由  
## brittle test（壊れやすいテスト）を減らすため

RSpecを書いた時、こんな感じでした。

```rb
expect(UserMailer).to receive(:welcome_email)

post :create
```

しかしRuboCop-RSpecでは、`RSpec/MessageSpies` に引っかかることがあります。

推奨されるのは以下のspyスタイルです。

```rb
allow(UserMailer).to receive(:welcome_email)

post :create

expect(UserMailer).to have_received(:welcome_email)
```

---

# なぜ `have_received` が推奨されるのか？

## 背景：RSpec 3.0でspyが正式導入された

RSpec 3.0では、

> 「test double の役割を明確化する」

という目的でspyが正式導入されました。

従来の `receive` は、

- stub
- mock
- spy

の役割が曖昧になりやすく、テスト意図が読み取りづらい問題がありました。

そのためRSpecコミュニティでは、

- 「許可（stub）」
- 「監視（spy）」
- 「事後検証（have_received）」

を分離するスタイルが推奨されるようになりました。

---

# `receive` が brittle test を生みやすい理由

`expect(...).to receive` は事前期待です。

つまり、

> 「このメソッドは必ず呼ばれる」

を先に宣言します。

これは呼び出し回数・順序への依存が強く、

- callback追加
- 非同期Job化
- リファクタリング
- メール送信タイミング変更

などで簡単に壊れることがあります。

これがいわゆる、

> brittle test（壊れやすいテスト）

です。

---

# spy は「実際に起きたこと」を検証する

spyは、

```rb
allow(...).to receive(...)
```

で監視し、

```rb
expect(...).to have_received(...)
```

で事後確認します。

つまり、

1. 実行
2. 実際に呼ばれたか確認

という自然な流れになります。

これは実装変更への耐性が高く、

- リファクタリングしやすい
- テスト意図が読みやすい
- 失敗原因を追いやすい

というメリットがあります。

---

# 副作用を持つ処理では特に有効

例えば、

- Mailer
- ActiveJob
- Callback
- Observer
- Webhook

など。

これらは副作用を持つことが多く、事前期待で振る舞いを固定しすぎると、本来の処理を壊すケースがあります。

例えば：

```rb
expect(UserMailer).to receive(:welcome_email).and_call_original
```

このようなコードは、

- 呼び出し順序
- callback
- deliver_later化

などの変更で壊れやすくなります。

spyは「監視」に近いため、副作用との相性が良いです。

---

# Before / After

## Before

```rb
expect(UserMailer).to receive(:welcome_email)

post :create
```

## After

```rb
allow(UserMailer).to receive(:welcome_email)

post :create

expect(UserMailer).to have_received(:welcome_email)
```

---

# 直すと何が嬉しいのか？

- テストが壊れにくくなる
- リファクタリングしやすい
- テスト意図が読みやすくなる
- callback変更に強くなる

つまり、

> 「実装変更に強いテスト」

になります。

---

# 再発防止策

今は、

- 「副作用がある処理はspyを優先」
- 「事前期待より事後検証」

を意識するようになりました。

---

# 2. `Rails/I18nLocaleTexts` が怒る理由  
## Railsが「文言直書き」を嫌う理由

最初はメール文面をそのまま書いていました。

```rb
mail(
  subject: "会員登録ありがとうございます"
)
```

しかしRuboCop Railsでは、`Rails/I18nLocaleTexts` に引っかかることがあります。

---

# 改善後

## ja.yml

```yml
ja:
  mailer:
    user:
      welcome_subject: "会員登録ありがとうございます"
```

## mailer.rb

```rb
mail(
  subject: I18n.t("mailer.user.welcome_subject")
)
```

---

# なぜRailsは文言直書きを嫌うのか？

## Railsは「国際化前提」で設計されている

Railsガイドでも、

> 「Railsアプリケーションは国際化を前提に設計されている」

と説明されています。

特にActionMailerは、

- 日本語
- 英語
- 中国語

など、多言語メール送信を前提に設計されています。

つまりRailsでは、

> 「文言はコードではなくlocaleへ置く」

のが基本思想です。

---

# ActionMailerは本文もi18nできる

例えばsubjectだけでなく、bodyもlocale化できます。

```erb
# app/views/user_mailer/welcome.ja.text.erb

こんにちは、<%= @user.name %>さん
```

Railsは、

- subject
- body
- validation message
- flash message

などをi18nへ寄せる思想がかなり強いです。

---

# なぜ直書きが問題なのか？

## ① 多言語対応が難しくなる

日本語を直接書くと、英語対応時にコード変更が大量発生します。

---

## ② 文言変更でデプロイが必要になる

直書きすると、

- PR
- レビュー
- デプロイ

が必要になります。

localeへ分離すると、文言管理だけを独立できます。

---

## ③ テストが壊れやすくなる

例えば：

```rb
expect(mail.subject).to eq("会員登録ありがとうございます")
```

これは句読点変更だけでも失敗します。

一方、

```rb
expect(mail.subject).to eq(
  I18n.t("mailer.user.welcome_subject")
)
```

なら、実装とテストを統一できます。

---

# Rails思想とのつながり

Railsはかなり強く、

- 文言
- 設定
- ロジック
- DB処理

を分離したがるフレームワークです。

つまり、

- 文言 → locale
- ロジック → service
- HTTP制御 → controller

へ責務分離することで、保守性を上げています。

---

# 直すと何が嬉しいのか？

- 多言語対応しやすい
- 文言変更が楽になる
- テストが壊れにくい
- 責務分離できる

つまり、

> 「変更に強いRailsアプリ」

になります。

---

# 再発防止策

今は、

- 日本語を書いた瞬間に「i18n対象では？」
- 文字列を見たらlocaleを疑う

ようになりました。

---

# 3. `Metrics/AbcSize` が示す Fat Controller の危険性  
## Railsで「変更に弱いメソッド」を検出するCop

Rails初学者がかなりやりがちなのがこれです。

```rb
def create
  @post = Post.new(post_params)

  if @post.save
    if current_user.admin?
      AdminMailer.notify.deliver_now
    else
      UserMailer.notify.deliver_now
    end

    render json: @post
  else
    render json: { errors: @post.errors }
  end
end
```

これ、動きます。

でもRuboCopでは `Metrics/AbcSize` に引っかかることがあります。

---

# AbcSizeとは？

AbcSizeは、

- Assignment（代入）
- Branch（分岐）
- Condition（条件）

を数値化した複雑度指標です。

元になっているのは、Smalltalk界隈で生まれた ABC metric です。

一般的には以下で計算されます。

```text
√(Assignment² + Branch² + Condition²)
```

---

# なぜ15〜20が一般的なのか？

RuboCopのデフォルト値は `15` です。

ただしRailsでは、

- strong parameters
- render
- callback
- 認可処理

などで、どうしてもControllerが多少大きくなりがちです。

そのため現場では、

- 15〜20

程度へ緩和しているケースも多いみたいです。

ただし重要なのは、

> 「20以下にすること」

ではなく、

> 「変更に弱いメソッドを検出すること」

です。

---

# Fat Controller が危険な理由

Fat Controllerになると、

- if文が増える
- 責務が増える
- テストしづらい
- 修正影響範囲が広い

という問題が起きます。

つまり、

> 「何をするか」と「どうやるか」が混在する

状態になります。

---

# Before / After

## Before

```rb
def create
  @post = Post.new(post_params)

  if @post.save
    if current_user.admin?
      AdminMailer.notify.deliver_now
    else
      UserMailer.notify.deliver_now
    end

    render json: @post
  else
    render json: { errors: @post.errors }
  end
end
```

## After

```rb
def create
  @post = Post.new(post_params)

  return render_error unless @post.save

  send_notification

  render_success
end

private

def send_notification
  return AdminMailer.notify.deliver_now if current_user.admin?

  UserMailer.notify.deliver_now
end
```

---

# AbcSizeを下げる具体的テクニック

## ① Guard Clause（early return）

```rb
return render_error unless @post.save
```

ネストを減らせます。

---

## ② Extract Method（private化）

Controllerでは、

- 「何をするか」

だけ読めれば十分です。

---

## ③ Extract Class（Service Object化）

責務が増える場合は、

```text
app/services
```

へ切り出すことも多いです。

例えば：

- Service Object
- Form Object
- Command Object
- Presenter
- Decorator
- Interactor
- Trailblazer
- dry-rb

などがあります。

---

# 直すと何が嬉しいのか？

- Controllerが読みやすくなる
- テストしやすくなる
- 修正影響範囲が減る
- リファクタリングしやすい

つまり、

> 「変更に強いRails設計」

になります。

---

# RuboCopは「Rails設計」を学ばせるツールだった

最初は、

> 「Lintが細かすぎる」

と思っていました。

でも実際は、

- brittle test を防ぐ
- Fat Controller を防ぐ
- 責務分離を促す
- Rails思想へ近づける

ためのルールだったと感じています。

今回紹介した3つのCopは、全部バラバラに見えて、

> 「変更に強いRailsコードを書く」

という共通目的を持っていました。

---

# まとめ

| Cop | 防ぎたい問題 | 得られるメリット |
|---|---|---|
| `RSpec/MessageSpies` | brittle test | 壊れにくいテスト |
| `Rails/I18nLocaleTexts` | 文言とロジックの密結合 | 多言語対応しやすい |
| `Metrics/AbcSize` | Fat Controller | 保守しやすい設計 |

RuboCopは単なる静的解析ツールではなく、

> 「将来壊れにくいRailsコードを書くためのガイド」

にかなり近い存在でした。

もし他にも、

- RuboCopでよく怒られるCop
- Rails設計で悩んだこと
- Fat Model / Fat Controller問題

などあれば、ぜひ教えてください。
