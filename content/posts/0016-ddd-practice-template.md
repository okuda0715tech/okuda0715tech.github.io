+++
title = "ドメイン設計テンプレート"
date = "2026-03-11T10:00:00+09:00"
tags = ["DDD", "テンプレート"]
draft = false
+++

このドキュメントは、DDD（ドメイン駆動設計）に基づいて、ソフトウェアを設計するためのテンプレートです。

上から順番にテンプレートを埋めていくことで、 DDD に基づいたドメイン周りの設計がスムーズに行えるように作っています。

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
- [4. エンティティ候補](#4-エンティティ候補)
- [5. 不変条件（Invariant）](#5-不変条件invariant)
- [6. 状態遷移](#6-状態遷移)
  - [【例】TransferRelation](#例transferrelation)
    - [状態](#状態)
    - [状態遷移](#状態遷移)
- [7. 集約設計](#7-集約設計)
  - [集約一覧](#集約一覧)
  - [集約詳細](#集約詳細)
  - [【例】TransferRelation](#例transferrelation-1)
    - [属性](#属性)
    - [不変条件](#不変条件)
    - [ドメインメソッド](#ドメインメソッド)
- [8. ドメインサービス](#8-ドメインサービス)
  - [サービス一覧](#サービス一覧)
  - [サービス詳細](#サービス詳細)
  - [【例】TransferService](#例transferservice)
- [9. リポジトリ](#9-リポジトリ)
- [10. ユースケース設計](#10-ユースケース設計)
  - [UseCase一覧](#usecase一覧)
  - [UseCase詳細](#usecase詳細)
  - [【例】CreateTransferRelationUseCase](#例createtransferrelationusecase)
    - [処理](#処理)

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

【例】

| 用語             | 意味                     |
| ---------------- | ------------------------ |
| Account          | 銀行口座                 |
| Service          | 支払い先サービス         |
| TransferRelation | 口座とサービスの振替関係 |

---

# 4. エンティティ候補

ドメインの中心となる概念。

【例】

* Account
* Service
* TransferRelation
* TransferSchedule

---

# 5. 不変条件（Invariant）

絶対に破ってはいけないルール。

【例】

* 振替関係は必ず1つの口座を持つ
* 振替関係は必ず1つのサービスを持つ
* 振替日は1〜31
* 同じ口座 + サービスの振替関係は1つだけ

---

# 6. 状態遷移

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

# 7. 集約設計

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

# 8. ドメインサービス

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

# 9. リポジトリ

集約の永続化を担当。

【例】

* AccountRepository
* TransferRelationRepository

---

# 10. ユースケース設計

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

