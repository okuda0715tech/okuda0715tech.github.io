+++
title = "SQL の条件を固定しない"
date = "2026-02-11T17:30:00+09:00"
tags = ["Dao", "SQL"]
draft = false
+++


- [SQL の条件を固定しない](#sql-の条件を固定しない)
  - [はじめに](#はじめに)
  - [改善前の実装](#改善前の実装)
    - [DAO にドメインの意味が入り込んでいた](#dao-にドメインの意味が入り込んでいた)
  - [問題意識](#問題意識)
  - [改善方針](#改善方針)
  - [改善後の実装](#改善後の実装)
    - [DAO：検索軸だけを知る](#dao検索軸だけを知る)
    - [Repository / UseCase：意味を与える](#repository--usecase意味を与える)
  - [この設計で得られたメリット](#この設計で得られたメリット)
    - [1. SQL / DAO の数が増えにくくなった](#1-sql--dao-の数が増えにくくなった)
    - [2. DAO の責務が明確になった](#2-dao-の責務が明確になった)
    - [3. ドメインの変更に強くなった](#3-ドメインの変更に強くなった)
  - [おわりに](#おわりに)


# SQL の条件を固定しない

DAO の責務を整理して、ドメインに意味を寄せる設計改善

## はじめに

Android アプリ開発において、
「とりあえず動く」状態から一段上の設計に進もうとすると、
**ViewModel・UseCase・Repository・DAO の責務の境界**で悩むことが多い。

今回は、

* SQL で条件を固定していた実装を見直し
* 条件をパラメータ化して SQL の数を減らし
* その「意味」を Domain / UseCase 側で与える

という設計改善を行ったので、その考え方をまとめる。

---

## 改善前の実装

### DAO にドメインの意味が入り込んでいた

もともと DAO には、次のような Query が定義されていた。

```kotlin
@Query("SELECT * FROM transfer_item WHERE isSourceItem = 1")
fun observeSources(): Flow<List<TransferItemEntity>>
```

この実装はシンプルで分かりやすいが、次の問題を抱えていた。

* DAO が「Source（振替元）」という**ドメインの意味**を知っている
* 条件が増えるたびに SQL / DAO メソッドが増える
* 将来的に Query が爆発する兆候がある

---

## 問題意識

ここで違和感を覚えたのは、次の点だった。

* 「振替元である」というのは**ドメインの意味**
* DAO は本来、**永続化の都合（カラムやインデックス）**だけを知るべき
* 「なぜ isSourceItem = 1 なのか」を DAO が理解しているのは不自然

つまり、

> DAO が「意味」を持ち始めているのではないか？

という疑問である。

---

## 改善方針

改善の方向性はシンプルに次の 2 点とした。

1. **SQL の条件を固定せず、パラメータ化する**
2. **条件の意味づけは Domain / UseCase 側で行う**

---

## 改善後の実装

### DAO：検索軸だけを知る

SQL を次のように変更した。

```kotlin
@Query("SELECT * FROM transfer_item WHERE isSourceItem = :isSource")
fun observeByIsSource(isSource: Boolean): Flow<List<TransferItemEntity>>
```

この DAO は、

* `isSourceItem` というカラムがある
* Boolean で絞り込める

という **検索の軸** しか知らない。

「それが振替元なのか」「どの画面で使うのか」は一切知らない。

---

### Repository / UseCase：意味を与える

```kotlin
class LoadSourceItemsUseCase(
    private val repository: TransferItemRepository
) {
    fun execute(): Flow<List<SourceItem>> =
        repository.observeByIsSource(isSource = true)
            .map { it.map { entity -> entity.toDomain() } }
}
```

ここで初めて、

* `isSource = true` が
* 「振替元を取得する」という **ドメインの意味**

を持つ。

DAO → Repository → UseCase と層を上がるにつれて、

* 実装寄り → 意味寄り

に責務が移動している。

---

## この設計で得られたメリット

### 1. SQL / DAO の数が増えにくくなった

* 条件ごとに Query を生やす必要がなくなった
* 将来的に `isActive` などが増えても拡張しやすい

---

### 2. DAO の責務が明確になった

* DAO は **「どうやって取るか」** だけを担当
* **「なぜ取るか」** は Domain / UseCase の責務

---

### 3. ドメインの変更に強くなった

* 「振替元」の定義が変わっても
* 修正箇所は UseCase / Repository に閉じる
* SQL や DAO への影響を最小限にできる

---

## おわりに

SQL の条件を固定するか、パラメータ化するかは
一見すると小さな実装差に見える。

しかし、

* **どの層が意味を知るべきか**
* **どこで判断を下すべきか**

を意識すると、設計の責務分離が一段クリアになる。

DAO を「賢く」しすぎず、
Domain / UseCase に意味を寄せる。

このバランスは、長く運用するアプリほど効いてくると感じている。

