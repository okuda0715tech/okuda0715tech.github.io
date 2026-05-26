+++
title = "UiState にドメインモデルをどこまで持たせるべきか考えた"
date = "2026-05-26T13:00:00+09:00"
categories = ["Architecture"]
tags = ["UI", "Domain"]
draft = false
+++


# UiState にドメインモデルをどこまで持たせるべきか考えた

Jetpack Compose で画面実装をしていると、よく悩むのが、

* UiState にどこまでドメインモデルを持たせるべきか
* UI専用モデルをどこまで作るべきか

という問題です。

最近、自分は `Id` の扱いについて考える機会がありました。

## きっかけ

たとえば、以下のような UiState があるとします。

```kotlin
data class UiState(
    val id: Id = Id.Unassigned,
    val name: String = "",
) {
    sealed interface Id {
        data object Unassigned : Id
        data class Assigned(val value: Int) : Id
    }
}
```

一方で、ドメインモデル側にも、ほぼ同じ定義があります。

```kotlin
sealed interface PaymentId {
    data object Unassigned : PaymentId
    data class Assigned(val value: Int) : PaymentId
}
```

さらに、今後ほかの画面でも、

```kotlin
PaymentEditUiState.Id
PaymentDetailUiState.Id
PaymentListItemUiState.Id
```

のような、似た ID 型が増えていく可能性がありました。

そこで、

> 「これ、UiState ごとに定義する必要ある？」

という疑問が出てきました。

---

# 結論

今回のケースでは、

> ドメイン側の ID 型をそのまま UiState で使う

方が自然だと感じました。

つまり、

```kotlin
data class UiState(
    val id: PaymentId = PaymentId.Unassigned,
    val name: String = "",
)
```

のようにする方針です。

---

# なぜそう考えたのか

## ID は比較的「安定した概念」

今回の `Id` は、

* UI専用概念を持たない
* 画面仕様に依存しない
* ドメインでもUIでも意味が同じ
* 将来的に意味が変わりにくい

という特徴があります。

つまり、

> 「変化しにくい ValueObject」

として扱えると考えました。

---

# 本当に危険なのは「変化しやすいドメインモデル」

UiState がドメインモデルを持つこと自体が問題なのではなく、

> 「変化しやすいドメインモデルを UI が直接握ること」

が問題になりやすいです。

たとえば以下のような Entity をそのまま持つと危険です。

```kotlin
data class Payment(
    val id: PaymentId,
    val payer: User,
    val histories: List<History>,
    val auditLog: AuditLog,
    val permissions: PermissionSet,
)
```

こうなると、

* DB構造変更
* API変更
* Aggregate変更
* ビジネスルール変更

などが UI に波及しやすくなります。

---

# 一方、ID は変化しにくい

しかし ID は、

* 「未採番」
* 「採番済み」

くらいしか状態がなく、意味も安定しています。

そのため、

```kotlin
sealed interface PaymentId
```

を UI とドメインで共有しても、影響範囲が爆発しにくいと考えました。

---

# むしろ UI専用 ID を増やすデメリットもある

無理に UiState 専用 ID を作ると、

* 同じ意味の型が増える
* mapper が増える
* 微妙な仕様差が生まれる
* 認知負荷が上がる

という別の問題が発生します。

特に、

```kotlin
PaymentEditUiState.Id
PaymentDetailUiState.Id
PaymentListItemUiState.Id
```

のような型が増殖すると、

> 「全部ほぼ同じでは？」

という状態になりやすいです。

---

# 最近は「全部分離」が正義ではないと感じる

以前は、

> UI とドメインは完全分離すべき

という考え方を強く持っていました。

しかし最近は、

* 安定した ValueObject は共有
* 変化しやすい Entity は分離

というバランス設計の方が、実装コストや保守性との釣り合いが良いと感じています。

たとえば、

```kotlin
data class UiState(
    val paymentId: PaymentId,
    val accountId: AccountId,
    val title: String,
    val items: List<PaymentItemUiModel>,
)
```

のように、

* ID
* enum
* Money
* Date
* ValueObject

などは共有し、

表示専用部分だけ UiModel 化する設計です。

---

# まとめ

今回学んだことは、

> 「UiState にドメインモデルを持たせるかどうか」

ではなく、

> 「その型はどれくらい変化しやすいのか」

を考えることが大切だということです。

特に ID のような安定した ValueObject は、

* UI専用型を量産するより
* ドメイン型を共有した方が
* シンプルで読みやすくなる

ケースも多いと感じました。


