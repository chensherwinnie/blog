---
title:  "UniTask- 官方Read.md中文翻譯"
date:   2021-01-16
categories: Unity
permalink: "blog/unity/:title"
tags: C# learning
---

### 譯注
<hr>
UniTask是一個非常強大的函式庫，由UniRx的作者neuecc與mayuki一同維護。他原先被包含在UniRx之中，後來被獨立分佈出來。他的用意是為了讓Unity的異步處理獲得性能上以及功能性上的提升。就算對異步處理不有巨大的需求，也可以將他直接引進project中，將所有的Task直接取代。有鑒於他的強大性（以及網路上還找不到中文文檔翻譯，還有我本人想要好好理解這個函式庫的緣故），所以花了點時間，將他GitHub上的文檔中譯。

本人只是一個大三的學生，對異步處理這方面（以及其他許許多多的方面）不是特別熟悉。即使翻譯完了，還是有很多地方不是很確定原文的意思。我會繼續嘗試跟作者確認他的意思。期間若有錯誤，請多多指教。

[原文連結](https://github.com/Cysharp/UniTask)

本文基於版本2.1.0翻譯。

已經過作者同意翻譯。

![contactWithNeucc]({{ site.baseurl }}/assets/images/why.jpg)

# 正文開始
<hr>
UniTask
===
[![GitHub Actions](https://github.com/Cysharp/UniTask/workflows/Build-Debug/badge.svg)](https://github.com/Cysharp/UniTask/actions) [![Releases](https://img.shields.io/github/release/Cysharp/UniTask.svg)](https://github.com/Cysharp/UniTask/releases)

為Unity帶來高效率的，無記憶體分配（zero allocation）的`async/await`整合。

* 使用結構建造（struct-based）的`UniTask<T>` 還有自訂的AsyncMethodBuilder來達成零記憶體使用。
* 所有的Unity AsyncOperations還有Coroutine都可以變成awaitable
* 使用玩家迴圈（PlayerLoop）為基礎的task(`UniTask.Yield`, `UniTask.Delay`, `UniTask.DelayFrame`, etc..) 來取代所有的coroutine operation
* MonoBehaviour的訊息事件（Unity Message Events）還有uGUI事件（uGUI Events） 都能成為awaitable/async-enumerable
* 完全執行在Unity的玩家迴圈，所以不需要使用線程在WebGL, wasm等等平台上執行
* 異步LINQ, 可與Channel以及AsyncReactiveProperty一同使用
* TaskTracker視窗來防止記憶體洩漏
* 高度配合Task/ValueTask/IValueTaskSource

技術細節, 請見此部落格文章: [UniTask v2 — Zero Allocation async/await for Unity, with Asynchronous LINQ
](https://medium.com/@neuecc/unitask-v2-zero-allocation-async-await-for-unity-with-asynchronous-linq-1aa9c96aa7dd)  
進階技巧，請見此·部落格: [Extends UnityWebRequest via async decorator pattern — Advanced Techniques of UniTask](https://medium.com/@neuecc/extends-unitywebrequest-via-async-decorator-pattern-advanced-techniques-of-unitask-ceff9c5ee846)

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Getting started](#getting-started)
- [Basics of UniTask and AsyncOperation](#basics-of-unitask-and-asyncoperation)
- [Cancellation and Exception handling](#cancellation-and-exception-handling)
- [Progress](#progress)
- [PlayerLoop](#playerloop)
- [async void vs async UniTaskVoid](#async-void-vs-async-unitaskvoid)
- [UniTaskTracker](#unitasktracker)
- [External Assets](#external-assets)
- [AsyncEnumerable and Async LINQ](#asyncenumerable-and-async-linq)
- [Awaitable Events](#awaitable-events)
- [Channel](#channel)
- [For Unit Testing](#for-unit-testing)
- [ThreadPool limitation](#threadpool-limitation)
- [IEnumerator.ToUniTask limitation](#ienumeratortounitask-limitation)
- [For UnityEditor](#for-unityeditor)
- [Compare with Standard Task API](#compare-with-standard-task-api)
- [Pooling Configuration](#pooling-configuration)
- [Allocation on Profiler](#allocation-on-profiler)
- [UniTaskSynchronizationContext](#unitasksynchronizationcontext)
- [API References](#api-references)
- [UPM Package](#upm-package)
  - [Install via git URL](#install-via-git-url)
  - [Install via OpenUPM](#install-via-openupm)
- [.NET Core](#net-core)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Getting started
---
通過[UPM package](#upm-package)或是在[UniTask/releases](https://github.com/Cysharp/UniTask/releases)提供的asset package(`UniTask.*.*.*.unitypackage`) 下載。

```csharp
// extension awaiter/methods can be used by this namespace
using Cysharp.Threading.Tasks;

// You can return type as struct UniTask<T>(or UniTask), it is unity specialized lightweight alternative of Task<T>
// zero allocation and fast excution for zero overhead async/await integrate with Unity
async UniTask<string> DemoAsync()
{
    // You can await Unity's AsyncObject
    var asset = await Resources.LoadAsync<TextAsset>("foo");
    var txt = (await UnityWebRequest.Get("https://...").SendWebRequest()).downloadHandler.text;
    await SceneManager.LoadSceneAsync("scene2");

    // .WithCancellation enables Cancel, GetCancellationTokenOnDestroy synchornizes with lifetime of GameObject
    var asset2 = await Resources.LoadAsync<TextAsset>("bar").WithCancellation(this.GetCancellationTokenOnDestroy());

    // .ToUniTask accepts progress callback(and all options), Progress.Create is a lightweight alternative of IProgress<T>
    var asset3 = await Resources.LoadAsync<TextAsset>("baz").ToUniTask(Progress.Create<float>(x => Debug.Log(x)));

    // await frame-based operation like coroutine
    await UniTask.DelayFrame(100);

    // replacement of yield return new WaitForSeconds/WaitForSecondsRealtime
    await UniTask.Delay(TimeSpan.FromSeconds(10), ignoreTimeScale: false);

    // yield any playerloop timing(PreUpdate, Update, LateUpdate, etc...)
    await UniTask.Yield(PlayerLoopTiming.PreLateUpdate);

    // replacement of yield return null
    await UniTask.Yield();
    await UniTask.NextFrame();

    // replacement of WaitForEndOfFrame(same as UniTask.Yield(PlayerLoopTiming.LastPostLateUpdate))
    await UniTask.WaitForEndOfFrame();

    // replacement of yield return new WaitForFixedUpdate(same as UniTask.Yield(PlayerLoopTiming.FixedUpdate))
    await UniTask.WaitForFixedUpdate();

    // replacement of yield return WaitUntil
    await UniTask.WaitUntil(() => isActive == false);

    // special helper of WaitUntil
    await UniTask.WaitUntilValueChanged(this, x => x.isActive);

    // You can await IEnumerator coroutine
    await FooCoroutineEnumerator();

    // You can await standard task
    await Task.Run(() => 100);

    // Multithreading, run on ThreadPool under this code
    await UniTask.SwitchToThreadPool();

    /* work on ThreadPool */

    // return to MainThread(same as `ObserveOnMainThread` in UniRx)
    await UniTask.SwitchToMainThread();

    // get async webrequest
    async UniTask<string> GetTextAsync(UnityWebRequest req)
    {
        var op = await req.SendWebRequest();
        return op.downloadHandler.text;
    }

    var task1 = GetTextAsync(UnityWebRequest.Get("http://google.com"));
    var task2 = GetTextAsync(UnityWebRequest.Get("http://bing.com"));
    var task3 = GetTextAsync(UnityWebRequest.Get("http://yahoo.com"));

    // concurrent async-wait and get result easily by tuple syntax
    var (google, bing, yahoo) = await UniTask.WhenAll(task1, task2, task3);

    // shorthand of WhenAll, tuple can await directly
    var (google2, bing2, yahoo2) = await (task1, task2, task3);

    // You can handle timeout easily
    await GetTextAsync(UnityWebRequest.Get("http://unity.com")).Timeout(TimeSpan.FromMilliseconds(300));

    // return async-value.(or you can use `UniTask`(no result), `UniTaskVoid`(fire and forget)).
    return (asset as TextAsset)?.text ?? throw new InvalidOperationException("Asset not found");
}
```

Basics of UniTask and AsyncOperation
---
UniTask的機能是基於C# 7.0的([task-like custom async method builder feature](https://github.com/dotnet/roslyn/blob/master/docs/features/task-types.md))，所以需要至少高於`Unity 2018.3`之後的版本，官方最低的支持版本為`Unity 2018.4.13f1`。

為什麼UniTask（自訂的task-like object）是必須的呢？因為正規的Task太過笨重，無法在Unitry的線程（單線程）運作。UniTask不使用線程，也不使用SynchronizationContext/ExecutionContext，因為Unity的異步物件都會被自動被Unity的引擎派出。Unity需要更快的以及更少記憶體需求的Task，來完全整合進Unity中。

當你使用`using Cysharp.Threading.Tasks;`時，你可以await
You can await `AsyncOperation`, `ResourceRequest`, `AssetBundleRequest`, `AssetBundleCreateRequest`, `UnityWebRequestAsyncOperation`, `AsyncGPUReadbackRequest`, `IEnumerator`。

UniTask提供了三種不同形式的擴充方法。

```csharp
* await asyncOperation;
* .WithCancellation(CancellationToken);
* .ToUniTask(IProgress, PlayerLoopTiming, CancellationToken);
```

`WithCancellation` 是一種更為簡單的 `ToUniTask`, 他們都會回傳 `UniTask`. 關於取消的細節，請見[Cancellation and Exception handling](#cancellation-and-exception-handling)。

> 注：WithCancellation會在正常的玩家迴圈時被回傳，但是ToUniTask會在指定的玩家迴圈時刻（PlayerLoopTimeing）被回傳。細節請見：[PlayerLoop](#playerloop)。

> 注：AssetBundleRequest有`asset`還有`allAssets`兩種屬性，預設await會回傳`asset`。如果你想要取得`allAssets`，你可以使用`AwaitForAllAssets()`。

UniTask類型可以使用`UniTask.WhenAll`, `UniTask.WhenAny`等等功能。他就像是Task.WhenAll/WhenAny但是回傳的值更加有用。他回傳value tuple，所以我們可以拆解各個結果，也可以傳進不同的類型進入方法內。

```csharp
public async UniTaskVoid LoadManyAsync()
{
    // parallel load.
    var (a, b, c) = await UniTask.WhenAll(
        LoadAsSprite("foo"),
        LoadAsSprite("bar"),
        LoadAsSprite("baz"));
}

async UniTask<Sprite> LoadAsSprite(string path)
{
    var resource = await Resources.LoadAsync<Sprite>(path);
    return (resource as Sprite);
}
```

如果你想要將callback轉換成UniTask，你可以使用`UniTaskCompletionSource<T>`。它基本上就是個較輕便的`TaskCompletionSource<T>`。

```csharp
public UniTask<int> WrapByUniTaskCompletionSource()
{
    var utcs = new UniTaskCompletionSource<int>();

    // when complete, call utcs.TrySetResult();
    // when failed, call utcs.TrySetException();
    // when cancel, call utcs.TrySetCanceled();

    return utcs.Task; //return UniTask<int>
}
```

你可以將Task轉化為UniTask：`AsUniTask`, `UniTask` -> `UniTask<AsyncUnit>`: `AsAsyncUnitUniTask`, `UniTask<T>` -> `UniTask`: `AsUniTask`. `UniTask<T>` -> `UniTask`。這些轉化都是無需消耗資源的。

你也可以將async轉化成coroutine：`.ToCoroutine()`。這在你只能使用coroutine系統的時候很方便。

UniTask不能被await超過一次。這是一個跟在.NET Standard 2.1被引進的[ValueTask/IValueTaskSource](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask-1?view=netcore-3.1)類似的限制。

> 下列的行動永遠不應該被執行在一個ValueTask<TResult>的實例（Instance）：
>
> * Await一個實例超過一次。
> * 呼叫AsTask超過一次。
> * 在行動還尚未完成的時候，使用.Result或是.GetAwaiter().GetResult()，或是使用他們超過一次。
> * Using more than one of these techniques to consume the instance.
> * （譯注：上面這句話看不太懂。如果你永遠不應該使用這些技巧，那怎麼會發生這個情況呢？）
>
> 如果你做了上列的任何一項，結果都是無法定義的（undefined）。
> (譯注：注意，這裏指的是`ValueTask<TResult>`，與之下的`UniTask.Lazy`跟)


```csharp
var task = UniTask.DelayFrame(10);
await task;
await task; // NG, throws Exception
```

如果想儲存至一個類別欄位（class field），你可以使用`UniTask.Lazy`來保證多次呼叫。`.Preserve()` 也支持多次呼叫（會在內部儲存回傳結果）。這在一個方法範圍內需要多次呼叫時很有用。

譯注：原文這裏寫的不是太清楚，我增加從[這裏](https://qiita.com/toRisouP/items/8f66fd952eaffeaf3107)以及[這裏](https://baba-s.hatenablog.com/entry/2019/09/11/083000)找到的兩個範例。

Lazy:

```csharp
//譯注：UniRx是另一項為Unity製造的第三方工具。在UniRx 7.0版本之後，UniTask被從UniRx分離。
//現在要使用的話，使用using Cysharp.Threading.Tasks;
using UniRx.Async;
using UnityEngine;

public sealed class Example : MonoBehaviour
{
    public UniTask<TextAsset> TextAsset { get; }
        = UniTask.Lazy( () => LoadTextAsset() );

    private async void Start()
    {
        var textAsset1 = await TextAsset; // 1回目は非同期処理が実行される （譯注：第一次被呼叫，執行非同期處理）
        var textAsset2 = await TextAsset; // 2回目以降は結果のみが取得される（譯注：第二次以及之後被呼叫，直接從已經獲得的結果拿取結果）
        var textAsset3 = await TextAsset;
    }

    private static async UniTask<TextAsset> LoadTextAsset()
    {
        // 1回だけログが出力される
        Debug.Log( "初期化" );
        var textAsset = await Resources.LoadAsync<TextAsset>( "text" );
        return textAsset as TextAsset;
    }
}
```

.Preserve()
```csharp
private async UniTaskVoid DoAsync(CancellationToken token)
{
    try
    {
        var uniTask = GetAsync("https://unity.com/ja", token);

        // Preserve()で何回でもawait可能なUniTaskに変換
        var reusable = uniTask.Preserve();

        await reusable;
        await reusable;
    }
    catch (InvalidOperationException e)
    {
        Debug.LogException(e);
    }
}
```

還有，`UniTaskCompletionSource`可以被await多於一次而且也可以被多個呼叫者await。

Cancellation and Exception handling
---
有些UniTask的工廠方法（factory methods）有`CancellationToken cancellationToken = default`的參數。還有一些Unity異步行動也有`WithCancellation(CancellationToken)`還有 `ToUniTask(..., CancellationToken cancellation = default)`的擴充方法。

你可以輸入`CancellationToken`至參數，用標準的[`CancellationTokenSource`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtokensource).

```csharp
var cts = new CancellationTokenSource();

cancelButton.onClick.AddListener(() =>
{
    cts.Cancel();
});

await UnityWebRequest.Get("http://google.co.jp").SendWebRequest().WithCancellation(cts.Token);

await UniTask.DelayFrame(1000, cancellationToken: cts.Token);
```

CancellationToken可以被`CancellationTokenSource`或是MonoBehaviour的擴充方法`GetCancellationTokenOnDestroy`製造。

```csharp
// this CancellationToken lifecycle is same as GameObject.
await UniTask.DelayFrame(1000, cancellationToken: this.GetCancellationTokenOnDestroy());
```

當偵測到取消時，所有的方法都會拋出一個`OperationCanceledException`到上流去. `OperationCanceledException`是一個特殊的例外狀況，若沒有被處理的話，他最後會被拋到`UniTaskScheduler.UnobservedTaskException`。

當`UniTaskScheduler`拿到一個未被處理的例外狀況時，預設的行為是將那個例外狀況列出在log中。Log level可以被`UniTaskScheduler.UnobservedExceptionWriteLogType`轉變。如果你想要自訂行為，那就設置行動到`UniTaskScheduler.UnobservedTaskException`。

如果你想要在一個async UniTask的方法內取消行為，那就手動拋出`OperationCanceledException`。

```csharp
public async UniTask<int> FooAsync()
{
    await UniTask.Yield();
    throw new OperationCanceledException();
}
```

如果你想要處理錯誤，但是想要忽視`OperationCanceledException`（讓他被拋到廣處理（global concellation handling）），那就過濾例外狀況。

```csharp
public async UniTask<int> BarAsync()
{
    try
    {
        var x = await FooAsync();
        return x * 2;
    }
    catch (Exception ex) when (!(ex is OperationCanceledException))
    {
        return -1;
    }
}
```

throws/catch`OperationCanceledException`有一點消耗資源。如果你在乎效能的話，用`UniTask.SuppressCancellationThrow`來迴避拋出OperationCanceledException。他會回傳`(bool IsCanceled, T Result)`而不會拋出錯誤。

```csharp
var (isCanceled, _) = await UniTask.DelayFrame(10, cancellationToken: cts.Token).SuppressCancellationThrow();

if (isCanceled)
{
    // ...
}
```

Note: Only suppress throws if you call it directly into the most source method. Otherwise, the return value will be converted, but the entire pipeline will not be suppressed throws.
（譯注：這句有點看不懂，請高人指教⋯⋯）

Progress
---
有一些異步處理行動有`ToUniTask(IProgress<float> progress = null, ...)`的擴充方法。

```csharp
var progress = Progress.Create<float>(x => Debug.Log(x));

var request = await UnityWebRequest.Get("http://google.co.jp")
    .SendWebRequest()
    .ToUniTask(progress: progress);
```

你不應該使用標準的`new System.Progress<T>`，因為他會導致太多次記憶體分佈。`Cysharp.Threading.Tasks.Progress`取而代之。這個progresse工廠有兩個方法，`Create`還有`CreateOnlyValueChanged`。`CreateOnlyValueChanged`只會在progress的值變化的時候被呼叫。

植入IProgress interface到呼叫者中會更好，lambo不會導致記憶體分配。

```csharp
public class Foo : MonoBehaviour, IProgress<float>
{
    public void Report(float value)
    {
        UnityEngine.Debug.Log(value);
    }

    public async UniTaskVoid WebRequest()
    {
        var request = await UnityWebRequest.Get("http://google.co.jp")
            .SendWebRequest()
            .ToUniTask(progress: this); // pass this
    }
}
```

PlayerLoop
---
（譯注：說實在話，這下面的內容我有點看不懂⋯⋯）

UniTask在一個自訂的[玩家迴圈（PlayerLoop）](https://docs.unity3d.com/ScriptReference/LowLevel.PlayerLoop.html)上執行。UniTask中以玩家迴圈為基礎的方法（像`Delay`, `DelayFrame`, `asyncOperation.ToUniTask`, 等等）都會接收`PlayerLoopTiming`。

```csharp
public enum PlayerLoopTiming
{
    Initialization = 0,
    LastInitialization = 1,

    EarlyUpdate = 2,
    LastEarlyUpdate = 3,

    FixedUpdate = 4,
    LastFixedUpdate = 5,

    PreUpdate = 6,
    LastPreUpdate = 7,

    Update = 8,
    LastUpdate = 9,

    PreLateUpdate = 10,
    LastPreLateUpdate = 11,

    PostLateUpdate = 12,
    LastPostLateUpdate = 13

#if UNITY_2020_2_OR_NEWER
    TimeUpdate = 14,
    LastTimeUpdate = 15,
#endif
}
```

他指示了什麼時候會運作，你可以在[PlayerLoopList.md](https://gist.github.com/neuecc/bc3a1cfd4d74501ad057e49efcd7bdae)確認Unity的預設玩家迴圈以及輸入UniTask的自訂迴圈。

`PlayerLoopTiming.Update`跟coroutine的`yield return null`很像，但是他會在每次Update(Update還有uGUI事件(button.onClick, etc...)在 `ScriptRunBehaviourUpdate`上被呼叫, `yield return null`則是在`ScriptRunDelayedDynamicFrameRate`被呼叫。`PlayerLoopTiming.FixedUpdate`跟`WaitForFixedUpdate`很像，`PlayerLoopTiming.LastPostLateUpdate`跟`WaitForEndOfFrame`類似。

> await UniTask.WaitForEndOfFrame()不等同於coroutine的`yield return new WaitForEndOfFrame()`。 Coroutine的WaitForEndOfFrame應該會在PlayerLoop*結束後*執行。有一些方法需要coroutine的幀尾（end of frame）(像是`ScreenCapture.CaptureScreenshotAsTexture`, `CommandBuffer`, etc) 當async/await時無法被正確執行。這個情況時，用coroutine。

`yield return null`還有`UniTask.Yield`有些類似，但是卻不一樣。`yield return null`總是會回傳下一幀，但是`UniTask.Yield`會在下一次呼叫時回傳。也就是說，在`PreUpdate`時呼叫`UniTask.Yield(PlayerLoopTiming.Update)`，他會回傳同一幀。`UniTask.NextFrame()`保證回傳下一幀，而且也被認為跟`yield return null`一模一樣。

> UniTask.Yield(沒有CancellationToken)是一個特殊的類型。它會回傳`YieldAwaitable`並且在YieldRunner上執行。他是最輕便的，而且更快。

異步行動會被原生的時間回傳。比如說，await`SceneManager.LoadSceneAsync`會從`EarlyUpdate.UpdatePreloading`被回傳。在被呼叫後，已經載入的scene的`Start`會從`EarlyUpdate.ScriptRunDelayedStartupFrame`被呼叫。還有，`await UnityWebRequest`是從 `EarlyUpdate.ExecuteMainThreadJobs`被回傳的。

在UniTask中，直接使用await還有`WithCancellation`使用原生的時間，`ToUniTask`則使用被指定的時間。這通常不是個問題，但是跟`LoadSceneAsync`, 一起被使用時，或導致Start的執行以及之後的順序被改（causes different order of Start and continuation after await），所以不建議使用`LoadSceneAsync.ToUniTask`。

在stacktrace中，你可以查看哪裡是在玩家迴圈中執行。

![image](https://user-images.githubusercontent.com/46207/83735571-83caea80-a68b-11ea-8d22-5e22864f0d24.png)

預設中，UniTask的玩家迴圈是在`[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]`中被實例化的。

在BeforeSceneLoaded中，方法被執行的順序無法被判定，所以如果你想要用UniTask在其他的BeforeSceneLoad的方法中，你應該先實例化他。

```csharp
// AfterAssembliesLoaded is called before BeforeSceneLoad
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.AfterAssembliesLoaded)]
public static void InitUniTaskLoop()
{
    var loop = PlayerLoop.GetCurrentPlayerLoop();
    Cysharp.Threading.Tasks.PlayerLoopHelper.Initialize(ref loop);
}
```

如果輸入了Unity的`Entities`包，那個包會重設自訂的玩家迴圈到`BeforeSceneLoad`然後輸入ECS的迴圈。在UniTask被實例化後，當Unity呼叫ECS的植入方法，UniTask就不會運作了。

你可以通過在ECS實例化後重新實例化UniTask PlayerLoop來解決這個問題。

```csharp
// Get ECS Loop.
var playerLoop = ScriptBehaviourUpdateOrder.CurrentPlayerLoop;

// Setup UniTask's PlayerLoop.
PlayerLoopHelper.Initialize(ref playerLoop);
```

你可以通過`PlayerLoopHelper.IsInjectedUniTaskPlayerLoop()`來確定UniTask的玩圈迴圈是否已經準備完畢，還有`PlayerLoopHelper.DumpCurrentPlayerLoop`來顯示所有的玩家迴圈到Unity的console上。

```csharp
void Start()
{
    UnityEngine.Debug.Log("UniTaskPlayerLoop ready? " + PlayerLoopHelper.IsInjectedUniTaskPlayerLoop());
    PlayerLoopHelper.DumpCurrentPlayerLoop();
}
```

async void vs async UniTaskVoid
---
`async void`是一個標準的C# Task系統，所以他不會在UniTask系統上運作。用`async UniTaskVoid`更好，他是一個輕便版本的`async UniTask`，因為他不需要有awiatbale的結束，而且會馬上回到錯誤至`UniTaskScheduler.UnobservedTaskException`。如果你不需要await他（射後不理(fire and forget)（譯注：這是正式的翻譯）），那`UniTaskVoid`會比較好。不幸的是，如果想要取消警告，那麼你需要用`Forget()`。

```csharp
public async UniTaskVoid FireAndForgetMethod()
{
    // do anything...
    await UniTask.Yield();
}

public void Caller()
{
    FireAndForgetMethod().Forget();
}
```

還有，UniTask有`Forget`方法。他跟`UniTaskVoid`有著同樣的效果。但是UniTaskVoid在完全不使用await的情況下比較有效率。

```csharp
public async UniTask DoAsync()
{
    // do anything...
    await UniTask.Yield();
}

public void Caller()
{
    DoAsync().Forget();
}
```

使用async lamba來登記事件，但是他會用`async void`。這不太好。你可以使用`UniTask.Action`或是`UniTask.UnityAction`（他們會製造`async UniTaskVoid` lamba）來迴避這個問題。

```csharp
Action actEvent;
UnityAction unityEvent; // especially used in uGUI

// Bad: async void
actEvent += async () => { };
unityEvent += async () => { };

// Ok: create Action delegate by lambda
actEvent += UniTask.Action(async () => { await UniTask.Yield(); });
unityEvent += UniTask.UnityAction(async () => { await UniTask.Yield(); });
```

`UniTaskVoid`也可以在MonoBehaviour的`Start`被使用。

```csharp
class Sample : MonoBehaviour
{
    async UniTaskVoid Start()
    {
        // async init code.
    }
}
```

UniTaskTracker
---
這對偵查洩漏的UniTasks很有用。你通過`Window -> UniTask Tracker`打開這個視窗。

![image](https://user-images.githubusercontent.com/46207/83527073-4434bf00-a522-11ea-86e9-3b3975b26266.png)

* 開啟AutoReload(Toggle) - 自動重新載入
* Reload - 重新載入
* GC.Collect - 使用GC.Collect.
* 開啟Tracking(Toggle) - 開始偵測async/await UniTask。對效能的影響：低
* 開啟StackTrace(Toggle) - 當task開始時，捕捉StackTrace。對效能的影響：低

開啟tracking and capture stacktrace很有用，但是他們會影響效能。建議在需要找到task leak的時候開啟他們，但是不需要的時候關閉。

External Assets
---
預設中，UniTask支持 TextMeshPro(`BindTo(TMP_Text)`還有`TMP_InputField`的擴充，就像標準的`InputField`)，DOTween(`Tween`轉為awaitable)還有Addressables(`AsyncOperationHandle`跟`AsyncOpereationHandle<T>`轉為awaitable).

他們在不同的asmdef（譯注：assembly definiton，第一次看到這樣的縮寫），例如`UniTask.TextMeshPro`, `UniTask.DOTween`, `UniTask.Addressables`。

TextMeshPro還有Addressables的支持會被自動開啟，當他們的package從package manager被輸入進來時。但是DOTween的支持，需要 `com.demigiant.dotween`從[OpenUPM](https://openupm.com/packages/com.demigiant.dotween/)被輸入進來，或是define `UNITASK_DOTWEEN_SUPPORT`來開啟。

```csharp
// sequential
await transform.DOMoveX(2, 10);
await transform.DOMoveZ(5, 20);

// parallel with cancellation
var ct = this.GetCancellationTokenOnDestroy();

await UniTask.WhenAll(
    transform.DOMoveX(10, 3).WithCancellation(ct),
    transform.DOScale(10, 3).WithCancellation(ct));
```

（譯注：我沒有使用過DOTween，因此不是很確定下面這段是什麼意思，無法翻譯。我保留原文，以便有需要的人閱讀。）

DOTween support's default behaviour(`await`, `WithCancellation`, `ToUniTask`) awaits tween is killed. It works both Complete(true/false) and Kill(true/false). But if you want to tween reuse(`SetAutoKill(false)`), it does not work you expected. Or, if you want to await for another timing, the following extension methods exist in Tween, `AwaitForComplete`, `AwaitForPause`, `AwaitForPlay`, `AwaitForRewind`, `AwaitForStepComplete`.

AsyncEnumerable and Async LINQ
---
Unity 2020.2 開始支持C# 8.0，所以你可以使用 `await foreach`

```csharp
// Unity 2020.2, C# 8.0
await foreach (var _ in UniTaskAsyncEnumerable.EveryUpdate(token))
{
    Debug.Log("Update() " + Time.frameCount);
}
```

在 C# 7.3 的環境中，你可以使用，`ForEachAsync`。他基本上跟上述方法功能一致。


```csharp
// C# 7.3(Unity 2018.3~)
await UniTaskAsyncEnumerable.EveryUpdate(token).ForEachAsync(_ =>
{
    Debug.Log("Update() " + Time.frameCount);
});
```

UniTaskAsyncEnumerable 植入了異步的 LINQ，跟在`IEnumerable<T>`或是迴響式擴充（譯注：Reactive Extension，另一個大坑）的`IObservable<T>`的LINQ類似。所有的標準的LINQ query operators都可以在異步流（asynchronous stream）中被使用。例如，下列的碼展示了要如何輸入一個Where到一個連續點擊兩次按鈕的異步流之中。

```csharp
await okButton.OnClickAsAsyncEnumerable().Where((x, i) => i % 2 == 0).ForEachAsync(_ =>
{
});
```

射後忘記的方式，你也可以用`Subscribe`。

```csharp
okButton.OnClickAsAsyncEnumerable().Where((x, i) => i % 2 == 0).Subscribe(_ =>
{
});
```

異步LINQ會在`using Cysharp.Threading.Tasks.Linq;`中被啟用，還有`UniTaskAsyncEnumerable`在`UniTask.Linq`中被定義。

他跟UniRX（迴響式擴充）有點像，但是UniTaskAsyncEnumerable是一個拉取式的異步流（pull-based asynchronous stream），而Rx是一個推取式的異步流（push-based asynchronous stream）。注意，雖然他們相似，但是他們的性質是不同的，而且細節執行面上會有很多不同的地方。

`UniTaskAsyncEnumerable`是跟`Enumerbale`類似的起始點。除了那些標準的query operators外，還有一些特殊的Unity專用的生產器，像`EveryUpdate`, `Timer`, `TimerFrame`, `Interval`, `IntervalFrame`, 還有`EveryValueChanged`. 除此之外，還有一些UniTask原創的query operators，`Append`, `Prepend`, `DistinctUntilChanged`, `ToHashSet`, `Buffer`, `CombineLatest`, `Do`, `Never`, `ForEachAsync`, `Pairwise`, `Publish`, `Queue`, `Return`, `SkipUntil`, `TakeUntil`, `SkipUntilCanceled`, `TakeUntilCanceled`, `TakeLast`, `Subscribe`。

有Func為引數的方法還有三種不同的多載方法，`***Await`, `***AwaitWithCancellation`。

```csharp
Select(Func<T, TR> selector)
SelectAwait(Func<T, UniTask<TR>> selector)
SelectAwaitWithCancellation(Func<T, CancellationToken, UniTask<TR>> selector)
```

如果你想要在Func內使用`async`，那就用`***Await`或`***AwaitWithCancellation`。

要怎麼產生一個異步疊代器（async iterator）呢？C# 8.0之後支持異步疊代器(`async yield return`)，但他只允許`IAsyncEnumerable<T>`。 UniTask支持 `UniTaskAsyncEnumerable.Create`來產生自訂的異步疊代器。

```csharp
// IAsyncEnumerable, C# 8.0 version of async iterator. ( do not use this style, IAsyncEnumerable is not controled in UniTask).
public async IAsyncEnumerable<int> MyEveryUpdate([EnumeratorCancellation]CancellationToken cancelationToken = default)
{
    var frameCount = 0;
    await UniTask.Yield();
    while (!token.IsCancellationRequested)
    {
        yield return frameCount++;
        await UniTask.Yield();
    }
}

// UniTaskAsyncEnumerable.Create and use `await writer.YieldAsync` instead of `yield return`.
public IUniTaskAsyncEnumerable<int> MyEveryUpdate()
{
    // writer(IAsyncWriter<T>) has `YieldAsync(value)` method.
    return UniTaskAsyncEnumerable.Create<int>(async (writer, token) =>
    {
        var frameCount = 0;
        await UniTask.Yield();
        while (!token.IsCancellationRequested)
        {
            await writer.YieldAsync(frameCount++); // instead of `yield return`
            await UniTask.Yield();
        }
    });
}
```

Awaitable Events
---
所有的uGUI部件都植入了`***AsAsyncEnumerable`來轉換事件的異步流（asynchronous streams of events）

```csharp
async UniTask TripleClick()
{
    // In default, used button.GetCancellationTokenOnDestroy to manage lieftime of async
    await button.OnClickAsync();
    await button.OnClickAsync();
    await button.OnClickAsync();
    Debug.Log("Three times clicked");
}

// more efficient way
async UniTask TripleClick()
{
    using (var handler = button.GetAsyncClickEventHandler())
    {
        await handler.OnClickAsync();
        await handler.OnClickAsync();
        await handler.OnClickAsync();
        Debug.Log("Three times clicked");
    }
}

// use async LINQ
async UniTask TripleClick(CancellationToken token)
{
    await button.OnClickAsAsyncEnumerable().Take(3).Last();
    Debug.Log("Three times clicked");
}

// use async LINQ2
async UniTask TripleClick(CancellationToken token)
{
    await button.OnClickAsAsyncEnumerable().Take(3).ForEachAsync(_ =>
    {
        Debug.Log("Every clicked");
    });
    Debug.Log("Three times clicked, complete.");
}
```

所有的 MonoBehaviour Message Events都能用`AsyncTriggers`轉化成異步流。他們在`using Cysharp.Threading.Tasks.Triggers;`裡。

```csharp
using Cysharp.Threading.Tasks.Triggers;

async UniTaskVoid MonitorCollision()
{
    await gameObject.OnCollisionEnterAsync();
    Debug.Log("Collision Enter");
    /* do anything */

    await gameObject.OnCollisionExitAsync();
    Debug.Log("Collision Exit");
}
```

跟uGUI事件類似，`AsyncTrigger`可以用`GetAsync***Trigger`獲得。Trigger本身是UniTaskAsyncEnumerable。

```csharp
// use await multiple times, get AsyncTriggerHandler is more efficient.
using(var trigger = this.GetOnCollisionEnterAsyncHandler())
{
    await OnCollisionEnterAsync();
    await OnCollisionEnterAsync();
    await OnCollisionEnterAsync();
}

// every moves.
await this.GetAsyncMoveTrigger().ForEachAsync(axisEventData =>
{
});
```

`AsyncReactiveProperty`, `AsyncReadOnlyReactiveProperty`是UniTask版本的ReactiveProperty。`IUniTaskAsyncEnumerable<T>`的擴充方法`BindTo`可以將異步流跟Unity部件(Text/Selectable/TMP/Text)結合。

```csharp
var rp = new AsyncReactiveProperty<int>(99);

// AsyncReactiveProperty itself is IUniTaskAsyncEnumerable, you can query by LINQ
rp.ForEachAsync(x =>
{
    Debug.Log(x);
}, this.GetCancellationTokenOnDestroy()).Forget();

rp.Value = 10; // push 10 to all subscriber
rp.Value = 11; // push 11 to all subscriber

// WithoutCurrent ignore initial value
// BindTo bind stream value to unity components.
rp.WithoutCurrent().BindTo(this.textComponent);

await rp.WaitAsync(); // wait until next value set

// also exists ToReadOnlyReactiveProperty
var rp2 = new AsyncReactiveProperty<int>(99);
var rorp = rp.CombineLatest(rp2, (x, y) => (x, y)).ToReadOnlyReactiveProperty();
```

直到異步處理完畢前，拉取式的異步流不會獲得下一個值。這可以將資料從推取式的事件（push-type event）中拉出（例如按鈕）

```csharp
// can not get click event during 3 seconds complete.
await button.OnClickAsAsyncEnumerable().ForEachAwaitAsync(async x =>
{
    await UniTask.Delay(TimeSpan.FromSeconds(3));
});
```

這有時候很有用（阻止連續點擊），但是有時候卻不太有用。

用`Queue()`方法。在異步處理時，他會將事件排列起來（queue events during asynchronous processing）

```csharp
// queued message in asynchronous processing
await button.OnClickAsAsyncEnumerable().Queue().ForEachAwaitAsync(async x =>
{
    await UniTask.Delay(TimeSpan.FromSeconds(3));
});
```

或是用`Subscribe`，射後不理。

```csharp
button.OnClickAsAsyncEnumerable().Subscribe(async x =>
{
    await UniTask.Delay(TimeSpan.FromSeconds(3));
});
```

Channel
---
（譯注：下面我也有點看不太懂。我盡力翻譯，但留下原文。我對異步處理的知識實在有所缺陷⋯⋯）

`Channel`跟[System.Threading.Tasks.Channels](https://docs.microsoft.com/ja-jp/dotnet/api/system.threading.channels?view=netcore-3.1)一樣，他們跟GoLang Channel有點類似。

`Channel` is same as [System.Threading.Tasks.Channels]https://docs.microsoft.com/ja-jp/dotnet/api/system.threading.channels?view=netcore-3.1) that is similar as GoLang Channel.

目前只支持multiple-producer, single-consumer unbounded channel。可以被`Channel.CreateSingleConsumerUnbounded<T>()`製造。

Currently only supports multiple-producer, single-consumer unbounded channel. It can create by Channel.CreateSingleConsumerUnbounded<T>().

對製造者來說(`.Writer`), 使用`TryWrite`來推取值（push value）以及使用`TryComplete`來結束channel。對消費者來說(`.Reader`)，用`TryRead`，`WaitToReadAsync`， `ReadAsync`，`Completion`還有`ReadAllAsync`來讀取已被派列的訊息。

For producer(`.Writer`), `TryWrite` to push value and TryComplete to complete channel. For consumer(`.Reader`), `TryRead`, `WaitToReadAsync`, `ReadAsync`, `Completion` and `ReadAllAsync` to read queued messages.

`ReadAllAsync`會回傳`IUniTaskAsyncEnumerable<T>`，所以可以使用query LINQ operators。讀取者只能允許單一一個消費者，但是你可以用`.Publish()` query operator來multicast訊息。比如說，實作pub/sub功能。

ReadAllAsync returns IUniTaskAsyncEnumerable<T> so query LINQ operators. Reader only allows single-consumer but use .Publish() query operator to enable multicast message. For example, make pub/sub utility.

```csharp
public class AsyncMessageBroker<T> : IDisposable
{
    Channel<T> channel;

    IConnectableUniTaskAsyncEnumerable<T> multicastSource;
    IDisposable connection;

    public AsyncMessageBroker()
    {
        channel = Channel.CreateSingleConsumerUnbounded<T>();
        multicastSource = channel.Reader.ReadAllAsync().Publish();
        connection = multicastSource.Connect(); // Publish returns IConnectableUniTaskAsyncEnumerable.
    }

    public void Publish(T value)
    {
        channel.Writer.TryWrite(value);
    }

    public IUniTaskAsyncEnumerable<T> Subscribe()
    {
        return multicastSource;
    }

    public void Dispose()
    {
        channel.Writer.TryComplete();
        connection.Dispose();
    }
}
```

For Unit Testing
---
Unity的`[UnityTest]`屬性可以拿來測試coroutine(IEnumerator)但是無法測試異步處理。`UniTask.ToCoroutine`是可以將async/await與coroutine連結的橋樑，讓你可以測試異步方法。

```csharp
[UnityTest]
public IEnumerator DelayIgnore() => UniTask.ToCoroutine(async () =>
{
    var time = Time.realtimeSinceStartup;

    Time.timeScale = 0.5f;
    try
    {
        await UniTask.Delay(TimeSpan.FromSeconds(3), ignoreTimeScale: true);

        var elapsed = Time.realtimeSinceStartup - time;
        Assert.AreEqual(3, (int)Math.Round(TimeSpan.FromSeconds(elapsed).TotalSeconds, MidpointRounding.ToEven));
    }
    finally
    {
        Time.timeScale = 1.0f;
    }
});
```

UniTask本身的單位測試是用Unity Test Runner還有[Cysharp/RuntimeUnitTestToolkit](https://github.com/Cysharp/RuntimeUnitTestToolkit)寫成的。他被確認CI還有IL2CPP都能運作。

ThreadPool limitation
---
大部分的UniTask方法都是單線程的（玩家迴圈），但是只有`UniTask.Run`跟`UniTask.SwitchToThreadPool`會在線程池（thread pool）上運作。如果你使用線程池的話，他不會在WebGL上運作。

`UniTask.Run`會在未來被廢棄（附上Obsolete屬性），只有`RunOnThreadPool`會沿用。以及，如果你用了`UniTask.Run`的話，想想看你是否能用`UniTask.Create`或是`UniTask.Void`。

IEnumerator.ToUniTask limitation
---
你可以將coroutine(IEnumerator)轉換為UniTask(或是直接await)，但是這有一些限制。

* `WaitForEndOfFrame`/`WaitForFixedUpdate`/`Coroutine`沒有被支持。
* 執行的時間跟StartCoroutine不一樣。他會在被指定的玩家迴圈時間被執行，預設的`PlayerLoopTiming.Update`會在MonoBehaviour的Update以及StartCoroutine的迴圈前被執行。

如果你想要將coroutine完全變成async的話，用`IEnumerator.ToUniTask(MonoBehaviour coroutineRunner)`這個超載方法。他會在為引數的MonoBehaviour實例上執行StartCoroutine，等待在UniTask中結束。

For UnityEditor
---
UniTask可以在Unity Editor上運行，跟Editor Coroutine一樣。但是，依然有一些限制。

* Delay, DelayFrame不會正確運行，因為delaTime沒辦法在editor裡拿到。await時，他會馬上回傳值。你可以用`DelayType.Realtime`來等候正確的時間。
* 所以的玩家迴圈時間都會在`EditorApplication.update`上執行

Compare with Standard Task API
---
UniTask有很多跟標準Task很像的API。下列的列表顯示了那些API是哪些的替代。

Use standard type.

| .NET Type | UniTask Type |
| --- | --- |
| `IProgress<T>` | --- |
| `CancellationToken` | --- |
| `CancellationTokenSource` | --- |

Use UniTask type.

| .NET Type | UniTask Type |
| --- | --- |
| `Task`/`ValueTask` | `UniTask` |
| `Task<T>`/`ValueTask<T>` | `UniTask<T>` |
| `async void` | `async UniTaskVoid` |
| `+= async () => { }` | `UniTask.Void`, `UniTask.Action`, `UniTask.UnityAction` |
| --- | `UniTaskCompletionSource` |
| `TaskCompletionSource<T>` | `UniTaskCompletionSource<T>`/`AutoResetUniTaskCompletionSource<T>` |
| `ManualResetValueTaskSourceCore<T>` | `UniTaskCompletionSourceCore<T>` |
| `IValueTaskSource` | `IUniTaskSource` |
| `IValueTaskSource<T>` | `IUniTaskSource<T>` |
| `ValueTask.IsCompleted` | `UniTask.Status.IsCompleted()` |
| `ValueTask<T>.IsCompleted` | `UniTask<T>.Status.IsCompleted()` |
| `new Progress<T>` | `Progress.Create<T>` |
| `CancellationToken.Register(UnsafeRegister)` | `CancellationToken.RegisterWithoutCaptureExecutionContext` |
| `CancellationTokenSource.CancelAfter` | `CancellationTokenSource.CancelAfterSlim` |
| `Channel.CreateUnbounded<T>(false){ SingleReader = true }` | `Channel.CreateSingleConsumerUnbounded<T>` |
| `IAsyncEnumerable<T>` | `IUniTaskAsyncEnumerable<T>` |
| `IAsyncEnumerator<T>` | `IUniTaskAsyncEnumerator<T>` |
| `IAsyncDisposable` | `IUniTaskAsyncDisposable` |
| `Task.Delay` | `UniTask.Delay` |
| `Task.Yield` | `UniTask.Yield` |
| `Task.Run` | `UniTask.Run` |
| `Task.WhenAll` | `UniTask.WhenAll` |
| `Task.WhenAny` | `UniTask.WhenAny` |
| `Task.CompletedTask` | `UniTask.CompletedTask` |
| `Task.FromException` | `UniTask.FromException` |
| `Task.FromResult` | `UniTask.FromResult` |
| `Task.FromCanceled` | `UniTask.FromCanceled` |
| `Task.ContinueWith` | `UniTask.ContinueWith` |
| `TaskScheduler.UnobservedTaskException` | `UniTaskScheduler.UnobservedTaskException` |

Pooling Configuration
---
UniTask會積極地將所有的異步物件進行快取，來達成零記憶體分配（技術細節，見此部落格 [UniTask v2 — Zero Allocation async/await for Unity, with Asynchronous LINQ](https://medium.com/@neuecc/unitask-v2-zero-allocation-async-await-for-unity-with-asynchronous-linq-1aa9c96aa7dd))。在預設中，他會快取所有的promises，但是你可以調整`TaskPool.SetMaxPoolSize`到你想要的數值。這個數值表示了每個類型的緩衝大小。`TaskPool.GetCacheSizeInfo`會回傳所以在池內的緩衝物件。

```csharp
foreach (var (type, size) in TaskPool.GetCacheSizeInfo())
{
    Debug.Log(type + ":" + size);
}
```

Allocation on Profiler
---
在UnityEditor profiler中，會顯示compiler製造的AsyncStateMachine的記憶體分配，但他只會在debug(development)build中顯示。在Debug模式中，C# Compiler會把AsyncStateMachine生產為類別（class），但是在Release模式中，則會生產為結構（struct）。

在Unity 2020.1之後，支持編碼最佳化的選項在UnityEditor之中（右邊, 葉腳)。

![](https://user-images.githubusercontent.com/46207/89967342-2f944600-dc8c-11ea-99fc-0b74527a16f6.png)

你可以將C# Compiler最佳化設置為Release，這會移除AsyncStateMachine的記憶體分配。還有，這個最佳化選項也可以通過`Compilation.CompilationPipeline-codeOptimization`以及`Compilation.CodeOptimization`設置。

UniTaskSynchronizationContext
---
Unity預設的同步處理域(`UnitySynchronizationContext`)是一個對效能來說糟糕的植入。UniTasl本身繞過了`SynchronizationContext`(還有`ExecutionContext`)，但是如果UniTask存在於`async Task`中，他還是會使用`SynchronizationContext`。`UniTaskSynchronizationContext`可以取代`UnitySynchronizationContext`。他對效能來說比較好。

```csharp
public class SyncContextInjecter
{
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    public static void Inject()
    {
        SynchronizationContext.SetSynchronizationContext(new UniTaskSynchronizationContext());
    }
}
```

這是一個選項，但不總是被推薦。`UniTaskSynchronizationContext`比`async UniTask`效能低，而且不是一個完整可取代UniTask的東西。他也無法保證跟`UnitySynchronizationContext`行為相符（behavioral compatible）。

API References
---
UniTask的API索引在[cysharp.github.io/UniTask](https://cysharp.github.io/UniTask/api/Cysharp.Threading.Tasks.html) by [DocFX](https://dotnet.github.io/docfx/)跟Cysharp/DocfXTemplate](https://github.com/Cysharp/DocfxTemplate)上。

舉例來說，UniTask的工廠方法（factory method）可以在[UniTask#methods](https://cysharp.github.io/UniTask/api/Cysharp.Threading.Tasks.UniTask.html#methods-1)上查詢。 UniTaskAsyncEnumerable的工廠/擴充方法則是在[UniTaskAsyncEnumerable#methods](https://cysharp.github.io/UniTask/api/Cysharp.Threading.Tasks.Linq.UniTaskAsyncEnumerable.html#methods-1)上。

UPM Package
---
### Install via git URL

在Unity 2019.3.4f1，Unity 2020.1a21後，支持path query parameter of git package。你可以直接在Package Manager上添加`https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask`。

![image](https://user-images.githubusercontent.com/46207/79450714-3aadd100-8020-11ea-8aae-b8d87fc4d7be.png)

![image](https://user-images.githubusercontent.com/46207/83702872-e0f17c80-a648-11ea-8183-7469dcd4f810.png)

或在`Packages/manifest.json`上添加`"com.cysharp.unitask": "https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask"`

如果你想要指定版本，UniTask使用了`*.*.*`的發布標籤，所以你可以指定版本，像是`#2.1.0`。例：`https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask#2.1.0`。

### Install via OpenUPM

這個Package也在[openupm registry](https://openupm.com)上提供。我們也推薦通過[openupm-cli](https://github.com/openupm/openupm-cli)下載。

```
openupm add com.cysharp.unitask
```

.NET Core
---
For .NET Core, use NuGet.（譯注：這句我就⋯⋯不翻譯了⋯⋯）

> PM> Install-Package [UniTask](https://www.nuget.org/packages/UniTask)

UniTask的.Net Core版本是Unity UniTask的子集（subset），移除了基於玩家迴圈的方法。

他會比標準的Task/ValueTask更有效率，但是在使用時你應該小心忽略掉ExecutionContext/SynchronizationContext。`AysncLocal`也無法運作，因為他無視了ExecutionContext.

如果你想要在內部使用UniTask，但是將ValueTask提供給外部的API，你可以寫出類似下列的範例（由[PooledAwait](https://github.com/mgravell/PooledAwait)啟發)。

```csharp
public class ZeroAllocAsyncAwaitInDotNetCore
{
    public ValueTask<int> DoAsync(int x, int y)
    {
        return Core(this, x, y);

        static async UniTask<int> Core(ZeroAllocAsyncAwaitInDotNetCore self, int x, int y)
        {
            // do anything...
            await Task.Delay(TimeSpan.FromSeconds(x + y));
            await UniTask.Yield();

            return 10;
        }
    }
}

// UniTask does not return to original SynchronizationContext but you can use helper `ReturnToCurrentSynchronizationContext`.
public ValueTask TestAsync()
{
    await using (UniTask.ReturnToCurrentSynchronizationContext())
    {
        await UniTask.SwitchToThreadPool();
        // do anything..
    }
}
```

.Net Core的版本是為了讓使用者在與Unity共享碼時將UniTask當作Interface使用（像是[Cysharp/MagicOnion](https://github.com/Cysharp/MagicOnion/)). .NET Core版本的碼將流暢的分享碼的機制化為可能。

功能性的方法，像是跟UniTask相等的WhenAll在[Cysharp/ValueTaskSupplement](https://github.com/Cysharp/ValueTaskSupplement)被提供。

License
---
這個資料庫被登記在MIT License之下。
