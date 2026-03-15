+++
title = "Boolean に潰された「状態」"
date = "2026-03-04T12:00:00+09:00"
categories = ["Architecture"]
tags = ["State"]
draft = false
+++

- [Boolean に潰された「成功」 ― リファクタリングで気づいた抽象化の罠](#boolean-に潰された成功--リファクタリングで気づいた抽象化の罠)
  - [はじめに](#はじめに)
  - [元のコード](#元のコード)
  - [リファクタリングでやったこと](#リファクタリングでやったこと)
  - [最初は「うっかり」だと思った](#最初はうっかりだと思った)
  - [本当の原因](#本当の原因)
  - [抽象化の粒度が粗すぎた](#抽象化の粒度が粗すぎた)
  - [型で守る設計](#型で守る設計)
  - [学び](#学び)
  - [終わりに](#終わりに)


# Boolean に潰された「成功」 ― リファクタリングで気づいた抽象化の罠

## はじめに

ViewModel の保存処理をリファクタリングしていたとき、
私は一度コードを壊しました。

原因は単純な「うっかりミス」に見えました。

しかし振り返ってみると、問題はもっと深いところにありました。

それは、

> 「成功」という概念を Boolean に潰してしまったこと

でした。

この記事では、その過程と学びを書きます。

---

## 元のコード

保存処理は、新規作成と更新で分岐していました。

```kotlin
private fun saveData() {
    viewModelScope.launch {
        val isExistingItem: Boolean = (取得)

        if (destId == 0) {
            // 新規作成
            val resultSuccess = directDebitDefRepo.createDestination(...)

            if (resultSuccess) {
                // フォームの初期化
                _formInputState.update { FormInputState() }
                showSuccess()
            } else {
                showFailure()
            }

        } else {
            // 更新
            val resultSuccess = directDebitDefRepo.updateDestination(...)

            if (resultSuccess) {
                showSuccess()
            } else {
                showFailure()
            }
        }
    }
}
```

ポイントはここです。

* create 成功 → フォーム初期化あり
* update 成功 → フォーム初期化なし

つまり、

```
create成功 ≠ update成功
```

でした。

---

## リファクタリングでやったこと

私は「成功時の処理はほぼ同じ」と考え、共通化しました。

しかしその結果、

* create のときだけ必要だったフォーム初期化を消してしまいました。

そして動作が壊れました。

---

## 最初は「うっかり」だと思った

私はこう思いました。

> create と update の違いを吸収しすぎた、単なるミスだ。

でも本当にそうでしょうか？

---

## 本当の原因

問題はここでした。

```
true  = 成功
false = 失敗
```

Boolean は 2つの状態 です。

しかし実際のドメインはこうでした。

* CreatedSuccess
* UpdatedSuccess
* Failure

3つの状態があったものを、 Boolean に圧縮していました。

その瞬間に、

```
CreatedSuccess と UpdatedSuccess が同一化した
```

のです。

そして「成功は同じ」と思い込んでしまった。

---

## 抽象化の粒度が粗すぎた

今回の失敗はロジックの問題ではありませんでした。

抽象化の粒度の問題でした。

「成功」という言葉を、技術的成功（DB操作成功）として扱ってしまった。

しかし UI から見ると、

* 新規作成成功
* 更新成功

は意味が違う。

このズレがバグを生みました。

---

## 型で守る設計

もし最初からこうしていたらどうだったでしょうか。

```kotlin
sealed interface SaveResult {
    data object Created : SaveResult
    data object Updated : SaveResult
    data object Failed : SaveResult
}
```

そして：

```kotlin
when (result) {
    Created -> {
        _formInputState.update { FormInputState() }
        showSuccess()
    }
    Updated -> {
        showSuccess()
    }
    Failed -> showFailure()
}
```

この設計では、

Created と Updated を無意識に同一視することはできません。

型が設計を強制します。

型が未来の自分を守ります。

---

## 学び

今回の経験から得たことは2つです。

1. Boolean は情報を潰す
2. ドメインの意味をそのまま型に表すと事故が減る

---

## 終わりに

今回のバグは、小さなものでした。

しかし、

> 抽象化は便利だが、意味を削りすぎると危険

という重要な気づきを得ました。

リファクタリングはコードを綺麗にする作業ではありません。

ドメインの意味を正しく表現し直す作業です。

そしてときどき、
型が私たちを守ってくれます。

