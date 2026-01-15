# SNMPマネージャーのサンプル作った
最近の業務でお客様から使用している機器の接続状態を監視したいという要望がありました。
そこでサンプルを作成してお客様に提示することになりました。

# 使用したライブラリ
SharpSnmpLib


# 実際のコード
SharpSnmpLibのサンプルコードをベースに作成しています。
そこにプラスしてログの表示と出力を行っています。
また非同期で処理するようにサンプルとは違うメソッドを使用しています。

```
using Lextm.SharpSnmpLib;
using Lextm.SharpSnmpLib.Messaging;
using Lextm.SharpSnmpLib.Security;
using Mono.Options;
using SNMPSample;
using System.Collections.Concurrent;
using System.Net;
using System.Net.Sockets;
using System.Reflection;
using System.Text.Json;


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

            while (!cancellationToken.Token.IsCancellationRequested)
            {
                Console.WriteLine($"[Info] 開始: {DateTime.Now}");
                await Task.WhenAll(configs.Devices.Select(d => program.SendRequestAsync(d)));
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

    /// <summary>
    /// リクエスト送信
    /// </summary>
    /// <param name="config"></param>
    public async Task SendRequestAsync(DeviceConfig config)
    {
        // デフォルト値の初期化
        string community = config.Community;        // SNMP v1/v2c のコミュニティ文字列
        bool showVersion = false;           // バージョン表示フラグ
        VersionCode version = config.Version; // SNMP バージョン (デフォルト v1)
        int timeout = 10000;                 // タイムアウト (ms)
        Levels level = config.SecurityLevel;   // セキュリティレベル (デフォルト noAuthNoPriv)
        string user = config.UserName;         // SNMPv3 ユーザー名
        string contextName = config.ContextName;  // SNMPv3 コンテキスト名
        string authentication = config.AuthProtocol; // 認証方式 (MD5, SHA, etc.)
        string authPhrase = config.AuthPassword;   // 認証パスフレーズ
        string privacy = config.PrivProtocol;      // 暗号化方式 (DES, AES, etc.)
        string privPhrase = config.PrivPassword;   // 暗号化パスフレーズ

        // バージョン表示指定時
        if (showVersion)
        {
            var entryAssembly = Assembly.GetEntryAssembly();
            var versionAttr = entryAssembly?.GetCustomAttribute<AssemblyVersionAttribute>();
            string versionStr = versionAttr?.Version ?? "Version not found";

            _logQueue.Enqueue(versionStr);
            return;
        }

        // ホスト名/IP の解析
        IPAddress? ip;
        int port = 161; // SNMP デフォルトポート
        string hostNameOrAddress = config.Ip;

        // host:port 形式対応
        if (hostNameOrAddress.Contains(':'))
        {
            string[] parts = hostNameOrAddress.Split(':');
            if (parts.Length == 2 && int.TryParse(parts[1], out int parsedPort))
            {
                hostNameOrAddress = parts[0];
                port = parsedPort;
            }
        }

        // IPアドレス or ホスト名解決
        bool parsed = IPAddress.TryParse(hostNameOrAddress, out ip);
        if (!parsed)
        {
            IPAddress[] addresses = await Dns.GetHostAddressesAsync(hostNameOrAddress);
            ip = addresses.FirstOrDefault(a => a.AddressFamily == AddressFamily.InterNetwork);

            if (ip == null)
            {
                _logQueue.Enqueue("invalid host or wrong IP address found: " + hostNameOrAddress);
                return;
            }
        }

        try
        {
            // OID を変数リストに変換
            List<Variable> vList = new List<Variable>();
            for (int i = 0; i < config.Oids.Length; i++)
            {
                vList.Add(new Variable(new ObjectIdentifier(config.Oids[i])));
            }

            IPEndPoint receiver = new IPEndPoint(ip, port);

            // SNMP v1/v2c の場合
            if (version != VersionCode.V3)
            {
                // ここでGETリクエストを送信(これを複数)
                using (CancellationTokenSource cancellationToken = new CancellationTokenSource(TimeSpan.FromMilliseconds(timeout)))
                {
                    var variables = await Messenger.GetAsync(version, receiver, new OctetString(community), vList, cancellationToken.Token);

                    foreach (Variable variable in variables)
                    {
                        DisplayLog(variable, config.Name);
                    }
                }
                return;
            }

            // SNMP v3 の場合
            if (string.IsNullOrEmpty(user))
            {
                _logQueue.Enqueue("User name need to be specified for v3.");
                return;
            }

            // 認証プロバイダの設定
            IAuthenticationProvider auth = (level & Levels.Authentication) == Levels.Authentication
                ? GetAuthenticationProviderByName(authentication, authPhrase)
                : DefaultAuthenticationProvider.Instance;

            // 暗号化プロバイダの設定
            IPrivacyProvider priv = (level & Levels.Privacy) == Levels.Privacy
                ? GetPrivacyProviderByName(privacy, privPhrase, auth)
                : new DefaultPrivacyProvider(auth);

            // SNMPv3 のディスカバリ (タイムウィンドウ同期用)
            // SNMPv3 は「タイムウィンドウ同期」が必要
            Discovery discovery = Messenger.GetNextDiscovery(SnmpType.GetRequestPdu);
            ReportMessage report = await discovery.GetResponseAsync( receiver);

            // GET リクエスト生成
            GetRequestMessage request = new GetRequestMessage(
                VersionCode.V3,
                Messenger.NextMessageId,
                Messenger.NextRequestId,
                new OctetString(user),
                new OctetString(string.IsNullOrWhiteSpace(contextName) ? string.Empty : contextName),
                vList,
                priv,
                Messenger.MaxMessageSize,
                report);

            // 応答取得
            using (CancellationTokenSource cancellationToken = new CancellationTokenSource(TimeSpan.FromMilliseconds(timeout)))
            {
                ISnmpMessage reply = await request.GetResponseAsync(receiver, cancellationToken.Token);

                // レポートメッセージ処理 (RFC 3414 に従い再送)
                // SNMPv3 では「NotInTimeWindow」というエラーがよく返る
                // RFC 3414 に従って、再度リクエストを送って時間を同期する
                if (reply is ReportMessage)
                {
                    if (reply.Pdu().Variables.Count == 0)
                    {
                        _logQueue.Enqueue("wrong report message received");
                        return;
                    }

                    var id = reply.Pdu().Variables[0].Id;
                    if (id != Messenger.NotInTimeWindow)
                    {
                        _logQueue.Enqueue(id.GetErrorMessage());
                        return;
                    }

                    // タイムウィンドウ同期のため再リクエスト
                    request = new GetRequestMessage(
                        VersionCode.V3,
                        Messenger.NextMessageId,
                        Messenger.NextRequestId,
                        new OctetString(user),
                        new OctetString(string.IsNullOrWhiteSpace(contextName) ? string.Empty : contextName),
                        vList,
                        priv,
                        Messenger.MaxMessageSize,
                        reply);
                    reply = await request.GetResponseAsync(receiver, cancellationToken.Token);
                }
                else if (reply.Pdu().ErrorStatus.ToInt32() != 0)
                {
                    // エラー応答処理
                    throw ErrorException.Create("error in response", receiver.Address, reply);
                }

                // 取得した変数を表示
                foreach (Variable v in reply.Pdu().Variables)
                {
                    DisplayLog(v,config.Name);
                }
            }
        }
        // SNMPエラー
        catch (SnmpException ex)
        {
            _logQueue.Enqueue($"[Error] {config.Name} { ex.ToString()}");
        }
        // ソケットエラー
        catch (SocketException ex)
        {
            _logQueue.Enqueue($"[Error] {config.Name} {ex.ToString()}");
        }
        // 非同期処理エラー
        catch (OperationCanceledException ex)
        {
            _logQueue.Enqueue($"[Error] {config.Name} {ex.ToString()}");
        }
        // その他エラー
        catch (Exception ex)
        {
            _logQueue.Enqueue($"[Error] {config.Name} {ex.ToString()}");
        }
    }

    /// <summary>
    /// プライバシープロバイダ（暗号化方式）を名前から生成
    /// </summary>
    /// <param name="privacy"></param>
    /// <param name="phrase"></param>
    /// <param name="auth"></param>
    /// <returns></returns>
    /// <exception cref="ArgumentException"></exception>
    private static IPrivacyProvider GetPrivacyProviderByName(string privacy, string phrase, IAuthenticationProvider auth)
    {
        // privacy が指定されていない場合は「暗号化なし」で返す
        if (string.IsNullOrEmpty(privacy))
        {
            return new DefaultPrivacyProvider(auth);
        }

        // privacy の文字列を大文字にして判定
        switch (privacy.ToUpperInvariant())
        {
            //case "DES":
            //    // DES がサポートされていれば DES プロバイダを返す
            //    if (DESPrivacyProvider.IsSupported)
            //    {
            //        return new DESPrivacyProvider(new OctetString(phrase), auth);
            //    }
            //    // サポートされていなければ例外
            //    throw new ArgumentException("DES privacy is not supported in this system");

            //case "3DES":
            //    // 3DES プロバイダを返す
            //    return new TripleDESPrivacyProvider(new OctetString(phrase), auth);

            case "AES":
                // AES がサポートされていれば AES プロバイダを返す
                if (AESPrivacyProvider.IsSupported)
                {
                    return new AESPrivacyProvider(new OctetString(phrase), auth);
                }
                throw new ArgumentException("AES privacy is not supported in this system");

            case "AES192":
                // AES192 がサポートされていれば AES192 プロバイダを返す
                if (AESPrivacyProvider.IsSupported)
                {
                    return new AES192PrivacyProvider(new OctetString(phrase), auth);
                }
                throw new ArgumentException("AES192 privacy is not supported in this system");

            case "AES256":
                // AES256 がサポートされていれば AES256 プロバイダを返す
                if (AESPrivacyProvider.IsSupported)
                {
                    return new AES256PrivacyProvider(new OctetString(phrase), auth);
                }
                throw new ArgumentException("AES256 privacy is not supported in this system");

            default:
                // 未知の暗号化方式が指定された場合は例外
                throw new ArgumentException("unknown privacy name: " + privacy);
        }
    }

    /// <summary>
    /// 認証方式の名前から適切な認証プロバイダを生成
    /// </summary>
    /// <param name="authentication"></param>
    /// <param name="phrase"></param>
    /// <returns></returns>
    /// <exception cref="ArgumentException"></exception>
    private static IAuthenticationProvider GetAuthenticationProviderByName(string authentication, string phrase)
    {
        // SHA-256 認証
        if (authentication.ToUpperInvariant() == "SHA256")
        {
            return new SHA256AuthenticationProvider(new OctetString(phrase));
        }

        // SHA-384 認証
        if (authentication.ToUpperInvariant() == "SHA384")
        {
            return new SHA384AuthenticationProvider(new OctetString(phrase));
        }

        // SHA-512 認証
        if (authentication.ToUpperInvariant() == "SHA512")
        {
            return new SHA512AuthenticationProvider(new OctetString(phrase));
        }

        // 未知の認証方式が指定された場合は例外を投げる
        throw new ArgumentException("unknown name", nameof(authentication));
    }

    /// <summary>
    /// Json読込
    /// </summary>
    /// <param name="device"></param>
    /// <returns></returns>
    private bool ReadJson(out RootConfig? device)
    {
        string path = "config.json";

        if (File.Exists(path))
        {
            string json = File.ReadAllText("config.json");
            device = JsonSerializer.Deserialize<RootConfig>(json);

            return true;
        }
        else
        {
            _logQueue.Enqueue("[Error]JSONファイルが見つかりません: " + path);
            device = new RootConfig();
            return false;
        }

    }

    /// <summary>
    /// ログ出力
    /// </summary>
    /// <param name="token"></param>
    /// <returns></returns>
    public static async Task ProcessLogsAsync(CancellationToken token)
    {
        string folderPath = @"C:\Log" + "\\" + string.Format(@"{0:yyyyMMdd}", DateTime.Now);
        string fileName = "Log.csv";

        if (!Directory.Exists(folderPath))
        {
            Directory.CreateDirectory(folderPath);
        }

        using (var writer = new StreamWriter(folderPath + "\\" + fileName, append: true))
        {
            while (!token.IsCancellationRequested)
            {
                while (_logQueue.TryDequeue(out var log))
                {
                    await writer.WriteLineAsync(log);   // CSVに追記
                    Console.WriteLine(log);             // コンソールにも出力
                }

                await writer.FlushAsync();
                await Task.Delay(500, token); // 少し待ってから次のループ
            }
        }
    }

    /// <summary>
    /// ログ表示
    /// </summary>
    /// <param name="variable"></param>
    private static void DisplayLog(Variable variable,string deviceName)
    {
        Dictionary<string, string> oidMessageMap = new Dictionary<string, string>
        {
            { "1.3.6.1.2.1.1.1.0", "システムの説明" },
            { "1.3.6.1.2.1.2.2.1.10.1", "受信バイト数 (ifInOctets.1)" },
            { "1.3.6.1.2.1.1.5.0", "ホスト名" },
            { "1.3.6.1.2.1.1.3.0", "稼働時間 (sysUpTime)" },
            { "1.3.6.1.2.1.1.6.0", "設置場所 (sysLocation)" },
            { "1.3.6.1.2.1.2.2.1.8.1", "インターフェース状態 (ifOperStatus.1)" }
        };

        string oid = variable.Id.ToString();
        string value = variable.Data.ToString();

        if (oidMessageMap.ContainsKey(oid))
        {
            // 稼働時間
            if (oid == "1.3.6.1.2.1.1.3.0")
            {
                // SNMPの返り値は centiseconds (100分の1秒)
                if (variable.Data is TimeTicks tt)
                {
                    // TimeTicks → TimeSpan に変換
                    TimeSpan uptime = tt.ToTimeSpan();

                    _logQueue.Enqueue($"[Info] {deviceName} {oidMessageMap[oid]}: {uptime.Days}日 {uptime.Hours}時間 {uptime.Minutes}分 {uptime.Seconds}秒");
                }
            }
            else if (oid == "1.3.6.1.2.1.2.2.1.8.1")
            {
                string state;
                switch (variable.Data.ToString())
                {
                    case "1":
                        state = "稼働中";
                        break;
                    case "2":
                        state = "停止中";
                        break;
                    case "3":
                        state = "テスト中";
                        break;
                    case "4":
                        state = "不明";
                        break;
                    case "5":
                        state = "休止状態";
                        break;
                    case "6":
                        state = "存在しない";
                        break;
                    case "7":
                        state = "下位レイヤーがダウン";
                        break;
                    default:
                        state = "";
                        break;
                }

                _logQueue.Enqueue($"[Info] {deviceName} {oidMessageMap[oid]}: {state}");
            }
            else
            {
                _logQueue.Enqueue($"[Info] {deviceName} {oidMessageMap[oid]}: {value}");
            }
        }
        else
        {
            _logQueue.Enqueue($"[Info] {deviceName} {oid}: {value}");
        }
    }
}

```

