## どうやって構造を作るか？

- ツリーをビルド
- ScreenIdentifier
  - CanvasScreen
  - GameObjectScreen
  - PrefabScreen
  - SceneScreen
  - AddressablePrefabScreen
  - AddressableSceneScreen
- ScreenIdentifierとScreenEntryPointを結びつける必要がある
  - ScreenEntryPointはシーンに配置する必要がある？
- IScreenNavigator
  - Push<T> ()
    - 完全型付け方式？
      - EntryPointだけでなくLifecycleも関わるので完全は難しい。恐らくTypeDictionaryで紐づける形になる
  - Pop ()
  - 演出機構はSceneManagementと共有
- ScreenEntryPoint
  - 動的生成でもいい

```cs
// Screenに渡すParameterを完全型付けしたい
m_Navigator.Push(CraftScreenDefinitions.Startup.With(new StartupScreenParameter()));
```

使い方

1. ScreenContainerを継承したコンポーネントをシーンに配置する
   1. 必要なCanvasやプレハブをインスペクタで設定する
2. Buildすると、ScreenNavigatorで使用するScreenResolverが生成される
   1. Dictionaryが作られる
   2. EntryPoint（コンポーネント派生）だけでなく、Lifecycle（DI用）があるので、完全型付けは難しい
3. NavigatorでPushなどをすると、ScreenResolverがScreenを解決し、ScreenHandleからScreenを生成する
   1. 事前配置、動的生成は問わない
   2. ScreenHandleはScreenEntryPointを返す必要がある（プレハブなどからGetComponentする必要がある）