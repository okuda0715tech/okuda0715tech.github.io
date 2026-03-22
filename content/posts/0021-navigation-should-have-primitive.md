+++
title = "Jetpack Compose Navigation のパラメータ指定の罠"
date = "2026-03-18T11:00:00+09:00"
categories = ["Architecture"]
tags = ["UI"]
draft = false
+++

- [すぐに思いつく実装](#すぐに思いつく実装)
- [「新規作成 or 編集」を型で表現する](#新規作成-or-編集を型で表現する)
- [なぜエラーになるのか](#なぜエラーになるのか)
- [正しい設計：Navigation はプリミティブ型を渡す](#正しい設計navigation-はプリミティブ型を渡す)
- [重要なポイント](#重要なポイント)
  - [Navigationの責務](#navigationの責務)
  - [UI の責務](#ui-の責務)

---

一覧画面から「新規作成」と「編集」の両方に遷移するケースは、アプリ開発でよくあります。

例えば以下のような仕様です：

* リストのアイテムをタップ → 編集画面へ
* 「追加」ボタンをタップ → 新規作成画面へ
* 画面UIは同じ（内部の動作だけ異なる）

## すぐに思いつく実装

すぐに思いつく実装方法は以下ではないでしょうか？

Navigation の定義は以下の通り

```kotlin
    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier,
    ) {
        // リスト画面
        composable<ItemList> {
            PayeeListScreen(
                onClickAdd = {
                    navController.navigate(RegisterItem(null))
                },
                onClickItem = { itemId ->
                    navController.navigate(RegisterItem(itemId))
                }
            )
        }

        // 登録・編集画面
        composable<RegisterItem> { backStackEntry ->
            val registerItem: RegisterItem = backStackEntry.toRoute()

            RegisterItemScreen(
                onClickNavigateUp = { navController.navigateUp() },
                itemId = registerItem.id
            )
        }
```

Destination の定義は以下の通り

```kotlin
@Serializable
data class RegisterItem(
    val id: Int?
)
```

* idあり → 編集
* idなし（null） → 新規作成

最終的に、この実装方法が最適だという結果になるのですが、この時の私は、もっと良い設計が可能だと考えていました。

なぜなら、「 null が新規作成を意味する」状態だからです。

このコードは、属人化につながります。

私は null を型で排除しようと試みました。

---

## 「新規作成 or 編集」を型で表現する

私は、型で意味を表現しようとして、以下のように実装しました。

```kotlin
@Serializable
data class RegisterItem(
    val mode: ItemEditMode
)

sealed interface ItemEditMode {
    data object Add : ItemEditMode
    data class Edit(val id: Int) : ItemEditMode
}
```

これにより NavHost 内が

```kotlin
                onClickAdd = {
                    navController.navigate(RegisterItem(Add))
                },
                onClickItem = { itemId ->
                    navController.navigate(RegisterItem(Edit(itemId)))
                }
```

となり、コードの意図が明確になり、 null が排除できると考えました。

しかし、この実装ではエラーになります。

---

## なぜエラーになるのか

理由はシンプルで、

**Navigationは「完全にシリアライズ可能なデータ」しか扱えないためです。**

* sealed interface はそのままだと扱いが難しい
* data object を含むとさらに不安定
* polymorphic serialization が必要になる

つまり、 **Navigationにドメイン的な型をそのまま渡すのは難しい**

Android Developers の公式ドキュメントでも、画面遷移時のパラメータ渡しは、基本的にはプリミティブを渡すように書かれていた気がします。

「苦肉の策として、シリアライズ化すれば、クラスも渡せるよ」程度の書き方がされたいた印象があります。

---

## 正しい設計：Navigation はプリミティブ型を渡す

Navigationでは、プリミティブな値だけを扱います。

結局のところ、最適解は [すぐに思いつく実装](#すぐに思いつく実装) セクションで紹介した方法になります。

## 重要なポイント

### Navigationの責務

Navigation の責務は「画面遷移」であり、「画面のモード」まで知っているのは、責務を負いすぎている。と私は感じました。

画面が、今は「新規登録モード」なのか「編集モード」なのかは、ドメインに近い情報であると考えます。

そのため、「プリミティブ型」しか扱わないような Navigation に、モードまで扱わせるのは **荷が重い** と感じました。

### UI の責務

一方で、 UI 側は、プリミティブ型のデータを Navigation 経由で受け取り、その値に基づいて、画面側でモードの判定を行うことが可能です。

よく考えれば、画面のモードを判定する責務が、その画面自体に存在していることはかなり自然な状態だと思います。

Navigation に「基本的にプリミティブ型のみ扱うべき」という制約がなければ、 Navigation 側でモードを判定してもおかしくはないとは思いますが。

