---
title: "Unityの最強シーン遷移管理フレームワーク『Navigathena』と画面管理の設計思想について解説"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unity", "設計", "シーン管理", "ゲーム開発"]
published: true
published_at: 2023-11-02 12:00
---

## はじめに

Unityにおいて、シーンの遷移・管理は開発の中核的な役割を担います。それ故に、適切な設計なしで開発に踏み込むと、複雑で扱いづらいプロジェクトになるリスクが高まります。しかし、効果的なシーン管理の設計と実装をするには、それなりに時間と知識を要します。

この課題を解決するため、「高機能」「拡張性」「堅牢性」を念頭に設計・実装した画面管理フレームワークの **『Navigathena』** をオープンソースライブラリとして公開しました。

[[GitHub - mackysoft/Navigathena]](https://github.com/mackysoft/Navigathena)

本記事では、Navigathenaの特徴や使い方を紹介しつつ、Unityにおけるシーン管理の設計について深掘っていきます。

## シーン管理には何が必要か？

実際に、Unityでの開発を行う際、どのような機能や考慮点が必要となるでしょうか？
これまでの経験や開発事例を踏まえると、大まかに以下のような機能が必要になります。

- 基本的な遷移操作
- 遷移履歴管理
- シーン間データ受け渡し
- 柔軟なシーン遷移演出
- 単一シーン起動
- 割り込みシーン操作

今回開発したフレームワークでは、これらの要素をうまく組み合わせて、効果的なシーン管理を実装しています。

そして何より、**「シーン遷移周りの設計が複雑化しやすい問題の解決」** が必要です。前述の通り、シーン管理は開発の中核を担うものであり、適切な設計がなされないと、メンテナンス性が損なわれる可能性が高まります。例えば、「シーン遷移時のロジックが複数箇所に散らばっている」「シーンロジックの管理者が明確でない」などの問題が考えられます。

これに対しては、全体を通してそういった設計上の問題を防止しやすい作りになっています。理念的なものも含まれるため、設計に関しては記事の後半部分で深掘っていきます。

## Navigathenaの基本概念

Navigathenaには、シーン遷移のための2つの基本概念、 **「SceneNavigator」** と  **「SceneEntryPoint」** があります。

### `SceneNavigator`

Navigathenaでは、シーンの遷移と管理を担当する`ISceneNavigator`インターフェースを提供しています。、多くのご家庭で親しまれているSceneManagerのような機能を持っており、シーンの履歴を取り扱いつつ基本的な遷移操作を提供します。

以下は`ISceneNavigator`の使用例です。

```cs
ISceneIdentifier identifier = new BuiltInSceneIdentifier("Game");

// シーンをロードして、履歴に追加する
await GlobalSceneNavigator.Instance.Push(identifier);

// 履歴の先頭のシーンを削除して、一つ前のシーンをロードする
await GlobalSceneNavigator.Instance.Pop();
```

シーンの指定には`ISceneIdentifier`インターフェースを使用します。デフォルトで`BuiltInSceneIdentifier`が実装されており、UnityのBuild Settingsに登録されているシーンをロードする際に使用することができます。

:::message
Addressablesを利用する例については、後述の[『Addressables Systemとの統合』](#addressables-systemとの統合)セクションで解説を行います。
:::

`ISceneNavigator`の主要な遷移操作関数は以下の通りです。

```cs
interface ISceneNavigator
{
  // 新たなシーンをロードして、履歴に追加します
  UniTask Push (ISceneIdentifier identifier, ITransitionDirector transitionDirector = null, ISceneData sceneData = null, IAsyncOperation interruptOperation = null, CancellationToken cancellationToken = default);

  // 履歴の先頭のシーンを削除して、一つ前のシーンをロードします
  UniTask Pop (ITransitionDirector overrideTransitionDirector = null, IAsyncOperation interruptOperation = null, CancellationToken cancellationToken = default);

  // 新たなシーンをロードして、履歴をそのシーンのみに上書きします
  UniTask Change (ISceneIdentifier identifier, ITransitionDirector transitionDirector = null, ISceneData sceneData = null, IAsyncOperation interruptOperation = null, CancellationToken cancellationToken = default);

  // 新たなシーンをロードして、履歴の先頭を上書きします
  UniTask Replace (ISceneIdentifier identifier, ITransitionDirector transitionDirector = null, ISceneData sceneData = null, IAsyncOperation interruptOperation = null, CancellationToken cancellationToken = default);

  // 現在のシーンをリロードします
  UniTask Reload (ITransitionDirector overrideTransitionDirector = null, IAsyncOperation interruptOperation = null, CancellationToken cancellationToken = default);
}
```

:::message
上記の関数群は正確に言うと、`ISceneNavigator`の拡張関数として実装されています。
:::

遷移ロジックは`ISceneNavigator`インターフェースで抽象化されているため、特定のプロジェクトで特別な処理が必要な場合、カスタムSceneNavigatorを実装することができます。（指定が無い場合は、`StandardSceneNavigator`が使用されます）

```cs:NavigathenaInitializer.cs
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
static void Initialize ()
{
  // カスタムSceneNavigatorを登録する
  GlobalSceneNavigator.Instance.Register(new MyCustomSceneNavigator());
}
```

ゲームのライフサイクルを通じて、シーン管理は同じロジックを使用することがほとんどであるため、シングルトンなコンポーネントである`GlobalSceneNavigator`を実装してあります。これは登録された`ISceneNavigator`をラップし、後述の「単一シーン起動」などの機能をサポートをしています。

ちなみに`GlobalSceneNavigator`のインスペクターでは、登録された`ISceneNavigator`と、現在の履歴を確認できます。

![](https://storage.googleapis.com/zenn-user-upload/af995c2c903c-20231023.png)

### `SceneEntryPoint`

次に、シーンのライフサイクルを監視するための`SceneEntryPoint`という概念が導入されています。これは各シーンに一つだけ配置可能なコンポーネントで、シーンの開始、終了、データの受け渡し等のイベントを監視する役割を果たします。`SceneEntryPoint`がシーンに単一で存在しつつ、シーンのロジックを統括することで、シーンロジックの複雑化を防止することに繋がります。

基本的に、`SceneEntryPoint`は、`SceneEntryPointBase`コンポーネントを継承して、各イベントをオーバーライドして使用します。

```cs:TitleSceneEntryPoint.cs
using System.Threading;
using Cysharp.Threading.Tasks;
using MackySoft.Navigathena.SceneManagement;

// タイトルシーンのSceneEntryPointを実装するコンポーネント
public sealed class TitleSceneEntryPoint : SceneEntryPointBase
{
  protected override async UniTask OnEnter (ISceneDataReader reader, CancellationToken cancellationToken)
  {
    await DoSomething(cancellationToken);
  }
}
```

:::message
`MonoBehaviour`の継承に依存しない実装方法については、[「依存性注入との統合」](#依存性注入との統合)セクションを参照してください。
:::

SceneEntryPointコンポーネントを定義したら、シーンに1つ配置します。

![](https://storage.googleapis.com/zenn-user-upload/6e9ebbecfc2f-20231031.png)

あとはこのシーンを実行すれば、`TitleSceneEntryPoint`の起動時イベントが呼び出されます。

以下は`SceneEntryPoint`のイベントの一覧です。

```cs:ISceneEntryPoint.cs
public interface ISceneEntryPoint
{
  // 遷移演出の開始後に呼び出されます。
  UniTask OnInitialize (ISceneDataReader reader, IProgress<IProgressDataStore> transitionProgress, CancellationToken cancellationToken);

  // 遷移演出の終了後に呼び出されます。
  UniTask OnEnter (ISceneDataReader reader, CancellationToken cancellationToken);

  // 遷移演出の開始前に呼び出されます。
  UniTask OnExit (ISceneDataWriter writer, CancellationToken cancellationToken);

  // 遷移演出の開始後に呼び出されます。
  UniTask OnFinalize (ISceneDataWriter writer, IProgress<IProgressDataStore> transitionProgress, CancellationToken cancellationToken);

#if UNITY_EDITOR
  // エディタ内での実行時、一番最初にロードされたシーンで`OnInitialize`の前に呼び出される。 (エディタ専用)
  UniTask OnEditorFirstPreInitialize (ISceneDataWriter writer, CancellationToken cancellationToken);
#endif
}
```

遷移操作が呼ばれたときのシーン遷移の流れは以下のようになります。

1. 遷移元`ISceneEntryPoint`の`OnExit`
2. 遷移操作時に渡された`ITransitionDirector`から生成された`ITransitionHandle`の`Start`
3. 遷移元`ISceneEntryPoint`の`OnFinalize`
4. 遷移元シーンのアンロード
5. 遷移操作時に渡された`IAsyncOperation`の`ExecuteAsync`
6. 遷移先シーンのロード
7. 遷移先`ISceneEntryPoint`の`OnInitialize`
8. 2で使用された`ITransitionHandle`の`End`
9. 遷移先`ISceneEntryPoint`の`OnEnter`

各イベントが果たす役割については、後述のセクションで詳しく解説します。

### 概要図

Navigathenaのシーン管理の主要な要素は以下の図のようにまとまっています。

![](https://storage.googleapis.com/zenn-user-upload/c8257eec1c55-20231023.png)

多くの場合、`SceneEntryPointBase`を継承して各シーンの挙動を実装し、`GlobalSceneNavigator`を介して遷移操作を行うだけで十分です。必要に応じて`ISceneNavigator`を拡張してカスタムロジックを実装し、`GlobalSceneNavigator`に登録してアプリケーションに適用しましょう。

## シーン遷移演出

ゲームの体験を彩る要素の一つとして、シーン遷移演出があります。それはシンプルなフェードイン・フェードアウトであったり、Tipsの表示、ミニキャラが画面を駆け抜けるアニメーションまで多岐にわたる可能性があります。

どの様に実装を行うかにしても、Canvas要素をアニメーションさせるかもしれないし、追加で遷移演出シーンをロードするかもしれないです。いずれにせよ、システムが柔軟であること、すなわち異なるタイプの演出を簡単に追加・変更できることが求められます。

Navigathenaでのシーン遷移時の演出は`ITransitionDirector`インターフェースによって定義されており、シーン遷移操作時の引数には`ITransitionDirector`を渡すことができます。

```cs
public interface ITransitionDirector
{
  ITransitionHandle CreateHandle ();
}

public interface ITransitionHandle
{
  UniTask Start (CancellationToken cancellationToken = default);
  UniTask End (CancellationToken cancellationToken = default);
}
```

`ITransitionDirector`は`ITransitionHandle`を生成するためのファクトリになっており、`ITransitionHandle`は遷移演出の開始と終了を制御するためのインターフェースです。

この`ITransitionDirector`および`ITransitionHandle`を基に拡張を行うことで、独自の遷移演出を制御できます。

以下は、シンプルなフェードによる遷移演出を実現する`SimpleTransitionDirector`の実装例です。

```cs:SimpleTransitionDirector.cs
public sealed class SimpleTransitionDirector : ITransitionDirector
{

  readonly CanvasGroup m_CanvasGroup;

  public SimpleTransitionDirector (CanvasGroup canvasGroup)
  {
    m_CanvasGroup = canvasGroup;
  }

  public ITransitionHandle Create ()
  {
    return new SimpleTransitionHandle(m_CanvasGroup);
  }

  sealed class SimpleTransitionHandle : ITransitionHandle
  {

    readonly CanvasGroup m_CanvasGroup;

    public SimpleTransitionHandle (CanvasGroup canvasGroup)
    {
      m_CanvasGroup = canvasGroup;
    }

    public async UniTask Start (CancellationToken cancellationToken = default)
    {
      // DOTweenでフェードインを実行
      await m_CanvasGroup.DOFade(1f, 1f).ToUniTask(cancellationToken: cancellationToken);
    }

    public async UniTask End (CancellationToken cancellationToken = default)
    {
      await m_CanvasGroup.DOFade(0f, 1f).ToUniTask(cancellationToken: cancellationToken);
    }
  }
}
```

```cs
// SimpleTransitionDirectorで演出を実行しつつ、新たなシーンをロードする
await GlobalSceneNavigator.Instance.Push(new BuiltInSceneIdentifier("MyScene"), new SimpleTransitionDirector(m_CanvasGroup));
```

### 遷移中のプログレス表示

遷移演出では、進捗率の表示や、進捗率によって変動する演出が必要になることも少なくないでしょう。

Navigathenaでは、遷移演出中には`IProgress<IProgressDataStore>`をを介し、任意のデータ型を渡すことが可能です。遷移演出中に挟まれる処理（`ISceneEntryPoint.OnInitialize`/`OnFinalize`、`IAsyncOperation.ExecuteAsync`）には、`IProgress<IProgressDataStore>`が渡されます。各イベント内で`IProgressDataStore`にデータを書き込み、`IProgress<IProgressDataStore>`に通知することで、遷移の進捗状況やメッセージなど、プレイヤーに提示したい情報を遷移演出に組み込むことができます。

以下は、先ほどの`SimpleTransitionDirector`を拡張し、進捗表示に対応させる例です。

```cs:MyProgressData.cs
// プログレス情報として使用したい任意のデータ型を定義する
public readonly struct MyProgressData
{
  
  public float Progress { get; }
  public string Message { get; }

  public MyProgressData (float progress, string message)
  {
    Progress = progress;
    Message = message;
  }
}
```

```cs:SimpleTransitionDirector.cs
public sealed class SimpleTransitionDirector : ITransitionDirector
{

  readonly CanvasGroup m_CanvasGroup;
  readonly Text m_ProgressText;
  readonly Text m_MessageText;
  readonly Slider m_ProgressSlider;

  public SimpleTransitionDirector (CanvasGroup canvasGroup, Text progressText, Text messageText, Slider progressSlider)
  {
    m_CanvasGroup = canvasGroup;
    m_ProgressText = progressText;
    m_MessageText = messageText;
    m_ProgressSlider = progressSlider;
  }

  public ITransitionHandle Create ()
  {
    return new SimpleTransitionHandle(m_CanvasGroup, m_ProgressText, m_MessageText, m_ProgressSlider);
  }

  // IProgress<IProgressDataStore>を実装することで、遷移演出中にプログレス情報を受け取ることができる
  sealed class SimpleTransitionHandle : ITransitionHandle, IProgress<IProgressDataStore>
  {

    readonly CanvasGroup m_CanvasGroup;
    readonly Text m_ProgressText;
    readonly Text m_MessageText;
    readonly Slider m_ProgressSlider;

    public SimpleTransitionHandle (CanvasGroup canvasGroup, Text progressText, Text messageText, SLider progressSlider)
    {
      m_CanvasGroup = canvasGroup;
      m_ProgressText = progressText;
      m_MessageText = messageText;
      m_ProgressSlider = progressSlider;
    }

    public async UniTask Start (CancellationToken cancellationToken = default)
    {
      await m_CanvasGroup.DOFade(1f,1f).ToUniTask(cancellationToken: cancellationToken);
    }

    public async UniTask Complete (CancellationToken cancellationToken = default)
    {
      await m_CanvasGroup.DOFade(0f,1f).ToUniTask(cancellationToken: cancellationToken);
    }

    void IProgress<IProgressDataStore>.Report (IProgressDataStore progressDataStore)
    {
      // IProgressStoreからMyProgressDataを取り出す
      if (progressDataStore.TryGetData(out MyProgressData myProgressData))
      {
        m_ProgressText.text = myProgressData.Progress.ToString("P0");
        m_MessageText.text = myProgressData.Message;
        m_ProgressSlider.value = myProgressData.Progress;
      }
    }
  }
}
```

```cs
// 遷移操作時にSimpleTransitionDirectorを渡す
await GlobalSceneNavigator.Instance.Push(new BuiltInSceneIdentifier("MyScene"), new SimpleTransitionDirector(m_CanvasGroup, m_ProgressText, m_MessageText, m_ProgressSlider));
```

```cs:MySceneEntryPoint.cs
// ...

// progressに通知を行うと、SimpleTransitionDirectorまで通知される
protected override UniTask OnInitialize (ISceneDataReader reader, IProgress<IProgressDataStore> progress, CancellationToken cancellationToken)
{
  ProgressDataStore<MyProgressData> store = new();

  progress.Report(store.SetData(new MyProgressData(0.5f, "Generate Map")));

  await m_MapGenerator.Generate(cancellationToken);

  progress.Report(store.SetData(new MyProgressData(1f, "Complete")));
}
```

汎用性を確保するため、`IProgressDataStore.TryGetData<T>`を介してデータを取得できるようにしています。（内部的には`IProgressDataStore`を`IProgressDataStore<T>`にキャストして型を取り出しています）
型安全ではないのですが、ボックス化による無駄な割り当ての防止と、ジェネリクスの伝搬によって簡易性が失われるのを避けるためにこのような実装が妥当と判断しました。

シーンがロード・アンロードされる時、`StandardSceneNavigator`のデフォルトでは`IProgressDataStore`に`LoadSceneProgressData`を格納するようになっています。ここの挙動は`StandardSceneNavigator`のコンストラクタからカスタマイズすることも可能です。

```cs:NavigathenaInitializer.cs
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
static void Initialize ()
{
  // 自前のMySceneProgressFactoryでStandardSceneNavigatorを初期化して登録する
  GlobalSceneNavigator.Instance.Register(new StandardSceneNavigator(TransitionDirector.Empty(), new MySceneProgressFactory()));
}
```

### シーン遷移中処理の組込み

シーン遷移時には事前ロード処理や、何かしらの事前チェック処理を挟みこみたい場合があります。

Navigathenaでは「現在のシーンのアンロードが終わってから、次のシーンをロードするまでの間に挟みたい処理」を実行するための、`IAsyncOperation`インターフェースが定義されています。`IAsyncOperation`は`ISceneNavigator`における各遷移操作時に渡すことができます。

```cs
IAsyncOperation op = m_PreloadAsyncOperation;
await GlobalSceneNavigator.Push(nextScene, interruptOperation: op);
```

`IAsyncOperation`の構成は非常に単純で、非同期処理を実行する関数が定義されています。`IProgress<IProgressDataStore>`が引数として渡されるため、遷移演出中のプログレス表示と統合する事も可能です。

```cs:IAsyncOperation.cs
public interface IAsyncOperation
{
  UniTask ExecuteAsync (IProgress<IProgressDataStore> progress, CancellationToken cancellationToken = default);
}
```

基本的には`IAsyncOperation`を実装した型を定義する事になるかと思いますが、便利に扱うための便利関数もいくつか用意されています。

```cs
// 匿名のIAsyncOperationを作成する
IAsyncOperation operation = AsyncOperation.Create(async (progress, cancellationToken) => {
  // 何らかの非同期処理
  await DoSomething(progress, cancellationToken);
});

// 複数のIAsyncOperationをマージ
IAsyncOperation compositeOperation = AsyncOperation.Combine(op1, op2, op3);
```

## シーン間のデータ受け渡し

ゲームを作っていると、次のシーンに値を渡したいシチュエーションに遭遇します。よくあるのが、「選択した要素（ステージ、キャラクターなど）のIDを次のシーンに渡したい」といったケースです。

`ISceneEntryPoint`のコールバックでは、

- `OnInitialize` / `OnEnter` に `ISceneDataReader`
- `OnExit` / `OnFinalize` に `ISceneDataWriter` 

が渡され、これらのインターフェースによってシーン間のデータのやり取りを行うことができます。

例えば、`ISceneNavigator.Push`の引数に、`ISceneData`インターフェースを実装したデータ型を渡すことで、遷移先の`ISceneEntryPoint`の`OnInitialize`と`OnEnter`に対して、そのデータを渡すことができます。

```cs
// シーン間で渡したいデータを定義する
public sealed class IngameSceneData : ISceneData
{
  public int CharacterId { get; init; }
}

// ...

// sceneDataに、渡したいデータを格納する
await GlobalSceneNavigator.Instance.Push(new BuiltInSceneIdentifier("Ingame"), sceneData: new IngameSceneData
{
  CharacterId = 1
});
```

```cs:IngameSceneEntryPoint.cs
protected override UniTask OnEnter (ISceneDataReader reader, CancellationToken cancellationToken)
{
  // 遷移時のデータを取得できる
  if (reader.TryRead(out IngameSceneData sceneData))
  {
    // キャラクターを生成したりしてみる
    Spawn(sceneData.CharacterId);
  }
}
```

一方、シーンを離脱するときに渡される`ISceneDataWriter`ですが、こちらはシーン離脱時の状態をシーン履歴に保存し、そのシーンに戻ってきたときにシーンの状態を復元するような処理の実装を可能にします。

例えば、「シーン内で特定の画面を開いた状態」で次のシーンに遷移し、前のシーンに戻る時は、`ISceneDataWriter`に書き込まれたデータが`OnInitialize` / `OnEnter`に渡されるので、シーン初期化時には「どの画面を開いていたか」の情報を取得して、シーンの状態を復元することができます。

```cs:HomeSceneEntryPoint.cs
public sealed class HomeSceneData : ISceneData
{
  public HomeScreenType LastDisplayedScreenType { get; init; }
}

public sealed class HomeSceneEntryPoint : SceneEntryPoint
{

  [SerializeField]
  HomeView m_View;

  protected override UniTask OnInitialize (ISceneDataReader reader, CancellationToken cancellationToken)
  {
    // シーン起動時、保存されている状態を取得してシーンの要素に適用する
    if (reader.TryRead(out HomeSceneData sceneData))
    {
      m_View.SetScreen(sceneData.LastDisplayedScreenType);
    }
  }

  protected override UniTask OnFinalize (ISceneWriter writer, CancellationToken cancellationToken)
  {
    // シーン離脱時、状態をデータに保存する
    writer.Write(new HomeSceneData
    {
      LastDisplayedScreenType = m_View.ScreenType
    });
  }
}
```

:::message
型安全性が損なわれることを避けるため、`ISceneData`型の定義は「1シーンにつき1つ」で統一することを推奨します。
:::

## 単一シーン起動

Unityエディターでのデバッグでは、「どのシーンからでもゲームを正常に起動することができる」ことで開発効率が向上します。

特にシーンの階層構造が深かったり、機能までのシーケンスが長いと効果が顕著に現れます。Photonでマルチプレイヤーゲームを作っている時は、マッチング処理を飛ばしてインゲームのデバッグ出来るようにすることで、開発サイクルを短縮していました。

`ISceneEntryPoint`では、`OnEditorFirstPreInitialize`というエディター専用のコールバックを用意しており、「エディター上で一番最初に起動されたシーン」で呼ばれます。このコールバックで、一番最初に起動された時には`ISceneDataWriter`が渡されるため、初期データを書き込んで後続の `OnInitialize` / `OnEnter` に対してデータを渡すことができます。

```cs:IngameSceneEntryPoint.cs
protected override UniTask OnEnter (ISceneDataReader reader, CancellationToken cancellationToken)
{
  // エディターで最初に起動されたとき、OnEditorFirstPreInitializeで書き込まれたデータを取得できる
  if (reader.TryRead(out IngameSceneData sceneData))
  {
    Spawn(sceneData.CharacterId);
  }
}

#if UNITY_EDITOR
protected override UniTask OnEditorFirstPreInitialize (ISceneDataWriter writer, CancellationToken cancellationToken)
{
  // ISceneDataWriterに初期データを書き込む
  writer.Write(new IngameSceneData
  {
    CharacterId = m_EditorTestCharacterId
  });
  return UniTask.CompletedTask;
}
#endif
```

:::message
`OnEditorFirstPreInitialize`は`UNITY_EDITOR`ディレクティブで囲う必要があります。
:::

## 割り込みシーン操作

時々、シーン遷移の処理中に、様々なチェックや通信の失敗などの事象が発生し、別のシーンへの遷移処理で上書きしたくなる場合があります。

例えば、「イベントシーンへ遷移中に、`OnInitialize`でのチェックでイベントの期間が終了していたため、`Pop`を使用して前のシーンに戻りたい」のようなケースがあります。この時、現在の遷移処理を中断してシーン遷移操作を割り込ませるのですが、これを正しく実装しようとすると内部の状態管理が地味に大変です。

Navigathenaでは、このような割り込み処理にも対応できるように実装しています。
以下は、実際の動作の簡単な例です。

```cs:EventSceneEntryPoint.cs
protected override async UniTask OnInitialize (ISceneDataReader reader, IProgress<IProgressDataStore> progress, CancellationToken cancellationToken)
{
  // イベントの期限が切れていると仮定する
  bool isExpired = true;

  if (isExpired) {
    // NOTE: cancellationTokenはキャンセル状態になるため、遷移操作には渡さない
    await GlobalSceneNavigator.Instance.Pop(CancellationToken.None);

    // true
    Debug.Log(cancellationToken.IsCancellationRequested);
  }
}

protected override UniTask OnEnter (ISceneDataReader reader, CancellationToken cancellationToken)
{
  // OnInitialize内で遷移操作が呼び出されていたら実行されない
  Debug.Log("EventSceneEntryPoint.OnEnter");
  return UniTask.CompletedTask;
}
```
新たな遷移処理を割り込ませることで、SceneNavigatorが管理する`CancellationTokenSource`がキャンセルされ、それに伴って渡される`CancellationToken`もキャンセル状態となることを確認できます。上記の例では、`Pop`呼び出し後に`cancellationToken`がキャンセル要求状態となり、その結果、`OnEnter`が呼ばれなくなることが示されています。

## シーン履歴の直接操作

時折、現在の履歴を無視して新しい履歴を構築したい場面があります。例えば「任意のシーンにジャンプするが、そのシーンで`Pop`をした時は通常通りのシーンに戻ってほしい」といったケースです。

Navigathenaでは、`ISceneNavigator`の履歴を直接操作する`ISceneHistoryBuilder`を`GetHistoryBuilderUnsafe`を使用して取得できます。

```cs
using MackySoft.Navigathena.SceneManagement;
using MackySoft.Navigathena.SceneManagement.Unsafe; // IUnsafeSceneNavigatorの機能の使用に必要

//...

// 現在の履歴から現在のシーン以外を削除し、一つ前のシーンとしてHomeシーンを追加し、履歴を再構築する。
GlobalSceneNavigator.Instance.GetHistoryBuilderUnsafe()
  .RemoveAllExceptCurrent()
  .Add(new SceneHistoryEntry(SceneDefinitions.Home, TransitionDefinitions.Loading, new SceneDataStore())))
  .Build();

// Popすると、Homeシーンに戻る
await GlobalSceneNavigator.Instance.Pop();
```

取得した`ISceneHistoryBuilder`は、`ISceneNavigator`の遷移操作が行われた後はバージョン差異によって使用不可になります。なので、遷移操作を行う前に`ISceneHistoryBuilder`での履歴操作処理を完了させる必要があります。

この機能は`ISceneNavigator`には付属しておらず、`IUnsafeSceneNavigator`で定義されています。カスタム`ISceneNavigator`を実装する際にシーン履歴操作機能が必要な場合は、`IUnsafeSceneNavigator`を追加で実装してください。（`GlobalSceneNavigator`と`StandardSceneNavigator`は、明示的に`IUnsafeSceneNavigator`を実装しています）

:::message
名前にも付いている通り、`IUnsafeSceneNavigator`はUnsafeな機能なので多用は推奨していません。
:::

## Addressables Systemとの統合

Navigathenaにおいて、各シーンのロード・アンロードの低レベル処理は`ISceneIdentifier`インターフェースで扱われます。

デフォルトでは`BuiltInSceneIdentifier`のみ実装されていますが、プロジェクトに[『Addressables』](https://docs.unity3d.com/Packages/com.unity.addressables@2.0/manual/index.html)パッケージが含まれていれば`AddressalbeSceneIdentifier`が使用可能になり、Addressablesとして扱われているシーンのロード・アンロードをシームレスに組み込むことが可能になります。

以下は、実際に作っているゲームでの使用例です。

```cs:SceneDefinitions.cs
using MackySoft.Navigathena.SceneManagement;
using MackySoft.Navigathena.SceneManagement.AddressableAssets;

// プロジェクトで使用されるシーンの定義
public static class SceneDefinitions
{

  // スプラッシュではBuiltInSceneIdentifierを使用する
  public static ISceneIdentifier Splash { get; } = new BuiltInSceneIdentifier("Splash");

  // スプラッシュ以降のシーンでは、Addressablesの依存性を解決する必要があるため、AddressableSceneIdentifierを使用する
  // 引数には`object key`を渡す
  public static ISceneIdentifier Title { get; } = new AddressableSceneIdentifier("Title");
  public static ISceneIdentifier Introduction { get; } = new AddressableSceneIdentifier("Introduction");
  public static ISceneIdentifier Home { get; } = new AddressableSceneIdentifier("Home");
  public static ISceneIdentifier Game { get; } = new AddressableSceneIdentifier("Game");
}
```

`ISceneIdentifier`インターフェースでカプセル化されているため、使用するシーンがビルトインかAddressableかどうか気にする必要はありません。
`ISceneIdentifier`の中身は単純で、`CreateHandle`で`ISceneHandle`を返し、返された`ISceneHandle`が実際のロード・アンロードやリソースの処理を担っています。これらを基に拡張を行えば、独自のリソース管理方式にも対応可能です。

```cs
public interface ISceneIdentifier
{
  ISceneHandle CreateHandle ();
}

public interface ISceneHandle
{
  UniTask<Scene> Load (IProgress<float> progress = null, CancellationToken cancellationToken = default);
  UniTask Unload (IProgress<float> progress = null, CancellationToken cancellationToken = default);
}
```

`AddressableSceneIdentifier`の場合は、Addressablesの`LoadSceneAsync`/`UnloadSceneAsync`をラップしているので、もちろんアセットの依存関係も解決してくれます。

## 依存性注入との統合

シーン管理基盤は「誰がその画面のロジックを統括するのか」という、ゲーム開発の中でも設計の根幹を左右する中枢になりやすいです。そのため、シーン管理基盤が依存性の注入（DI）を統合しておくことで、プロジェクトのアーキテクチャ・設計において選択肢を増やすことができます。

Navigathenaでは、SceneEntryPoint の上にDIコンテナとの機能統合を組み込むことで、依存性注入に対応することができます。（デフォルトでは[『VContainer』](https://github.com/hadashiA/VContainer)をサポートしています）

この統合では、`SceneEntryPointBase`（`ISceneEntryPoint`）を継承する代わりに、`SceneLifecycleBase`(`ISceneLifecycle`)を継承した型をDIコンテナに登録し、Navigathenaで定義済みの`ScopedSceneEntryPoint`をシーンに配置することで、シーンのライフサイクルをコンポーネント（`MonoBehaviour`）から切り離すことができます。

```cs:TitleLifetimeScope.cs
public override void Configure (IContainerBuilder builder)
{
  builder.RegisterSceneLifecycle<TitleSceneLifecycle>();
}
```

:::message
VContainerのコンテナが親を決めるタイミングの仕様上、少しハッキーですが`LifetimeScope`のGameObjectは非アクティブにしておく必要があります。`ScopedSceneEntryPoint`のインスペクターに表示されている`Create LifetimeScope`ボタンから簡単に生成・設定が出来ます。
:::

これにより、依存性注入による諸々のメリットを享受することが可能ですが、この統合で特に嬉しいのは、コンポーネントから切り離すことで得られる [**『コンストラクタインジェクション』**](https://vcontainer.hadashikick.jp/ja/resolving/constructor-injection) でしょう。

大きく分けて、以下のような利点があります。

- 依存性が解決できない場合に例外が発生するため、依存性の解決漏れを防ぐことができます。参照が足りない場合は、シーンの初期化時に例外で止まるため、開発者がいち早く気付くことができます。
- インスタンスの生成時に依存性が保証されます。Unityでありがちな「初期化順序の関係で`NullReferenceException`が発生する！」という事を気にする必要が無くなります。
- 依存性を`readonly`にすることが可能であるため、不変性を保証することが可能になります。それにより、不正な書換えによる予期せぬ不具合を防止することができます。
- 依存関係が明示的になり、そのクラスが何に依存しているかがひと目で把握可能になります。責任過多なクラスが分かりやすくなり、保守性低下の兆候に気付きやすくなります。（地獄を見なくて済む！）

このように、DIとの統合はプロジェクト品質を保つうえで有効なツールになります。

ちなみに、UnityのプロジェクトでいざDIを導入しようとしたら「DI分からない～」になりがちですが、その原因は **「どこを起点に作っていけば分からない」** に集約されると考えています。

VContainerでも、Presenterなどの起点を作るための[エントリーポイント機能](https://vcontainer.hadashikick.jp/ja/integrations/entrypoint)が存在しますが、「エントリーポイントは一つまで」というような制約はないため、**「機能をどの様に連携させ、ゲームとしてのフローを成立させていくか」** というところで悩むことになります。

その点、「Navigathena + DI」では「SceneLifecycleという単一のエントリーポイント」が分かりやすい起点となってくれます。実際、SceneLifecycleもLifetimeScope（DIコンテナ）も「起点」という役割が同じなので、自然な流れが作りやすく、相性が良かったりします。

そのため、**「DIの扱いの分かりやすさ」** において、かなり扱いやすくなっているのでぜひ！（この統合を使っていても、VContainerのエントリーポイント機能自体はガンガン使っていく事になると思いますが）

## FAQ

### 複数シーン（サブシーン）のロードはどうするか？

例えば、インゲームのシーンをロードした時、追加でステージやUIを含んだシーンをロードしたいケースを考えてみましょう。

こういう実装をするときに時折、以下のようなシーン管理ラッパーでサブシーンの管理が行われていることがあります。

```cs
// インゲームの為のシーン群をロードする
MyMultiSceneManager.LoadSceneGroup("Ingame", "HUD", $"Stage_{stageId}");
```

Navigathenaではそういったサブシーン機能は明示的にサポートしていません。
これは、 **遷移先と遷移元に相互依存性を持たせないため** です。

適切な管理としては、IngameシーンがHUDシーンに対しての依存性を持っている場合、Ingameシーン自身がHUDシーンを管理する必要があります。もし、この管理を遷移元側で行ってしまうと、Ingameシーンは暗黙的に遷移元シーンに依存することになり、遷移元と遷移先は相互依存性を持つことになります。そして、**相互依存性を持つと「誰がシーンを管理しているか」という責任が曖昧になり、変更困難なコードを生み出すリスクが高まります。**

このようなケースでは、

**「『シーン間のデータの受け渡し』でステージIDなどのデータをやり取りし、遷移先シーンの`SceneEntryPoint`（および`SceneLifecycle`）のライフサイクル内で`UnityEngine.SceneManagement`（またはそれに類するラッパー）を使って、サブシーンを管理する」**

ことで責任を明確化することを推奨しています。以下はIngameシーンの実装例です。

```cs:IngameSceneEntryPoint.cs
long m_StageId;

protected override async UniTask OnInitialize (ISceneDataReader reader, IProgress<IProgressDataStore> progress, CancellationToken cancellationToken)
{
  // Ingameシーンで必要なサブシーンをロードする
  await SceneManager.LoadSceneAsync("HUD", LoadSceneMode.Additive).ToUniTask(cancellationToken: cancellationToken);

  // Ingameシーンに渡すために定義したデータ型（IngameSceneData）でデータを受け取る
  var data = reader.Read<IngameSceneData>();
  m_StageId = data.StageId;
  await SceneManager.LoadSceneAsync($"Stage_{data.StageId}", LoadSceneMode.Additive).ToUniTask(cancellationToken: cancellationToken);
}

protected override async UniTask OnFinalize (ISceneDataWriter writer, IProgress<IProgressDataStore> progress, CancellationToken cancellationToken)
{
  // Ingameシーンで管理しているサブシーンを破棄する
  await SceneManager.UnloadSceneAsync("HUD").ToUniTask(cancellationToken: cancellationToken);
  await SceneManager.UnloadSceneAsync($"Stage_{m_StageId}").ToUniTask(cancellationToken: cancellationToken);
}
```

```cs
await GlobalSceneNavigator.Instance.Push(SceneDefinitions.Ingame, sceneData: new IngameSceneData
{
  StageId = 1
});
```

遷移元は「遷移先シーンの構築に必要なデータを渡す」に責務を絞り、遷移先はサブシーンの管理責任を持つことで、「シーンがどのサブシーンを持ち、どのようにそれをロード・アンロードするか」のロジックをシーン内で完結させることができます。

遷移操作は原則として、**「遷移元は、遷移先が提供しているもの以上を使用しない」** という運用が望ましいです。

「遷移先が提供しているもの」とは、「シーンの識別子（シーン名など）」や「シーンに渡すために事前定義されたデータ型」の事です。例えば、Ingameシーンは「`Ingame`という識別子」と「`IngameSceneData`というデータ受け渡し用のデータ型」を提供することでIngameに対する遷移操作を提供しています。

この原則を守ることで、 **「シーンを正常に動かすために必要なロジックが様々な場所に分散していて、シーンのロジックを把握できない！」→「変更を加えたら、変なところで不具合が出る！」** といった問題を防止することができます。

### アプリケーションの寿命を通して、常に存在するシーンを持ちたい

これは個人的なニーズでもあるのですが、アプリケーションの寿命内で常に存在し続けるシーン（`Root`シーンと呼んでいます）があると開発上便利です。寿命が長い機能の一覧性が高まりますし、サードパーティアセットの都合でシーン上に保持しておく必要がある機能もあります。`DontDestroyOnLoad`という手もありますが、取り回しが効きづらいのが難点です。

開発中のプロジェクトでは`ScopedSceneEntryPoint`を拡張して、`EnsureParentScope`のタイミングで`Root`シーンをロードするという手法を取っています。

```cs
public sealed class MyProjectScopedSceneEntryPoint : ScopedSceneEntryPoint
{

  const string kRootSceneName = "Root";

  protected override async UniTask<LifetimeScope> EnsureParentScope (CancellationToken cancellationToken)
  {
    // Load root scene.
    if (!SceneManager.GetSceneByName(kRootSceneName).isLoaded)
    {
      await SceneManager.LoadSceneAsync(kRootSceneName, LoadSceneMode.Additive)
        .ToUniTask(cancellationToken: cancellationToken);
    }

    Scene rootScene = SceneManager.GetSceneByName(kRootSceneName);

#if UNITY_EDITOR
    // Reorder root scene.
    EditorSceneManager.MoveSceneBefore(rootScene, gameObject.scene);
#endif

    // Build root LifetimeScope container.
    if (rootScene.TryGetComponentInScene(out LifetimeScope rootLifetimeScope, true) && rootLifetimeScope.Container == null)
    {
      await UniTask.RunOnThreadPool(() => rootLifetimeScope.Build(), cancellationToken: cancellationToken);
    }
    return rootLifetimeScope;
  }
}
```

## おわりに

『Navigathena』は「Navigation」に、知略の神様である「Athena」を合わせた造語で、「画面遷移に知略を与える」というコンセプトを持っています。神様の名前を冠するだけあって、設計には時間を要していました。

業務や個人開発を平行しながら、類似事例を調べたりしてライブラリの設計を考えていましたが、ようやく納得できる形で一つの答えを出せたかと思います。
ゲーム開発者としてこれからの開発でも使っていくので、使っていて気になった機能の改良を随時加えていく所存です。

NavigathenaはMITライセンスの下で公開してあります。

https://github.com/mackysoft/Navigathena

GitHubで☆を付けていただけると、嬉しくなりますm(_ _)m

## 参考

- [ミラティブでのアウトゲーム設計の紹介](https://tech.mirrativ.stream/entry/2023/09/22/112042)
- [Web出身のUnityエンジニアによる大規模ゲームの基盤設計](https://developers.cyberagent.co.jp/blog/archives/4262/)
- [【Unity】新規ゲームのUI開発で気をつけた39のTips前編](https://qiita.com/ohbashunsuke/items/0570234362e2e45c0011)
- [Unityでの複数シーンを使ったゲームの実装方法とメモリリークについて](https://madnesslabo.net/utage/?page_id=11109)
- [Unity専用最速DIコンテナVContainer と、UnityにおけるDIの勘所](https://scrapbox.io/hadashiA/Unity%E5%B0%82%E7%94%A8%E6%9C%80%E9%80%9FDI%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8AVContainer_%E3%81%A8%E3%80%81Unity%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8BDI%E3%81%AE%E5%8B%98%E6%89%80)
