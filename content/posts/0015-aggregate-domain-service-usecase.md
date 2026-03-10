+++
title = "DDDにおける「集約」「ドメインサービス」「ユースケース」の違い"
date = "2026-03-10T16:00:00+09:00"
tags = ["DDD", "集約", "ドメインサービス", "ユースケース"]
draft = false
+++


- [DDDにおける「集約」「ドメインサービス」「ユースケース」の違い](#dddにおける集約ドメインサービスユースケースの違い)
- [まず結論](#まず結論)
- [集約（Aggregate）](#集約aggregate)
- [ドメインサービス](#ドメインサービス)
- [ユースケース（Application Service）](#ユースケースapplication-service)
- [ドメインロジックをどこに置くか](#ドメインロジックをどこに置くか)
- [状態を持つドメインサービスはどうなるか](#状態を持つドメインサービスはどうなるか)
  - [ケース1：新しいドメイン概念が生まれた](#ケース1新しいドメイン概念が生まれた)
  - [ケース2：一時的な処理状態](#ケース2一時的な処理状態)
- [判断のための質問](#判断のための質問)
- [よくある失敗](#よくある失敗)
- [まとめ](#まとめ)


## DDDにおける「集約」「ドメインサービス」「ユースケース」の違い

DDD（ドメイン駆動設計）を学び始めると、多くの人が次の疑問にぶつかります。

* 集約とドメインサービスは何が違うのか
* ユースケースはどこまで責任を持つのか
* ドメインロジックはどこに書くべきなのか

この記事では、この3つの概念を整理しながら、**ドメインロジックをどこに置くべきかの判断基準**を解説します。

---

## まず結論

3つの役割は次のように整理できます。

**集約（Aggregate）**

* ドメインの状態を持つ
* 不変条件を守る
* ドメインモデルの中心

**ドメインサービス（Domain Service）**

* エンティティに属さないドメインロジック
* 複数の集約にまたがる処理

**ユースケース（Use Case / Application Service）**

* ユーザー操作を実現する手順
* ドメインモデルを組み合わせる

依存関係は次のようになります。

```
UI
 ↓
UseCase
 ↓
DomainService
 ↓
Aggregate
```

重要なルールは **内側の層は外側を知らないこと** です。

---

## 集約（Aggregate）

集約は **状態と不変条件を守るドメインモデル** です。

例として「口座振替」を考えます。

振替関係には次のルールがあります。

* 振替日は 1〜31
* 「振替元 + 振替先」が同じ振替関係は 1 つだけ (重複不可)

このルールを守る主体が `TransferRelation` です。

```kotlin
class TransferRelation(
    val accountId: AccountId,
    val serviceId: ServiceId,
    private var paymentDay: PaymentDay
) {

    fun changePaymentDay(newDay: PaymentDay) {
        paymentDay = newDay
    }

    fun stop() {
        // 状態変更
    }
}
```

【集約の特徴】

* 状態を持つ
* 不変条件を守る
* ドメインの中心となる概念

DDDでは、 **まずは集約にロジックを入れることを検討します。**

---

## ドメインサービス

ドメインサービスは **集約 (エンティティ) に入らないドメインロジック** を表現します。

典型的なケースは **複数の集約にまたがる処理** です。

【例：銀行振込】

処理内容

* 口座Aから引き落とす
* 口座Bに入金する

この処理は

* AccountA
* AccountB

という **2つの集約** に関係します。

そのため集約の責務にすると不自然になります。

```kotlin
class TransferService {

    fun transfer(from: Account, to: Account, amount: Money) {
        from.withdraw(amount)
        to.deposit(amount)
    }
}
```

ドメインサービスの特徴

* 状態を持たない
* ドメインロジックを表す
* 複数の集約を扱う

---

## ユースケース（Application Service）

ユースケースは **アプリケーションの操作手順** を表します。

【役割】

* ユーザー操作を実現する
* ドメインモデルを組み合わせる
* Repositoryを呼び出す

【例】

```kotlin
class TransferUseCase(
    private val accountRepository: AccountRepository,
    private val transferService: TransferService
) {

    fun execute(fromId: AccountId, toId: AccountId, amount: Money) {

        val from = accountRepository.find(fromId)
        val to = accountRepository.find(toId)

        transferService.transfer(from, to, amount)

        accountRepository.save(from)
        accountRepository.save(to)
    }
}
```

【ユースケースの特徴】

* 手順を書く
* ドメインを呼び出す
* トランザクションを管理する

---

## ドメインロジックをどこに置くか

実務では次の順序で判断すると迷いません。

① **1つの集約の状態だけで完結するか？**

→ 集約に入れる

【例】

* 振替日を変更する
* 注文をキャンセルする

---

② **複数の集約にまたがるか？**

→ ドメインサービス

【例】

* 口座A → 口座Bへの送金

---

③ **それでもドメインの概念に属さないか？**

→ ユースケース

---

## 状態を持つドメインサービスはどうなるか

DDDでは **ドメインサービスは基本的に状態を持ちません。**

もし状態を持つようになった場合、次の可能性があります。

### ケース1：新しいドメイン概念が生まれた

【例】

以下の Service 「振込処理」があったとします。

```kotlin
class TransferService(
    val fromAccountId: AccountId,
    val toAccountId: AccountId,
    val amount: Money
)
```

そこに TransferStatus という状態 (以下) が追加されたとします。

```
PENDING
PROCESSING
COMPLETED
FAILED
```

すると次のようなモデリングが必要になります。

```kotlin
class Transfer(
    val fromAccountId: AccountId,
    val toAccountId: AccountId,
    val amount: Money,
    var status: TransferStatus
)
```

つまり

```
TransferService → Transfer（集約）
```

という **ドメイン概念の発見** が起こります。

---

### ケース2：一時的な処理状態

例えば

* APIリトライ回数
* バッチ処理の進捗

などは **ドメインの状態ではありません。**

この場合は **ユースケース（アプリケーション層）** に置きます。

---

## 判断のための質問

設計で迷ったときは次の質問をします。

**その状態はドメインの概念か？**

YES

→ 新しいエンティティ（集約）

NO

→ アプリケーション層

---

## よくある失敗

DDD初心者がよくやる設計があります。

```
UserService
OrderService
PaymentService
AccountService
```

これは **ドメインモデル貧血症（Anemic Domain Model）** と呼ばれます。

原因は

**ドメインロジックをすべてサービスに入れてしまうこと** です。

DDDでは逆のアプローチを取ります。

* ロジックはまず集約に置く
* サービスは最小限にする

---

## まとめ

今回触れた DDD の 3 つの役割は次の通りです。

**集約**

* 状態を持つ
* 不変条件を守る
* ドメインモデルの中心

**ドメインサービス**

* 集約に入らないドメインロジック
* 複数の集約を扱う

**ユースケース**

* ユーザー操作の手順
* ドメインモデルを組み合わせる

ドメインロジックを配置するときは次の順序で考えます。

1. まず集約に入れる
2. 複数の集約をまたぐ場合はドメインサービス
3. そもそもドメイン概念に関係ない場合はユースケース

DDDでは **ドメインモデルは最初から完成しているものではありません。**

設計を進める中で

* サービスだったものがエンティティになる
* 新しい概念が見つかる

ということがよく起こります。

この **モデルを発見していくプロセス** こそが、DDDの大きな特徴です。

**「この振る舞いは誰の責任か？」**

この問いを繰り返すことで、集約、ドメインサービス、ユースケースは徐々に洗練されていきます。