```
{
  "devices": [
    {
      "name": "PC",
      "ip": "",
      "community": "public",
      "version": 1,
      "oids": [
        "1.3.6.1.2.1.1.1.0",
        "1.3.6.1.2.1.2.2.1.10.1",
        "1.3.6.1.2.1.1.5.0",
        "1.3.6.1.2.1.1.3.0",
        "1.3.6.1.2.1.1.6.0",
        "1.3.6.1.2.1.2.2.1.8.1"
      ],
      "userName": "snmpuser",
      "contextName": "",
      "authProtocol": "SHA",
      "authPassword": "mypassword",
      "privProtocol": "AES",
      "privPassword": "mysecret",
      "securityLevel": 1
    },
    {
      "name": "Printer",
      "ip": "",
      "community": "public",
      "version": 1,
      "oids": [
        "1.3.6.1.2.1.1.1.0",
        "1.3.6.1.2.1.2.2.1.10.1",
        "1.3.6.1.2.1.1.5.0",
        "1.3.6.1.2.1.1.3.0",
        "1.3.6.1.2.1.1.6.0",
        "1.3.6.1.2.1.2.2.1.8.1",
        "1.3.6.1.2.1.43.11.1.1.6.1.1"
      ],
      "userName": "snmpuser",
      "contextName": "",
      "authProtocol": "SHA",
      "authPassword": "mypassword",
      "privProtocol": "AES",
      "privPassword": "mysecret",
      "securityLevel": 1
    },
    {
      "name": "Router",
      "ip": "",
      "community": "public",
      "version": 1,
      "oids": [
        "1.3.6.1.2.1.1.1.0",
        "1.3.6.1.2.1.2.2.1.10.1",
        "1.3.6.1.2.1.1.5.0",
        "1.3.6.1.2.1.1.3.0",
        "1.3.6.1.2.1.1.6.0",
        "1.3.6.1.2.1.2.2.1.8.1"
      ],
      "userName": "snmpuser",
      "contextName": "",
      "authProtocol": "SHA",
      "authPassword": "mypassword",
      "privProtocol": "AES",
      "privPassword": "mysecret",
      "securityLevel": 1
    }
  ]
}
```

