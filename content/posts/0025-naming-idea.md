+++
title = "「起きたこと」で名前を付けるか、「意図」で名前を付けるか"
date = "2026-06-02T19:00:00+09:00"
categories = ["Architecture"]
tags = ["Naming"]
draft = false
+++


# 「起きたこと」で名前を付けるか、「意図」で名前を付けるか

アプリ開発をしていると、

* `OnClickPayment`
* `OnClickDelete`
* `OpenPaymentEdit`
* `ShowDeleteDialog`

のように、イベントや関数の名前をどう付けるべきか迷うことがあります。

特に ViewModel を使った設計では、

> 「起きたこと」を名前にするべきか
> 「やりたいこと（意図）」を名前にするべきか

で悩むことがよくあります。

## 結論

私は次のように考えています。

* UI → ViewModel は「起きたこと」
* ViewModel → UI は「意図」

で名前を付けると整理しやすくなります。

## UI → ViewModel は「起きたこと」

例えば Compose の画面から ViewModel にイベントを伝える場合です。

```kotlin
viewModel.onClickPayment()

viewModel.onClickDelete()

viewModel.onTextChanged(text)
```

ここで ViewModel に伝えたいのは、

「ユーザーが何をしたか」

です。

まだ ViewModel は、その結果として何を行うかを決めていません。

そのため、

```kotlin
fun onClickPayment()

fun onClickDelete()

fun onTextChanged(text: String)
```

のような名前が自然です。

## ViewModel → UI は「意図」

一方で、ViewModel から UI に送るイベントは少し性質が異なります。

例えば、

```kotlin
sealed class UiEvent {
    data class OpenPaymentEdit(
        val paymentId: Int
    ) : UiEvent()

    data object ShowDeleteDialog : UiEvent()

    data class ShowSnackbar(
        val messageRes: Int
    ) : UiEvent()
}
```

ここで ViewModel が伝えたいのは、

「何が起きたか」

ではありません。

「UI に何をしてほしいか」

です。

つまり、

```text
支払編集画面を開いてほしい
削除確認ダイアログを表示してほしい
Snackbar を表示してほしい
```

という意図を表現しています。

そのため、

```kotlin
OpenPaymentEdit
ShowDeleteDialog
ShowSnackbar
```

のような名前の方が自然になります。

## 実際に違和感が生まれた例

ある画面で、最初は支払情報をタップした場合だけ支払編集画面へ遷移していました。

そのため、

```kotlin
UiEvent.OnClickPayment(paymentId)
```

というイベント名を付けていました。

しかし後になって、

* 支払情報をタップ
* 支払元をタップ
* 支払先をタップ

のどれでも支払編集画面へ遷移したくなりました。

すると、

```kotlin
UiEvent.OnClickPayment(paymentId)
```

という名前に違和感が出てきます。

なぜなら、このイベントは

```text
支払情報がクリックされた
```

を表しているわけではなく、

```text
支払編集画面を開いてほしい
```

を表しているからです。

この場合は、

```kotlin
UiEvent.OpenPaymentEdit(paymentId)
```

の方が本質を表現しています。

## 「クリックされたもの」ではなく「やりたいこと」に着目する

例えば、

```kotlin
fun onClickPayment()

fun onClickPayerName(id: Int)

fun onClickPayeeName(id: Int)
```

という 3 つの UI イベントがあったとします。

最終的にどれも同じ処理になるなら、

```kotlin
private fun openPaymentEdit(id: Int)
```

にまとめられます。

```kotlin
fun onClickPayment() {
    openPaymentEdit(paymentId)
}

fun onClickPayerName(id: Int) {
    openPaymentEdit(id)
}

fun onClickPayeeName(id: Int) {
    openPaymentEdit(id)
}
```

この時 ViewModel にとって重要なのは、

```text
何がクリックされたか
```

ではなく、

```text
どの支払情報を編集したいか
```

です。

そのため、内部処理や UiEvent は意図ベースの名前の方が長持ちします。

## すべてを意図ベースにする必要はない

ただし、すべてを意図ベースにする必要はありません。

例えば、

```kotlin
fun onClickDeletePayment()
```

は十分わかりやすい名前です。

これを、

```kotlin
fun openDeleteConfirmDialog()
```

にすると、

「ユーザーが削除ボタンを押した」

という事実が見えにくくなります。

UI から入ってくるイベントは、

```kotlin
onClickXxx
onLongClickXxx
onCheckedChangeXxx
onTextChangedXxx
```

のように「起きたこと」で表現する方が理解しやすい場合が多いです。

## まとめ

私が現在よく使う考え方は次の通りです。

* UI → ViewModel は「起きたこと」
* ViewModel → UI は「意図」
* ViewModel の内部処理も「意図」を表現する名前を優先する

例えば、

```kotlin
// UI → ViewModel
onClickPayment()
onClickDelete()

// ViewModel → UI
OpenPaymentEdit()
ShowDeleteDialog()
ShowSnackbar()
```

のような形です。

開発を続けていると、

「この名前、なんだかしっくりこないな」

と感じる瞬間があります。

それは設計が悪くなったのではなく、

「本当に表現したい概念」が見えてきたサインかもしれません。

実際、`OnClickPayment` に違和感を覚えたのは、「クリックされたこと」ではなく、「支払編集画面を開くこと」が本質だったと気付けたからでした。

名前に迷ったときは、

> これは「起きたこと」なのか、それとも「意図」なのか？

を考えてみると、整理しやすくなると思います。
