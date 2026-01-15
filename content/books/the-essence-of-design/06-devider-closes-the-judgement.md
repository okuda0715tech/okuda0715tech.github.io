+++
title = "第6章：境界は判断を閉じ込める装置である"
weight = 7
+++


- [判断が散らばると、コードは壊れ始める](#判断が散らばるとコードは壊れ始める)
- [境界の役割は「判断を一箇所に押し込む」こと](#境界の役割は判断を一箇所に押し込むこと)
- [Android / Compose における「境界」](#android--compose-における境界)
- [悪い例：UI が判断している](#悪い例ui-が判断している)
- [良い例：境界の内側で判断が完結している](#良い例境界の内側で判断が完結している)
- [ViewModel は「判断を閉じ込める箱」](#viewmodel-は判断を閉じ込める箱)
- [境界が曖昧だと、判断は漏れ出す](#境界が曖昧だと判断は漏れ出す)
- [この章のまとめ](#この章のまとめ)


設計において「境界」という言葉は、
しばしば次のように説明されます。

* 責務の境界
* レイヤーの境界
* モジュールの境界

しかし本書では、境界をもう一段深く捉えます。

**境界とは、判断を閉じ込めるための構造** です。

---

## 判断が散らばると、コードは壊れ始める

設計されていないコードでは、
同じ判断がいろいろなところに散らばります。

例えば、ボタンの有効 / 無効を判断する処理が、
UI 内と ViewModel 内にそれぞれ存在するとします。

判断方法の仕様が変更になった際に、
UI 側の判断は仕様通りに変更したが、
ViewModel 側の変更を忘れた。

というような **構造的な破綻** が起き始めます。

---

## 境界の役割は「判断を一箇所に押し込む」こと

良い設計では、
判断は **境界の内側に閉じ込められます**。

* 境界の外では、判断しない
* 境界の内側で、判断が完結する
* 境界を越えた時点で、前提が保証される

つまり、

> 境界を越えたら、もう迷わない

という状態を作るのが、設計です。

---

## Android / Compose における「境界」

Android / Jetpack Compose では、
典型的に次のような境界が現れます。

* Composable と ViewModel の境界
* ViewModel と UseCase の境界
* UseCase と Repository の境界

重要なのは、
**どこに判断を閉じ込めるかを意識しているか** です。

---

## 悪い例：UI が判断している

```kotlin
@Composable
fun SubmitButton(
    isLoading: Boolean,
    isValid: Boolean,
    onClick: () -> Unit
) {
    Button(
        enabled = !isLoading && isValid,
        onClick = onClick
    ) {
        Text("Submit")
    }
}
```

一見、普通のコードです。
しかしここでは UI が次の判断をしています。

* 今、押していいか？
* この状態は有効か？

この判断が UI にある限り、

* 別の画面でも同じ判断を書く
* 状態が増えるたびに UI が複雑になる
* ViewModel との責務が曖昧になる

という問題が避けられません。

---

## 良い例：境界の内側で判断が完結している

```kotlin
sealed interface SubmitUiState {
    data object Idle : SubmitUiState
    data object Loading : SubmitUiState
    data object Ready : SubmitUiState
}

@Composable
fun SubmitButton(
    state: SubmitUiState,
    onClick: () -> Unit
) {
    Button(
        enabled = state is SubmitUiState.Ready,
        onClick = onClick
    ) {
        Text("Submit")
    }
}
```

この場合、 `isLoading` と `isValid` を見て、
「ボタンの状態を決定する」という
**判断が境界の内側（ViewModel）で既に完結** しています。

この例では、あまり恩恵を感じませんが、
それは、ボタンの状態を決定する要因が二つだけあるためです。

もし、要因が五つくらいあったらどうでしょうか？
UI で判断するのが、厳しくなると想像できるでしょう。

---

## ViewModel は「判断を閉じ込める箱」

```kotlin
sealed interface FormState {
    data class Editing(
        val name: String,
        val isValid: Boolean
    ) : FormState

    data class Ready(
        val name: String
    ) : FormState

    object Submitting : FormState
}
```

```kotlin
class SubmitViewModel : ViewModel() {

    val uiState: StateFlow<SubmitUiState> = ...

    fun onInputChanged(input: Input) {
        // ここで判断を完結させる
        // ( input を判定して、 UI 状態を更新する)
    }

    fun onSubmit(ready: FormState.Ready) {
        // Ready 以外では呼ばれない前提が保証される
    }
}
```

ここで重要なのは、

* UI から不正な状態で呼ばれない
* 呼ばれた時点で前提が成立している

という **保証が構造として成立** していることです。

この保証こそが、
境界が存在する意味です。

---

## 境界が曖昧だと、判断は漏れ出す

判断が漏れ出すと、

* if が増える
* フラグが増える
* コメントが増える
* レビューでの指摘が増える

これは実装の問題ではなく、
**境界設計の失敗** です。

---

## この章のまとめ

* 境界とは、判断を閉じ込める構造である
* 境界の外では判断しない
* 境界の内側で判断を完結させる
* Compose UI は、判断を「表現する側」に徹する

次の章では、
この「境界」が **状態・イベント・責務分割とどう結びつくか**
――つまり、
**設計として破綻しない全体構造** について扱えます。



