+++
title = "Kotlin では null を避けるべきか？実はすごく便利だった"
date = "2026-06-06T17:00:00+09:00"
categories = ["Architecture"]
tags = ["Type Safety"]
draft = false
+++

- [今回のケース](#今回のケース)
- [ふと気付いたこと](#ふと気付いたこと)
- [Kotlinは「nullを排除する言語」ではない](#kotlinはnullを排除する言語ではない)
- [今回は nullable の方が自然だった](#今回は-nullable-の方が自然だった)
- [sealed class を使うべきケース](#sealed-class-を使うべきケース)
- [学び](#学び)

---

Kotlin を書いていると、「null をできるだけ使わない」という考え方をよく目にします。

私自身も、状態を表現するときは sealed class を積極的に使うようにしています。

しかし、ある画面の選択状態を実装しているときに、「本当に nullable を避けるべきなのか？」と考える機会がありました。

## 今回のケース

支払先を選択する画面があり、選択状態を以下のように表現していました。

```kotlin
sealed interface SelectionState {
    data object None : SelectionState
    data class Selected(val id: Int) : SelectionState
}
```

未選択なら `None`、選択済みなら `Selected(id)` です。

一見すると型安全で分かりやすそうです。

しかし UI 側では、

```kotlin
when (val selectionState = uiState.selectionState) {
    is SelectionState.Selected -> {
        onClickSave(selectionState.id)
    }
    SelectionState.None -> Unit
}
```

のような分岐が必要になります。

さらに、UIState の中から選択された ID を取得したい場面でも、毎回 `when` が必要になります。

## ふと気付いたこと

今回の状態は、

* 未選択
* 選択済み(IDあり)

の2パターンしかありません。

これを別の形で書くと、

```kotlin
Int?
```

と同じ情報量です。

つまり、

```text
None
or
Selected(id)
```

は、

```text
null
or
Int
```

と本質的に同じ状態を表現しています。

## Kotlinは「nullを排除する言語」ではない

Kotlin はよく「null を排除する言語」と言われます。

しかし実際には、

> null があり得ることを型で表現する言語

です。

例えば、

```kotlin
val selectedId: Int?
```

は、

「選択されていない状態が存在する」

という事実を型で表現しています。

これは Kotlin の思想に反しているわけではありません。

むしろ、存在しうる状態を正しく型に表現しています。

## 今回は nullable の方が自然だった

今回のケースでは、 nullable にしたことで、

```kotlin
data class Success(
    val selectedId: Int? = null,
    val payments: List<Item>,
)
```

のようにシンプルに表現できました。また、

```kotlin
val saveButtonEnabled: Boolean
    get() = selectedId != null
```

と書くこともできますし、

```kotlin
uiState.selectedId?.let(onClickSave)
```

のようにも扱えます。

余計な状態オブジェクトや `when` 分岐も不要になります。

また、 `selectedId` という性質からも、「 null の場合はおそらく未選択だろう」という想像もつきやすいです。

## sealed class を使うべきケース

もちろん、sealed class が不要という話ではありません。

例えば、

```kotlin
sealed interface SelectionState {
    data object None : SelectionState
    data class Single(val id: Int) : SelectionState
    data class Multiple(val ids: Set<Int>) : SelectionState
}
```

のように状態が増える場合や、状態の中に保持したいプロパティが多い場合は、 nullable よりも sealed class の方が分かりやすくなります。

## 学び

「nullable を使わないこと」が目的になってしまうと、かえってコードが複雑になることがあります。

今回の学びは、

> 状態が本当に `null / 値あり` の2パターンしかないなら、nullable が最も自然な表現になることもある

ということでした。

大切なのは null を排除することではなく、

> ドメインの状態を最も分かりやすく表現できる型を選ぶこと

なのだと思います。

最後に一言付け加えるなら、

> 型で表現できることは型で保証し、実行時の状態に依存することは `require` や例外などで検出する

という考え方も合わせて意識すると、設計の判断がしやすくなると思います。
