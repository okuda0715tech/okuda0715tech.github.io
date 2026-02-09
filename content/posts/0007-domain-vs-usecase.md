+++
title = "Domain と UseCase の違いを整理してみる"
date = "2026-02-09T11:00:00+09:00"
tags = ["Domain レイヤー", "Domain", "UseCase"]
draft = false
+++

- [Domain と UseCase の違いを整理してみる](#domain-と-usecase-の違いを整理してみる)
  - [一言で言うと何が違うのか](#一言で言うと何が違うのか)
  - [Domain とは何か](#domain-とは何か)
    - [Domain は「意味」と「ルール」の集合体](#domain-は意味とルールの集合体)
    - [Domain の例](#domain-の例)
  - [Domain がやらないこと](#domain-がやらないこと)
  - [UseCase とは何か](#usecase-とは何か)
    - [UseCase は「動詞の層」](#usecase-は動詞の層)
    - [UseCase の例](#usecase-の例)
  - [Domain と UseCase の関係](#domain-と-usecase-の関係)
  - [なぜ UseCase が分かりにくいのか](#なぜ-usecase-が分かりにくいのか)
  - [UseCase を作るべきタイミング](#usecase-を作るべきタイミング)
  - [Entity → Domain 変換はどこでやるべきか](#entity--domain-変換はどこでやるべきか)
  - [Domain と UseCase を分ける最大のポイント](#domain-と-usecase-を分ける最大のポイント)
  - [よくあるアンチパターン](#よくあるアンチパターン)
  - [おわりに](#おわりに)


# Domain と UseCase の違いを整理してみる

Clean Architecture や DDD を学んでいると、ほぼ確実に次の疑問にぶつかります。

* Domain と UseCase の違いがよく分からない
* どこまでが Domain で、どこからが UseCase なのか曖昧
* UseCase を作ろうとすると、何を書けばいいのか分からない

私自身、このあたりで何度も立ち止まりました。
この記事では、実装経験を通して整理できた **Domain と UseCase の違い** を、自分なりの言葉でまとめてみます。

---

## 一言で言うと何が違うのか

まず、かなり大胆に要約します。

* **Domain**
  →「この世界では、何が正しく、何が成り立つか」を表す

* **UseCase**
  →「その正しさを、どういう手順・文脈で使うか」を表す

この一文を起点にすると、境界が見えやすくなります。

---

## Domain とは何か

### Domain は「意味」と「ルール」の集合体

Domain が表すのは、次のようなものです。

* アプリが扱う概念（名詞）
* その概念に関する制約
* 不変条件
* 正しい / 正しくないの判断基準

重要なのは、Domain が **技術から切り離されている** という点です。

---

### Domain の例

```kotlin
@JvmInline
value class SourceId(val value: String)
```

```kotlin
data class Source(
    val id: SourceId,
    val name: String,
    val isActive: Boolean
) {
    init {
        require(name.isNotBlank())
    }
}
```

ここでやっているのは、

* 「Source とは何か」の定義
* 名前は空ではいけない、というルール

であって、

* DB
* API
* Flow
* Android

といったものは一切登場しません。

Domain は **世界観そのもの** を表します。

---

## Domain がやらないこと

これはかなり重要です。

Domain は、次のようなことをやりません。

* データを取得する
* 保存する
* 非同期処理を扱う
* 時系列を持つ
* Repository を呼ぶ

Domain は常に **現在形** で存在します。

---

## UseCase とは何か

### UseCase は「動詞の層」

UseCase は、Domain とは役割がはっきり違います。

UseCase が表すのは、

* ユーザーの行動
* アプリとしての操作単位
* 一連の手順や流れ

つまり、**何かをする** という文脈です。

---

### UseCase の例

```kotlin
class ObserveSourcesUseCase(
    private val repo: SourceRepository
) {
    operator fun invoke(): Flow<List<Source>> =
        repo.observeSources()
            .map { list ->
                list.filter { it.isActive }
            }
}
```

このコードには、

* Repository
* Flow
* データの流れ
* 条件による選別

といった **時間や手順** の概念が含まれています。

これが Domain との決定的な違いです。

---

## Domain と UseCase の関係

関係性を整理すると、次のようになります。

* Domain は単体で完結する
* UseCase は Domain を使って何かをする
* Domain は「正しさ」を持つ
* UseCase は「使われ方」を持つ

UseCase は Domain の上位互換ではありません。
**役割がまったく違う層** です。

---

## なぜ UseCase が分かりにくいのか

UseCase が分かりにくくなる原因は、だいたい次のどれかです。

* Repository の薄いラッパーになっている
* CRUD をそのまま移しただけ
* 「層を作ること」が目的になっている

こうなると、

> 「これ、本当に必要？」

という感覚になります。

その違和感は正しいです。

---

## UseCase を作るべきタイミング

私の中での判断基準は、次のようなケースです。

* 複数の Repository をまたぐ
* データを加工して意味のある形にする
* List / Map / Filter / Sort などを組み合わせる
* UI から切り離したい判断が含まれる

これらは **「どう使うか」** の話なので、UseCase の責務になります。

---

## Entity → Domain 変換はどこでやるべきか

よくある疑問に、

> Entity を Domain に変換するのは UseCase？ Repository？

というものがあります。

答えは一択ではありません。

* Entity と Domain がほぼ 1:1
  → Repository で変換するのが自然
* ユースケースごとに意味が変わる
  → UseCase で変換するのが自然

重要なのは、**ViewModel が Entity を知らないこと** です。

---

## Domain と UseCase を分ける最大のポイント

自分なりに一番しっくり来た整理は、これです。

> **Domain は「これは正しいか」を決める**
> **UseCase は「それをいつ・どう使うか」を決める**

この2つを混ぜ始めると、境界は一気に崩れます。

---

## よくあるアンチパターン

* Domain に Flow が出てくる
  → 時間の概念が混ざっている
* UseCase に Entity が出てくる
  → インフラが漏れている
* UseCase が ViewModel の代わりになっている
  → UI 依存が侵食している

このあたりは要注意です。

---

## おわりに

Domain と UseCase の違いは、
ルールを暗記しても分かりにくい部分です。

実装しながら、

* これは「意味」なのか
* これは「使い方」なのか

と問い続けることで、少しずつ輪郭が見えてきました。

