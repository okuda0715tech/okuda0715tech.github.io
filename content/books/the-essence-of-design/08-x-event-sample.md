+++
title = "第8章 補足：イベントとは何か（サンプルで理解する）"
weight = 10
+++


- [悪い例：境界を越えたか分からないイベント](#悪い例境界を越えたか分からないイベント)
- [何が問題なのか](#何が問題なのか)
- [良い例：境界を越えたことが分かるイベント](#良い例境界を越えたことが分かるイベント)
  - [状態（未判断の世界）](#状態未判断の世界)
  - [イベント（判断完了の世界）](#イベント判断完了の世界)
  - [ViewModel（判断を完了させる場所）](#viewmodel判断を完了させる場所)
- [ここで何が起きているか](#ここで何が起きているか)
  - [境界の「手前」](#境界の手前)
  - [境界の「向こう側」](#境界の向こう側)
- [イベントが「境界を越えた通知」になる理由](#イベントが境界を越えた通知になる理由)
- [なぜ sealed class が効くのか](#なぜ-sealed-class-が効くのか)
- [この章のまとめ](#この章のまとめ)


第8章では、
本書における「イベント」を次のように定義しました。

**イベントとは、「判断が完了したこと」を表す構造** です。

これは、

* onClick
* onTextChanged
* onResume

といった
**UI やフレームワークが発火する通知** とは、
まったく別の概念です。

ここでは、
この違いがコードとしてどう現れるのかを、
具体例で確認します。

---

## 悪い例：境界を越えたか分からないイベント

まずは、よくある実装から見てみます。

```kotlin
class FormViewModel : ViewModel() {

    private var uiState = FormUiState()

    fun onTextChanged(text: String) {
        uiState = uiState.copy(inputText = text)
    }

    fun onSubmitClicked() {
        // 本当に送信してよい状態か？
        // ここで毎回判断する必要がある
    }
}
```

一見、問題なさそうに見えます。

しかし、この `onSubmit()` には、
次の疑問が常につきまといます。

* 入力は本当に完了しているのか
* バリデーションは終わっているのか
* この時点で「判断」は終わっているのか

コード上では、
**どこで判断が完了したのかが分かりません。**

つまりこの `onSubmit()` は、

* 境界を越えてよいのか分からない
* 毎回、中身を読まないと安全性が判断できない

という状態です。

これは、
**イベントが「判断済み」であることを表現できていない**
典型例です。

---

## 何が問題なのか

問題は、
「onSubmit という関数があること」ではありません。

問題は、

**この呼び出しが、
判断の途中なのか、判断が完了した後なのかが、
型から分からない**

ことです。

そのため、

* ViewModel 内で再度 if が必要になり
* レビューやテストに判断が押し出され
* 「正しく使ってね」という前提が増えていきます

これは、
第3章で述べた
**判断が構造で表現されていない状態** です。

---

## 良い例：境界を越えたことが分かるイベント

ポイントは 1 つだけです。

> **ViewModel の中に「未判断の世界」と「判断済みの世界」を両方置き、
> その境界を Event で表現する**

---

### 状態（未判断の世界）

```kotlin
data class FormUiState(
    val inputText: String = ""
)
```

これは、

* 入力途中
* 編集中
* まだ Submit してよいか分からない

**未判断の世界の状態** です。

---

### イベント（判断完了の世界）

```kotlin
sealed interface FormEvent {
    data class Submit(
        val text: String
    ) : FormEvent
}
```

この Event は、

* バリデーションが終わり
* Submit してよいと判断された

**判断完了の結果** だけを表します。

---

### ViewModel（判断を完了させる場所）

```kotlin
class FormViewModel : ViewModel() {

    private var uiState = FormUiState()

    fun onTextChanged(text: String) {
        uiState = uiState.copy(inputText = text)
    }

    fun onSubmitClicked() {
        val text = uiState.inputText

        // ここで判断を行う
        if (text.isBlank()) {
            // 判断を通過できなかったデータは境界を越えられない
            return
        }

        val event = FormEvent.Submit(text)

        handleEvent(event)
    }

    private fun handleEvent(event: FormEvent) {
        when (event) {
            is FormEvent.Submit -> {
                // ここでは「判断済み」であることを信じてよい
                submit(event.text)
            }
        }
    }

    private fun submit(text: String) {
        // 安全に処理できる
    }
}
```

---

## ここで何が起きているか

この ViewModel の中には、**明確な境界** があります。

### 境界の「手前」

* `FormUiState`
* `onTextChanged`
* `onSubmitClicked`
* if による検証

ここは **未判断の世界** です。

---

### 境界の「向こう側」

* `FormEvent.Submit`
* `handleEvent`
* `submit`

ここは **判断済みの世界** です。

---

## イベントが「境界を越えた通知」になる理由

この構造では、

* 判断は UI 層側で完了し
* Event がその結果を表し
* ViewModel は「判断済みの世界」だけを扱う

という分離が成立します。

つまり、

**イベントとは、
「判断が完了し、境界を越えてよい状態になった」
という事実の通知** なのです。

---

## なぜ sealed class が効くのか

`sealed class` を使うことで、

* 判断が終わっていない状態を Event にできない
* 種類が増えたらコンパイルエラーで気づける
* 「何が起こりうるか」が構造として閉じる

という効果が得られます。

これはまさに、

**判断を構造で表現している**
状態です。

---

## この章のまとめ

* 本書におけるイベントは、UI イベントではない
* イベントとは、「判断が完了したこと」を表す構造
* Event が生成できた時点で、境界を越えてよい
* sealed class は、「判断済み」を型で保証する

第8章で述べた抽象的な定義は、
このように **コードとして具体化** されます。
