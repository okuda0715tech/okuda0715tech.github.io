+++
title = "リポジトリがドメインに依存しすぎた話"
date = "2026-04-02T13:00:00+09:00"
categories = ["Architecture"]
tags = ["Clean Architecture"]
draft = false
+++

- [題材](#題材)
- [リポジトリがドメインに依存するのはありか？](#リポジトリがドメインに依存するのはありか)
- [ドメインを知りすぎた実装](#ドメインを知りすぎた実装)
- [ドメインを知りすぎない実装](#ドメインを知りすぎない実装)

---

今回は、リポジトリがドメインに依存しすぎた話を書きたいと思います。


## 題材

題材は、口座振替管理アプリで、ローカルの DB にデータを保存する場合の話です。

支払情報編集画面という画面があり、この画面では、データの新規登録と既存データの更新が可能になっています。

この「新規登録」処理と「更新」処理を行う際に、リポジトリはどこまでドメインを知っていていいのか？という話になります。


## リポジトリがドメインに依存するのはありか？

まず、クリーンアーキテクチャ的には、リポジトリがドメインに依存することは間違ってはいません。

ただし、どこまで知っていてよいのかは検討の余地があるというのが今回の学びです。


## ドメインを知りすぎた実装

まずは、ドメインを知りすぎた良くない実装を紹介します。

以下は、支払情報を表す `Payment` オブジェクトです。

```kotlin
sealed interface Payment {
    val name: PaymentName
    val payerId: PayerId

    data class Persisted(
        val id: PaymentId,
        override val name: PaymentName,
        override val payerId: PayerId = PayerId.NONE,
    ) : Payment

    data class InMemory(
        override val name: PaymentName,
        override val payerId: PayerId = PayerId.NONE,
    ) : Payment
}
```

編集の場合は、すでにデータが永続化されているため、 `Persisted` オブジェクトで表現します。永続化済みのデータは ID を持っています。

新規登録の場合は、 ID を持っていないため `InMemory` オブジェクトで表現します。

---

リポジトリ側には、この `Payment` オブジェクトが渡され、このオブジェクトの実態が `Persisted` なのか、 `InMemory` なのかによって、新規作成か、更新かを分けています。

```kotlin
suspend fun save(payment: Payment) {
    when (payment) {
        is Payment.InMemory -> create(...)
        is Payment.Persisted -> update(...)
    }
}
```

---

この実装の問題点は、もし、今後、 `Payment` の定義が以下のように変わった場合、リポジトリ側に影響が及ぶという点です。

```kotlin
Payment.Draft
Payment.Archived
```

クリーンアーキテクチャ的には、ドメインが変わった場合は、 UI レイヤーやデータレイヤーに影響が出るのは仕方のないことなのですが、できれば影響は避けたいところです。


## ドメインを知りすぎない実装

では、ドメインを知りすぎない実装とは、どのようになるでしょうか？

以下のように ID が null かどうかだけに依存するのが最小限の依存だと、私は考えます。

```kotlin
suspend fun save(payment: Payment) {
    if (payment.id == null) {
        create(...)
    } else {
        update(...)
    }
}
```

こうすることで、先ほどの問題点は解消されます。

また、リポジトリの責務は、データの保存方法を管理することです。

データの保存方法を判断するのに、 `Payment` オブジェクトが `Persisted` なのか `InMemory` なのかは知る必要がありません。

知りたいのは、 ID が null なのかどうかだけです。

こうすることで、データの保存の判断に必要な最低限の情報にだけ依存する実装になりました。

よって、保存方法の判断に関係のない変更に影響を受けることがなくなりました。

また、 ID の有無は、データの保存方法を判断するための本質的なデータです。そのため、データ保存上、なくなったり変わったりしにくいデータです。

一方で、 `Payment` オブジェクトの `Persisted` や `InMemory` はドメインの捉え方の問題なので、 ID に比べると変わりやすいモデリングです。

今回学んだことは、将来変わりにくい本質的なデータに依存することが重要だという点です。
