+++
title = "Aggregate Root とは"
date = "2026-02-12T10:00:00+09:00"
tags = ["DDD"]
draft = false
+++


- [Aggregate Root とは](#aggregate-root-とは)
  - [はじめに](#はじめに)
  - [Aggregate（集約）とは](#aggregate集約とは)
  - [Aggregate Root（集約ルート）とは](#aggregate-root集約ルートとは)
  - [具体例（イメージ）](#具体例イメージ)
    - [注文ドメインの例](#注文ドメインの例)
  - [なぜ Aggregate Root が必要なのか](#なぜ-aggregate-root-が必要なのか)
    - [① 整合性を守るため](#-整合性を守るため)
    - [② トランザクション境界を明確にするため](#-トランザクション境界を明確にするため)
    - [③ 依存関係をシンプルにするため](#-依存関係をシンプルにするため)
  - [設計時の注意点](#設計時の注意点)
    - [① Aggregate を大きくしすぎない](#-aggregate-を大きくしすぎない)
    - [② 他 Aggregate を直接参照しない](#-他-aggregate-を直接参照しない)
    - [③ Repository は Aggregate Root 単位](#-repository-は-aggregate-root-単位)
  - [よくある誤解](#よくある誤解)
  - [まとめ](#まとめ)


# Aggregate Root とは

## はじめに

DDD（Domain-Driven Design）の文脈で頻繁に登場する **Aggregate Root（集約ルート）** について解説していきます。

本記事では、

* Aggregate / Aggregate Root とは何か
* なぜこの概念が必要なのか
* 設計時の注意点

を整理します。

---

## Aggregate（集約）とは

Aggregate とは、

* 関連する複数の Entity / Value Object を
* **一つのまとまり（整合性の境界）** として扱う

DDD の設計単位です。

重要なのは、

> **Aggregate は「常に一貫した状態を保つべき単位」**

という点です。

---

## Aggregate Root（集約ルート）とは

Aggregate Root とは、

* Aggregate の中で
* **外部から直接参照・操作してよい唯一の Entity**

を指します。

Aggregate 内には複数の Entity が存在することがありますが、

* 外部のコードは **Aggregate Root 経由でしか操作できない**
* 内部の Entity は、Root に守られた存在

という関係になります。

---

## 具体例（イメージ）

### 注文ドメインの例

* Order（注文） ← Aggregate Root
* OrderItem（注文明細）

この場合、

* OrderItem を単体で保存・取得しない
* OrderItem の追加・削除は **必ず Order を通して行う**

というルールを設けます。

```kotlin
class Order(
    val id: OrderId,
    private val items: MutableList<OrderItem>
) {
    fun addItem(item: OrderItem) {
        // 整合性チェック
        items.add(item)
    }
}
```

OrderItem は Order の内部状態であり、

* 勝手に変更されること
* 不正な状態になること

を Aggregate Root が防ぎます。

---

## なぜ Aggregate Root が必要なのか

### ① 整合性を守るため

もし内部 Entity を直接操作できると、

* 制約チェックが漏れる
* 状態が壊れる

といった事故が起こりやすくなります。

Aggregate Root は、

* **状態変更の入口を一箇所に集約**

することで、これを防ぎます。

---

### ② トランザクション境界を明確にするため

DDD では、

* **1 トランザクション = 1 Aggregate**

が原則です。

本記事でいう「トランザクション」は、  
DB の BEGIN / COMMIT を指すものではなく、  
**ビジネス的に一貫性が保たれるべき操作単位** を意味します。

Aggregate Root を単位にすれば、

* どこまでを一括で保存するか
* どこからを別トランザクションにするか

が自然に決まります。

---

### ③ 依存関係をシンプルにするため

外部コードが

* Root だけを知っていればよい

状態になるため、

* 依存が減る
* 設計が読みやすくなる

というメリットがあります。

---

## 設計時の注意点

### ① Aggregate を大きくしすぎない

「関連しているから」という理由だけで

* 何でも一つの Aggregate に詰め込む

のは危険です。

* 更新頻度
* トランザクションの必要性

を基準に分けるのがポイントです。

---

### ② 他 Aggregate を直接参照しない

Aggregate 間の参照は、

* **ID 参照のみ**

に留めるのが原則です。

```kotlin
class Order(
    val customerId: CustomerId
)
```

Customer オブジェクトを直接持たないことで、

* 境界が崩れる
* トランザクションが肥大化する

のを防ぎます。

---

### ③ Repository は Aggregate Root 単位

Repository は、

* Aggregate Root の取得・保存のみを責務とする

のが基本です。

```kotlin
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
}
```

内部 Entity 用の Repository を作らない点が重要です。

---

## よくある誤解

* Entity = Aggregate Root ではない
* テーブル = Aggregate ではない
* 画面単位で Aggregate を決める

Aggregate は、

> **ビジネスルールと整合性の単位**

から導かれるものです。

---

## まとめ

* Aggregate は「整合性の境界」
* Aggregate Root は「外部との唯一の窓口」
* 状態変更・保存・取得は Root 経由

この概念を意識することで、

* ドメインモデルが壊れにくく
* 変更に強い設計

になります。

今後、UseCase や Repository を設計するときも、

> 「これはどの Aggregate Root を操作しているか？」

を常に意識していきたいところです。

