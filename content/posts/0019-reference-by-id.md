+++
title = "DDDで「集約はIDで参照する」とはどういう意味か"
date = "2026-03-12T19:00:00+09:00"
categories = ["Architecture"]
tags = ["DDD"]
draft = false
+++

- [オブジェクト参照の場合](#オブジェクト参照の場合)
  - [Order](#order)
  - [User](#user)
  - [アプリケーションサービス](#アプリケーションサービス)
- [ID参照の場合](#id参照の場合)
  - [Order](#order-1)
  - [User](#user-1)
  - [アプリケーションサービス](#アプリケーションサービス-1)
- [違いを整理](#違いを整理)
  - [オブジェクト参照](#オブジェクト参照)
  - [ID参照](#id参照)
- [もう一つ重要な違い：イベント駆動に拡張できる](#もう一つ重要な違いイベント駆動に拡張できる)
  - [Application Service](#application-service)
- [イベントの定義](#イベントの定義)
- [EventBusの簡易実装](#eventbusの簡易実装)
- [イベントハンドラ](#イベントハンドラ)
- [ハンドラ登録](#ハンドラ登録)
- [イベントの流れ](#イベントの流れ)
- [まとめ](#まとめ)

---

ドメイン駆動設計（DDD）では、次のようなルールをよく目にします。

> 集約は他の集約を **IDで参照する**

しかし、初めてこのルールを見たときに疑問が浮かびます。

* ID参照にしても結局 `Repository.find(id)` で取得できるのでは？
* それなら普通のオブジェクト参照とあまり変わらないのでは？

この記事では、このルールの意味を **実際のコードで比較しながら**説明します。

---

# オブジェクト参照の場合

まず、他の集約を **オブジェクトとして直接持つ設計**を見てみます。

## Order

```kotlin
class Order(
    val id: OrderId,
    val user: User
) {

    fun place() {
        // 注文処理
        user.upgradeToPremium()
    }
}
```

tips : place という単語には、 注文を出す / 発注する という意味があります。

## User

```kotlin
class User(
    val id: UserId,
    var plan: Plan
) {

    fun upgradeToPremium() {
        plan = Plan.PREMIUM
    }
}
```

## アプリケーションサービス

```kotlin
fun placeOrder(orderId: OrderId) {

    val order = orderRepository.find(orderId)

    order.place()

    orderRepository.save(order)
}
```

ここで起きていることを図にすると次のようになります。

```
Order.place()
   ↓
User.upgradeToPremium()
```

つまり

```
Order更新
User更新
```

が **同じトランザクションの中で行われます。**

そして問題は、開発者が **無意識に別集約を更新できてしまうこと**です。

```kotlin
order.user.upgradeToPremium()
```

このコードを書いた時点では、

* Orderを更新しているのか
* Userを更新しているのか

という **集約の境界が見えにくくなります。**

---

# ID参照の場合

次に、DDDで推奨される **ID参照**の設計を見てみます。

## Order

```kotlin
class Order(
    val id: OrderId,
    val userId: UserId
) {

    fun place() {
        // 注文のドメインロジックのみ。
        // User の関数が呼び出されることはない。
    }
}
```

## User

```kotlin
class User(
    val id: UserId,
    var plan: Plan
) {

    fun upgradeToPremium() {
        plan = Plan.PREMIUM
    }
}
```

## アプリケーションサービス

```kotlin
fun placeOrder(orderId: OrderId) {

    val order = orderRepository.find(orderId)

    order.place()

    val user = userRepository.find(order.userId)
    user.upgradeToPremium()

    orderRepository.save(order)
    userRepository.save(user)
}
```

ここで重要なのは次のコードです。

```kotlin
val user = userRepository.find(order.userId)
```

この処理を書くことで、開発者は

```
別集約を操作している
```

ことを **明確に意識します。**

---

# 違いを整理

## オブジェクト参照

```
Order
 └ User
```

コード

```kotlin
order.user.upgrade()
```

問題

```
どの集約を更新しているか分かりにくい
```

---

## ID参照

```
Order
 └ userId
```

コード

```kotlin
val user = userRepository.find(order.userId)
```

特徴

```
別集約を操作していることが明確
```

つまり、DDDのルールは

```
更新できないようにする
```

のではなく

```
無意識に更新できないようにする
```

ことが目的です。

---

# もう一つ重要な違い：イベント駆動に拡張できる

ID参照にすると、もう一つ大きなメリットがあります。

**イベント駆動設計に拡張できることです。**

例として、注文が確定したらユーザーをアップグレードする処理を考えます。

## Application Service

```kotlin
fun placeOrder(orderId: OrderId) {

    val order = orderRepository.find(orderId)

    order.place()

    orderRepository.save(order)

    eventBus.publish(OrderPlaced(order.userId))
}
```

ここでは **OrderPlaced というドメインイベント**を発行しています。

---

# イベントの定義

```kotlin
data class OrderPlaced(
    val userId: UserId
)
```

これは、「注文が確定した」という事実ベースのイベントです。

DDD のイベント設計では、「～を実行せよ」という命令を書くのではなく、「～が起きた」という事実で設計することが重要です。

なぜなら、一つのイベントが発生した際に、複数の処理が実行される可能性が高く、命令では、正しく表現できないためです。

例えば、注文が確定した際に、以下の処理が実行されることがあります。

- 在庫の変更
- ユーザーのアップグレード
- セールの終了

---

# EventBusの簡易実装

イベントを配送する仕組みを簡単に実装すると次のようになります。

```kotlin
class EventBus {

    // Map のキーがイベントの種類
    // Map の値が、イベント発生時に実行する処理のリストです。
    // 値がリストになっているのは、一つのイベントに対して
    // 複数の処理が必要になる可能性が高いためです。
    private val handlers = mutableMapOf<Class<*>, MutableList<(Any) -> Unit>>()

    fun <T : Any> subscribe(
        eventType: Class<T>,
        handler: (T) -> Unit
    ) {
        handlers
            .getOrPut(eventType) { mutableListOf() }
            .add(handler as (Any) -> Unit)
    }

    // 渡されてきたイベントの型に応じて、
    // そのイベントに紐づく処理を順番に実行します。
    fun publish(event: Any) {
        handlers[event::class.java]?.forEach { handler ->
            // イベントに紐づく処理を一つずつ実行する。
            handler(event)
            // handler は関数型なので、
            // handler.invoke(event)
            // でも同じです。
        }
    }
}
```

---

# イベントハンドラ

```kotlin
class UpgradeUserHandler(
    private val userRepository: UserRepository
) {

    fun handle(event: OrderPlaced) {

        val user = userRepository.find(event.userId)

        user.upgradeToPremium()

        userRepository.save(user)
    }
}
```

---

# ハンドラ登録

アプリ起動時にイベントハンドラを登録します。

```kotlin
val eventBus = EventBus()

val handler = UpgradeUserHandler(userRepository)

eventBus.subscribe(OrderPlaced::class.java) { event ->
    // event は OrderPlaced のインスタンスです。
    // EventBus クラスの publish 関数の引数で渡されます。
    handler.handle(event)
}
```

---

# イベントの流れ

全体の流れは次のようになります。

```
注文確定
   ↓
OrderPlacedイベント発行
   ↓
EventBusがハンドラに通知
   ↓
User集約を更新
```

つまり

```
Orderトランザクション
↓
イベント
↓
User更新
```

という構造にできます。

このようにすると

* トランザクションを分離できる
* 非同期処理に拡張できる
* マイクロサービスにも対応しやすい

というメリットがあります。

---

# まとめ

オブジェクト参照

```
order.user.upgrade()
```

問題

* 集約境界が見えにくい
* 無意識に複数集約を更新してしまう

---

ID参照

```
val user = userRepository.find(order.userId)
```

メリット

* 集約境界が明確
* 別集約操作を意識できる
* イベント駆動設計に拡張できる

---

DDDの重要な考え方は次の一文にまとめられます。

```
集約 = トランザクション境界
```

つまり

```
1トランザクション
   ↓
1集約
```

この境界を守るために、

* 集約ルート
* ID参照
* ドメインイベント

といった設計ルールが使われています。
