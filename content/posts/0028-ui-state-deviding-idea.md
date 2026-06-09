+++
title = "UI 状態の分割アイデア"
date = "2026-06-09T11:00:00+09:00"
categories = ["Architecture"]
tags = ["State"]
draft = false
+++

- [前置き](#前置き)
- [結論](#結論)
- [実装例](#実装例)
  - [UI 状態の定義](#ui-状態の定義)
  - [UI 状態の内部に保持する各データクラスの定義](#ui-状態の内部に保持する各データクラスの定義)
  - [各 StateFlow の定義](#各-stateflow-の定義)
  - [combine 関数で UI 状態を生成する](#combine-関数で-ui-状態を生成する)
- [ポイント](#ポイント)
  - [4 つの UI 状態に制限したこと](#4-つの-ui-状態に制限したこと)
  - [どんなプロパティでも UI 状態に定義した 4 つのプロパティにだいたい収まる](#どんなプロパティでも-ui-状態に定義した-4-つのプロパティにだいたい収まる)
- [まとめ](#まとめ)

---

## 前置き

Jetpack Compose では、一つの画面で、 UI 状態を一つだけ定義しておくことが一般的かと思います。

画面が複雑になっていき、 UI のプロパティが増えていくと、 combine 関数で Flow を合成できなくなることがあります。

これは、 combine 関数の引数が最大でも 5 であるため、それよりたくさんの Flow を直接 combine することができないためです。

そのような場合の一つの対応案のここに残します。


## 結論

UI 状態のプロパティを以下の 4 つのプロパティにまとめます (制限します) 。

- UI ローカル状態
- フォーム入力状態
- 派生 UI 状態
- 非同期取得データ

これらをすべて StateFlow で定義し、データの変更を監視します。

データの変更が検出されるたびに、 combine 関数で StateFlow 内の実データを UI 状態のプロパティにセットします。

## 実装例

実際に私が実装していたコードが以下になります。

後から思い出せるように「ポイントだけの抜粋」になっているため、要点の把握程度の参考にしてください。

### UI 状態の定義

```kotlin
/**
 * UI で必要とされる全ての状態.
 *
 * 各上流 Flow のデータを加工した結果を状態として保持したい場合はここに定義する。
 */
data class DestinationEditUiState(
//    val transferDate: String = "",
//    val transferAmount: String = "",
    val uiLocalState: UiLocalState = UiLocalState(),
    val formInputState: FormInputState = FormInputState(),
    val derivedUiState: DerivedUiState = DerivedUiState(),
    val persistedDataState: PersistedDataState = PersistedDataState(
        sourceUiModels = emptyList(),
    ),
)
```

### UI 状態の内部に保持する各データクラスの定義

```kotlin
/**
 * 永続化の対象外の UI 状態.
 *
 * 主に UI の見た目にのみ関わる状態を管理する。
 */
data class UiLocalState(
    val destErrorMessage: Int? = null,
    val sourceErrorMessage: Int? = null,
    val isLoading: Boolean = false,
    val dialog: DestinationEditDialog? = null,
)

/**
 * ユーザーが入力し、永続化の対象となる UI 状態.
 */
data class FormInputState(
    val sourceId: Int = 0,
    val inputType: DestInputType = DestInputType.Keyboard,
    val keyboardDestId: Int = 0,
    val keyboardDestName: String = "",
    val dialogDestId: Int = 0,
)

/**
 * 画面表示専用の派生状態.
 *
 * Read-Only の状態だけを扱う。
 */
data class DerivedUiState(
    val sourceName: String = "",
    val dialogDestName: String = "",
)

/**
 * データレイヤーから取得したデータ.
 */
data class PersistedDataState(
    val sourceUiModels: List<SourceSelectionUiModel>
)
```

### 各 StateFlow の定義

```kotlin

    private val _uiLocalState = MutableStateFlow(UiLocalState())

    private val _formInputState = MutableStateFlow(FormInputState())

    private val derivedUiState: StateFlow<DerivedUiState> = combine(
        sourceLabelsById,
        _formInputState
    ) { sourceLabelsById, formInputState ->
        val dialogDestLabel = when (formInputState.inputType) {
            DestInputType.SourceList -> sourceLabelsById[formInputState.dialogDestId]
            else -> null
        }

        DerivedUiState(
            sourceName = sourceLabelsById[formInputState.sourceId] ?: "",
            dialogDestName = dialogDestLabel ?: "",
        )
    }.stateIn(
        scope = viewModelScope,
        started = WhileUiSubscribed,
        initialValue = DerivedUiState(sourceName = "")
    )

    private val persistedAsync: StateFlow<Async<PersistedDataState>> =
        sourcesQueryUseCase.loadSources()
            .map { sources ->
                PersistedDataState(
                    sourceUiModels = sources.toSourceSelectionUiModel(),
                )
            }
            .map<PersistedDataState, Async<PersistedDataState>> { state ->
                Async.Success(state)
            }
            .catch {
                Log.e(TAG, "Failed to read trans sources.", it)
                emit(Async.Error(R.string.load_error))
            }
            .stateIn(
                scope = viewModelScope,
                started = WhileUiSubscribed,
                initialValue = Async.Loading
            )
```

### combine 関数で UI 状態を生成する

```kotlin
    val uiState: StateFlow<DestinationEditUiState> =
        combine(
            _uiLocalState,
            _formInputState,
            derivedUiState,
            persistedAsync,
        ) { uiLocalState, formUiState, derivedUiState, persistedAsync ->
            when (persistedAsync) {
                is Async.Loading -> {
                    DestinationEditUiState(uiLocalState = UiLocalState(isLoading = true))
                }

                is Async.Error -> {
                    _eventChannel.send(UiEvent.ShowSnackbar(persistedAsync.errorMessage))
                    DestinationEditUiState()
                }

                is Async.Success -> {
                    DestinationEditUiState(
                        uiLocalState = uiLocalState.copy(isLoading = false),
                        formInputState = formUiState,
                        derivedUiState = derivedUiState,
                        persistedDataState = persistedAsync.data
                    )
                }
            }
        }.stateIn(
            scope = viewModelScope,
            started = WhileUiSubscribed,
            initialValue = DestinationEditUiState(uiLocalState = UiLocalState(isLoading = true))
        )
```

## ポイント

### 4 つの UI 状態に制限したこと

4 つの UI 状態に制限したことで、 combine 関数の引数上限 (5) に収まるようになっています。

プロパティが増えても、そのプロパティは UI 状態に直接追加されることはなく、 combine 関数の引数からあふれることがありません。


### どんなプロパティでも UI 状態に定義した 4 つのプロパティにだいたい収まる

UI 状態に定義した、以下 4 つプロパティの分け方をすると、多くのプロパティは、だいたいどれかに収まる。

- UI ローカル状態
- フォーム入力状態
- 派生 UI 状態
- 非同期取得データ

4 つのプロパティがそれぞれどのような意味なのか解説すると、

**UI ローカル状態** は、主に UI の見た目にのみ関わる状態を管理します。例えば、長文テキストの折畳み状態などがあります。 DB やサーバーなどの不揮発領域に保存することがないのが特徴です。

**フォーム入力状態** は、その名の通り、ユーザーが入力フォームに入力したデータを管理します。このデータは、これから不揮発領域に保存される可能性があるが、まだ保存されていないことが特徴です。

**派生 UI 状態** は、他のプロパティをそのまま使うのではなく、何らかのデータの加工をしてから使用する状態を管理します。ユーザーが直接変更できるデータは、フォーム入力状態に分類されるため、派生 UI 状態には、ユーザーが直接変更できるデータは含まれません。

**非同期取得データ** は、その名の通り、不揮発領域 (データレイヤー) から取得したデータを管理します。

## まとめ

現在作成中のアプリでは、最終的にはこの実装方針は採用しませんでしたが、今後のアプリでは採用の可能性もある気がするので、ここにメモとして残します。

設計では、 YAGNI (You aren't gonna need it) という原則もある通り、過剰に一般化することは、逆に複雑さを増加させることが多いです。

この設計は、どちらかと言えば過剰寄りだと思います。しかし、特段複雑な画面など、無法地帯になるような場面では、力を発揮するのではないかとも感じます。



