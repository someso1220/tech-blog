# SemaphoreSlimってなに
以前[SNNPマネージャーのサンプル](https://zenn.dev/someso/articles/152b1eb5a030cd)を作成しました。
しかし、あとから見直してみると多くの機器に対して同時にコマンドを送信すると
ネットワークがパンクしてしまう懸念がありました。
そこで今回はSemaphoreSlimを用いて上記の問題を解消したので
SemaphoreSlimについてと実際の解決方法をまとめていきます。

# SemaphoreSlimとは
簡単に言うと同時に実行できるタスクを制限してくれるものです

# SemaphoreとSemaphoreSlimの違い
チャッピー君にまとめてもらいました。
Semaphoreは非同期には対応しておらず、OSに依頼を出すので処理が重いみたいです。
SemaphoreSlimの方は軽量で非同期にも対応しています。
| 項目 | Semaphore | SemaphoreSlim |
|------|-----------|---------------|
| OS リソース | **使う（カーネルオブジェクト）** | **使わない（完全に .NET 管理）** |
| 重さ | **重い（OS 呼び出しが発生）** | **軽い（高速）** |
| Wait | **同期のみ** | **同期 + 非同期（WaitAsync）** |
| Release | どちらもある | どちらもある |
| 用途 | プロセス間同期（複数プロセス） | スレッド/タスク間同期（同一プロセス） |
| 非同期処理 | **できない** | **できる（await できる）** |

詳しい内容は下記を
https://learn.microsoft.com/ja-jp/dotnet/standard/threading/semaphore-and-semaphoreslim
# 使い方
使い方は下記になります。
今回は非同期処理なのでsemaphore.WaitAsyncを使用しています。
同期処理の場合はsemaphore.Wait()を使用します。
注意点としてはタスクが完了していない状態でRelease()を行うと
並列数制限を超えてタスクを実行してしまいます。
ですのでfinallyなどでタスクが完了したタイミングで
Release()を行う必要があります。
```
// 実行できるタスクの最大を設定する
SemaphoreSlim semaphore = new SemaphoreSlim(1);
// タスクを実行する
Task a =  Task.Run(async () => 
{
    // タスクを実行時にWaitAsyncを呼ぶ
    // WaitAsyncを呼ぶと実行できるタスク数を1減らす
    await semaphore.WaitAsync();
    try
    {
        Console.WriteLine("タスクA");
        // 時間がかかる処理
        await Task.Delay(1000);
    }
    finally
    {
        // タスクが終了した分の実行できるタスク数を1回復させる
        semaphore.Release();
    }

});

```
# 解決方法
使い方で記載したサンプルコードをベースに機器を50台づつ処理するよう変更しました。
変更箇所以外のコードは下記の記事をご覧ください。
https://zenn.dev/someso/articles/152b1eb5a030cd
```

partial class Program
{
    static ConcurrentQueue<string> _logQueue = new ConcurrentQueue<string>();

    static async Task Main()
    {
        Program program = new Program();
        RootConfig? configs = new RootConfig();

        try
        {
            // キャンセルトークンの取得
            var cancellationToken = new CancellationTokenSource();

            // Enterキーが押されるまで待機
            _ = Task.Run(() =>
            {
                Console.ReadLine();
                cancellationToken.Cancel();
            });

            // ログ出力
            Task logTask = Task.Run(() => ProcessLogsAsync(cancellationToken.Token));

            if (!program.ReadJson(out configs))
            {
                _logQueue.Enqueue("[Error]JSONファイルの読込に失敗しました");
                return;
            }

            if(configs == null)
            {
                _logQueue.Enqueue("[Error]JSONファイルの読込に失敗しました");
                return;
            }

            // 50台づつ処理を行う
            SemaphoreSlim semaphore = new SemaphoreSlim(50);

            while (!cancellationToken.Token.IsCancellationRequested)
            {
                // 以前のコード
                // await Task.WhenAll(configs.Devices.Select(d => program.SendRequestAsync(d)));

                var sendTask = configs.Devices.Select(async d =>
                {
                    await semaphore.WaitAsync();
                    try
                    {
                        await program.SendRequestAsync(d);
                    }
                    finally
                    {
                        semaphore.Release();
                    }
                });
                await Task.WhenAll(sendTask);
                await Task.Delay(TimeSpan.FromSeconds(10), cancellationToken.Token); // 10秒待つ
                
            }

            await logTask;
        }
        // 非同期処理エラー
        catch (OperationCanceledException)
        {
            _logQueue.Enqueue("[Error]キャンセルされました。終了します。");
        }
        // その他エラー
        catch (Exception ex)
        {
            _logQueue.Enqueue("[Error]"+ex.ToString());
        }
    }
}
```

# 参考にしたサイト
https://learn.microsoft.com/ja-jp/dotnet/api/system.threading.semaphoreslim?view=net-8.0
https://qiita.com/TsuyoshiUshio@github/items/79ad787899cddaa3ac1c
https://qiita.com/october/items/79f470653c96d65cef19