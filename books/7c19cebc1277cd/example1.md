---
title: "[PlayFab + UniTask] OnApplicationQuitAsync の実装例"
---

　ゲームを終了した際に、終了した時刻を PlayFab のプレイヤーデータに保持したいと思っていました。
MonoBehaviour にある OnApplicationQuit メソッドを利用したのですが、サーバー間のやり取りを行っている間、
処理が終わるまで待ってくれる訳もなく、失敗。
調べていたところ、UniTask を使うことで OnApplicationQuit メソッドにおいて、非同期処理が行える便利な機能を見つけました。

　ただし実装例を発見できなかったため、備忘録としてこちらに残しておこうと思います。

:::message
Unity : 2020.3
サーバー(Baas) : Azure PlayFab
ライブラリ : PlayFab C# SDK, UniTask
:::

　PlayFab の SDK については、非同期処理を書きやすくする(async / await 利用の)ため、
PlayFab C# SDK を利用しています。

-----

```CS
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Triggers;
using PlayFab;
using PlayFab.ClientModels;
using System;
using System.Collections.Generic;
using UnityEngine;


public class OnlineTimeManager : MonoBehaviour
{
    private const string FORMAT = "yyyy/MM/dd HH:mm:ss";

    private async UniTaskVoid Start() {

        var ct = this.GetCancellationTokenOnDestroy();

        await this.GetAsyncApplicationQuitTrigger().OnApplicationQuitAsync(ct);
    }

    /// <summary>
    /// ゲームが終了したときに自動的に呼ばれる
    /// </summary>
    private async UniTask OnApplicationQuit() {　　//　async UniTask に変更

        // PlayFab にゲーム終了時の時間を保存
        await UpdateLogOffTimeAsync();
  
        // 複数の await がある場合には WhenAll で対応
        
        Debug.Log("ゲームを終了します。");
    }

    /// <summary>
    /// OnApplicationQuit 時に実行するメソッド。今回はログオフ時間をサーバーにセーブ
    /// </summary>
    /// <returns></returns>
    public static async UniTask<bool> UpdateLogOffTimeAsync() {

        string dateTimeString = DateTime.Now.ToString(FORMAT);

        var request = new UpdateUserDataRequest {

            Data = new Dictionary<string, string> { { "LogOffTime", dateTimeString } }
        };

        var response = await PlayFabClientAPI.UpdateUserDataAsync(request);

        if (response.Error!= null) {
            Debug.Log("エラー");

            // 要エラーハンドリング

            return false;
        }

        Debug.Log("ログオフ時の時刻セーブ完了");
        return true;
    }
}
```

メソッドは３つです。

Start メソッド内で UniTask.Triggers に含まれる GetAsyncApplicationQuitTrigger().OnApplicationQuitAsync() メソッドを実行します。
このメソッドを実行することにより、MonoBehaviour にある OnApplicationQuit() メソッドにおいて
非同期処理が実装出来るようになります。

OnApplicationQuitAsync() メソッドの引数には、GetCancellationTokenOnDestroy() メソッドを登録し、
ゲームオブジェクトが破壊された場合に、このトリガー機能も終了するようにしています。

元々 MonoBehaviour にある OnApplicationQuit() メソッドには aysnc を追加して、戻り値を UniTask に変更します。
await を利用して非同期メソッドを実行することで、非同期処理が終了するまで処理を待機できますので、
ゲーム内の情報をサーバー(PlayFab)にセーブしてから、OnApplicationQuit() メソッドが終了するように制御出来るようになります。

UpdateLogOffTimeAsync() メソッドには bool 型の戻り値を設定していますが、この例では利用していません。
await 時の戻り値として利用することが出来ますので、処理結果を受けた分岐などを作る際に使ってください。

メソッド内の処理の内容自体は、日時の情報を文字列に置き換えて、それを PlayFab のプレイヤーデータ内に更新するものです。
この処理自体はいつもの PlayFab とのやりとりになりますので、特別なことはしていません。

-----

スクリプトが完成したら、ゲームを実行するための準備を行います。

ヒエラルキーの空いている場所で右クリックをしてメニューを開いて、Create Empty を選択するか、
ctrl + shift + N を押して、新しいゲームオブジェクトをヒエラルキーに作成します。
名称は任意です。(ここではスクリプト名と同じ名称にしています)

作成した OnlineTimeManager スクリプトをゲームオブジェクトにアタッチして、ゲームを実行します。

新しいコンポーネントが２つ、OnlineTimeMaanger に自動的にアタッチされます。
このうち、AsyncApplicationQuitTrigger がアタッチされていれば、
OnApplicationQuit メソッドが非同期処理で実行されるようになります。

![altテキスト](https://i.gyazo.com/ffdb26cd2f923f2d9e745325d157d0ab.png)

-----

あとは、ゲームを終了するだけです。Debug.Log メソッドにより、Console ビューに処理の流れが確認できますので、
非同期処理が待機されているかどうかを確認しておきます。

![altテキスト](https://i.gyazo.com/4779180a5a417d56ba6ccf0533d1986d.png)

また、PlayFab のゲームマネージャー画面も確認して、プレイヤーデータ(タイトル)が更新されているかを確認しておきます。

![altテキスト](https://i.gyazo.com/48dbfcc78e1a906fd5c214e8b1ac850c.png)


以上になります。