```
using Lextm.SharpSnmpLib;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Text.Json.Serialization;
using System.Threading.Tasks;

namespace SNMPSample
{
    internal class RootConfig
    {
        [JsonPropertyName("devices")]
        public List<DeviceConfig> Devices { get; set; }
    }

    internal class DeviceConfig
    {
        /// <summary>
        /// 機器名
        /// </summary>
        [JsonPropertyName("name")]
        public string Name{ get; set; }

        /// <summary>
        /// IPアドレス
        /// </summary>
        [JsonPropertyName("ip")]
        public string Ip { get; set; }

        /// <summary>
        /// コミュニティ文字列
        /// </summary>
        [JsonPropertyName("community")]
        public string Community { get; set; }

        /// <summary>
        /// バージョン
        /// </summary>
        [JsonPropertyName("version")]
        public VersionCode Version { get; set; }

        /// <summary>
        /// OID
        /// </summary>
        [JsonPropertyName("oids")]
        public string[] Oids { get; set; }

        /// <summary>
        /// ユーザー名
        /// </summary>

        [JsonPropertyName("userName")]
        public string UserName {  get; set; }

        /// <summary>
        /// コンテキスト名
        /// </summary>
        [JsonPropertyName("contextName")]
        public string ContextName { get; set; }

        /// <summary>
        /// 認証方式
        /// </summary>
        [JsonPropertyName("authProtocol")]
        public string AuthProtocol { get; set; }

        /// <summary>
        /// 認証パスワード
        /// </summary>
        [JsonPropertyName("authPassword")]
        public string AuthPassword { get; set; }

        /// <summary>
        /// 暗号化方式
        /// </summary>
        [JsonPropertyName("privProtocol")]
        public string PrivProtocol { get; set; }

        /// <summary>
        /// 暗号化パスワード
        /// </summary>
        [JsonPropertyName("privPassword")]
        public string PrivPassword { get; set; }

        /// <summary>
        /// セキュリティレベル
        /// </summary>
        [JsonPropertyName("securityLevel")]
        public Levels SecurityLevel { get; set; }
    }
}

```