+++
title = "ドメイン設計テンプレート（補強版）"
date = "2026-03-11T10:00:00+09:00"
tags = ["DDD", "テンプレート"]
draft = false
+++

このドキュメントは、 [ドメイン設計テンプレート](https://design.okuda-studio.com/posts/0016-ddd-practice-template/) に以下の項目を加えた補強版となっています。

* 4.境界コンテキスト
* 5.コンテキストマップ
* 7.値オブジェクト
* 12.ドメインイベント
* 15.集約境界図
* 16.不変条件の責任

---

- [1. システム概要](#1-システム概要)
  - [システムの目的](#システムの目的)
  - [対象ユーザー](#対象ユーザー)
  - [解決したい課題](#解決したい課題)
- [2. ユースケース](#2-ユースケース)
  - [ユースケース一覧](#ユースケース一覧)
  - [ユースケース詳細](#ユースケース詳細)
  - [【例】振替関係を作成する](#例振替関係を作成する)
    - [入力](#入力)
    - [処理概要](#処理概要)
    - [出力](#出力)
- [3. ユビキタス言語](#3-ユビキタス言語)
  - [概念](#概念)
  - [候補とその言葉のイメージ](#候補とその言葉のイメージ)
  - [賛否の意見](#賛否の意見)
  - [言葉の決定とその採用理由](#言葉の決定とその採用理由)
  - [ユビキタス言語辞書](#ユビキタス言語辞書)
- [4. 境界コンテキスト](#4-境界コンテキスト)
- [5. コンテキストマップ](#5-コンテキストマップ)
- [6. エンティティ候補](#6-エンティティ候補)
- [7. 値オブジェクト](#7-値オブジェクト)
- [8. 不変条件（Invariant）](#8-不変条件invariant)
- [9. 状態遷移](#9-状態遷移)
  - [【例】TransferRelation](#例transferrelation)
    - [状態](#状態)
    - [状態遷移](#状態遷移)
- [10. 集約設計](#10-集約設計)
  - [集約一覧](#集約一覧)
  - [集約詳細](#集約詳細)
  - [【例】TransferRelation](#例transferrelation-1)
    - [属性](#属性)
    - [不変条件](#不変条件)
    - [ドメインメソッド](#ドメインメソッド)
- [11. ドメインサービス](#11-ドメインサービス)
  - [サービス一覧](#サービス一覧)
  - [サービス詳細](#サービス詳細)
  - [【例】TransferService](#例transferservice)
- [12. ドメインイベント](#12-ドメインイベント)
- [イベント詳細](#イベント詳細)
  - [【例】TransferRelationCreated](#例transferrelationcreated)
    - [発生タイミング](#発生タイミング)
    - [ペイロード](#ペイロード)
- [13. リポジトリ](#13-リポジトリ)
- [14. ユースケース設計](#14-ユースケース設計)
  - [UseCase一覧](#usecase一覧)
  - [UseCase詳細](#usecase詳細)
  - [【例】CreateTransferRelationUseCase](#例createtransferrelationusecase)
    - [処理](#処理)
- [15. 集約境界図（Aggregate Boundary）](#15-集約境界図aggregate-boundary)
  - [【例】TransferRelation](#例transferrelation-2)
    - [集約名 / Aggregate Root](#集約名--aggregate-root)
    - [内部エンティティ](#内部エンティティ)
    - [値オブジェクト](#値オブジェクト)
    - [集約境界](#集約境界)
    - [外部参照](#外部参照)
    - [集約ルール](#集約ルール)
- [16. 不変条件の責任（Invariant Responsibility）](#16-不変条件の責任invariant-responsibility)
  - [不変条件一覧](#不変条件一覧)
  - [不変条件の責任](#不変条件の責任)
    - [振替日は1〜31](#振替日は131)
    - [同じ口座 + サービスの振替関係は1つだけ](#同じ口座--サービスの振替関係は1つだけ)
    - [振替関係は必ず口座を持つ](#振替関係は必ず口座を持つ)
    - [振替関係は必ずサービスを持つ](#振替関係は必ずサービスを持つ)
  - [不変条件の配置ルール](#不変条件の配置ルール)
  - [例](#例)

# 1. システム概要

## システムの目的

このシステムは何を解決するものか。

【例】

* フリーランスの固定費を管理する
* 口座振替の関係を可視化する

---

## 対象ユーザー

このシステムのユーザー。

【例】

* 個人
* フリーランス
* 企業

---

## 解決したい課題

【例】

* 固定費の支払い関係が分かりにくい
* 口座振替の全体構造を把握できない

---

# 2. ユースケース

ユーザーがシステムで行う操作を列挙する。

## ユースケース一覧

【例】

* ○○を作成する
* ○○を変更する
* ○○を削除する
* ○○を確認する

---

## ユースケース詳細

---

## 【例】振替関係を作成する

### 入力

【例】

* accountId
* serviceId
* paymentDay

### 処理概要

【例】

1. 口座を取得
2. サービスを取得
3. 振替関係を作成
4. 保存

### 出力

【例】

* 振替関係ID

---

# 3. ユビキタス言語

ドメインで使う用語を定義する。

概念は何度か練り直すと、よい概念に落ち着きやすい。

日本語だけではなく、英語でも定義する。

英語は実装時に使用する。

過不足なく正確に伝わる英単語が存在するかどうかも、日本語選定の基準となります。

---

## 概念

【例】

お金を支払う対象となるサービス。

## 候補とその言葉のイメージ

【例】

* 振替先
  * 口座からお金が引き落とされる対象
* 支払い先
  * ユーザーがお金を支払う相手
* 請求元
  * 請求を発行する会社
* 引き落とし先
  * 銀行口座から自動的にお金が引き落とされる相手

## 賛否の意見

【例】

* 振替先
  * 銀行用語っぽい
  * 一般ユーザーが使う言葉ではない
* 支払い先
  * 手動支払いも含みそう
* 請求元
  * 請求書前提
* 引き落とし先
  * 意味は近いが長い

## 言葉の決定とその採用理由

【例】

* 採用
  * 支払い先
* 理由
  * 一般ユーザーが理解しやすい
  * サービスにも会社にも使える

---

## ユビキタス言語辞書

【例】

| 用語(日本語) | 用語(英語)      | 意味                                         |
| ------------ | --------------- | -------------------------------------------- |
| 支払元       | Payer           | 発生した支払いに対して、お金が出ていくところ |
| 支払先       | Payee           | お金を支払う対象となるサービス               |
| 支払関係     | PaymentRelation | 支払先と支払元の紐づけ                       |

---

# 4. 境界コンテキスト

ドメインをコンテキストに分割する。

【例】

* PaymentContext
* AccountContext
* ServiceContext

---

# 5. コンテキストマップ

コンテキスト間の関係。

```
AccountContext
      ↓
PaymentContext
      ↓
ServiceContext
```

関係タイプ

* Customer/Supplier
* Shared Kernel
* Conformist

---

# 6. エンティティ候補

ドメインの中心となる概念。

【例】

* Account
* Service
* TransferRelation
* TransferSchedule

---

# 7. 値オブジェクト

値として扱う概念。

【例】

* PaymentDay
* Money
* AccountId
* ServiceId

---

# 8. 不変条件（Invariant）

絶対に破ってはいけないルール。

【例】

* 振替関係は必ず1つの口座を持つ
* 振替関係は必ず1つのサービスを持つ
* 振替日は1〜31
* 同じ口座 + サービスの振替関係は1つだけ

---

# 9. 状態遷移

状態を持つエンティティについて整理する。

## 【例】TransferRelation

### 状態

【例】

* Created
* Active
* Stopped
* Deleted

### 状態遷移

【例】

```
Created → Active
Active → Stopped
Stopped → Active
Active → Deleted
```

---

# 10. 集約設計

## 集約一覧

【例】

* TransferRelation
* Account

---

## 集約詳細

---

## 【例】TransferRelation

### 属性

【例】

* accountId
* serviceId
* paymentDay
* status

### 不変条件

【例】

* 振替日は1〜31
* 同じ口座 + サービスの振替関係は1つだけ

### ドメインメソッド

【例】

* changePaymentDay()
* stop()
* resume()

---

# 11. ドメインサービス

エンティティに属さないドメインロジック。

## サービス一覧

【例】

* TransferService

---

## サービス詳細

---

## 【例】TransferService

* 責務
  * 【例】
  * 口座Aから口座Bへの送金
* メソッド
  * 【例】
  * `transfer(fromAccount, toAccount, amount)`

---

# 12. ドメインイベント

重要なドメインの出来事。

【例】

* TransferRelationCreated
* PaymentDayChanged
* TransferStopped

---

# イベント詳細

---

## 【例】TransferRelationCreated

### 発生タイミング

振替関係作成時

### ペイロード

* transferRelationId
* accountId
* serviceId

---

# 13. リポジトリ

集約の永続化を担当。

【例】

* AccountRepository
* TransferRelationRepository

---

# 14. ユースケース設計

## UseCase一覧

【例】

* CreateTransferRelationUseCase
* StopTransferRelationUseCase

---

## UseCase詳細

---

## 【例】CreateTransferRelationUseCase

### 処理

【例】

1. 振替関係の作成
   1. account取得
   2. service取得
   3. TransferRelation生成
   4. 保存

---

# 15. 集約境界図（Aggregate Boundary）

* 集約の内部構造と境界を明確にする。
* 「どこまでが1つの整合性の単位なのか」を定義する。

## 【例】TransferRelation

### 集約名 / Aggregate Root

TransferRelation

### 内部エンティティ

【例】

* TransferSchedule

### 値オブジェクト

* PaymentDay
* TransferRelationId
* AccountId
* ServiceId

### 集約境界

```
TransferRelation (Aggregate Root)
 ├ transferRelationId
 ├ accountId
 ├ serviceId
 ├ paymentDay
 └ status
```

### 外部参照

この集約が参照する他集約

* AccountId
* ServiceId

※ 集約間は **ID参照のみ** とする

### 集約ルール

* 外部からは **Aggregate Root を通してのみ操作可能**
* 集約内部の整合性は **トランザクションで保証**

---

# 16. 不変条件の責任（Invariant Responsibility）

不変条件を「どこで守るのか」を明確にする。

DDDではこれが非常に重要。

---

## 不変条件一覧

【例】

* 振替日は1〜31
* 同じ口座 + サービスの振替関係は1つだけ
* 振替関係は必ず口座を持つ
* 振替関係は必ずサービスを持つ

---

## 不変条件の責任

### 振替日は1〜31

* 責任
  * PaymentDay（ValueObject）
* 理由
  * 値として常に正しい状態を保証する。

---

### 同じ口座 + サービスの振替関係は1つだけ

* 責任
  * TransferRelationRepository
* 理由
  * 既存データを確認する必要があるため。

---

### 振替関係は必ず口座を持つ

* 責任
  * TransferRelation（Aggregate）
* 理由
  * 生成時に保証する。

---

### 振替関係は必ずサービスを持つ

* 責任
  * TransferRelation（Aggregate）
* 理由
  * 生成時に保証する。

---

## 不変条件の配置ルール

不変条件は次の順序で配置する。

1. **ValueObject**
2. **Aggregate**
3. **DomainService**
4. **Repository**

できるだけ **内側に置く**。

---

## 例

```
PaymentDay
 └ 1〜31

TransferRelation
 └ 必ずAccountを持つ

TransferRelationRepository
 └ 重複禁止
```


