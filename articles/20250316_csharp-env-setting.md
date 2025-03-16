---
title: "デフォルト設定管理方法"
emoji: "🐈"
type: "tech"
topics: ["csharp", "windows"]
published: true
---

## はじめに

C# Visual Studio WPFでデスクトップアプリケーションを作成した際に学んだことです
他の言語にも共通すると思いますので、参考になれば幸いです

## 概要

以下のような要件がありました

- 設定ファイルが削除されてもデフォルト設定で動作させたい
- ビルド後で設定ファイルを書き換え、動作を変えられるようにしたい

## 失敗

最初は、C#コード内にデフォルト設定を直接記載していました
デフォルト設定を変える場合は、C#コードを書き換える必要がありました

- デフォルト設定: C#コード
- 上書き設定: 外部ファイル(.json等)

## 成功

正しくは、デフォルト設定と上書き設定が同じファイルで設定可能にすることですが、設定ファイルが削除された場合にデフォルト設定を適用させる方法が分かりませんでした

- デフォルト設定: 外部ファイル(.json等)
- 上書き設定: 外部ファイル(.json等)

C# Visual Studioでは、ファイルのプロパティで `ビルドアクション = 埋め込みリソース` にすることで、ビルド時にexeファイルにファイルの内容が埋め込まれて、ファイルが削除されても、ビルド時のファイルの内容が参照できるようになります!

設定ファイルが存在すれば、その設定ファイルの値を読み込むだけです

env.jsonファイルの例

![](/images/20250316_csharp-env-setting/1.png)

※`出力ディレクトリにコピー = 常にコピーする` を設定すると、exeファイルと同階層にファイルが出力されます
これを上書き用の設定ファイルにしました

## 実行例

コンソールアプリの例です

1. ビルド直後に、デフォルト設定の env.json ファイルがある状態でexeファイルを実行

![](/images/20250316_csharp-env-setting/2.png)
![](/images/20250316_csharp-env-setting/3.png)

2. env.json ファイルを書き換えて実行

![](/images/20250316_csharp-env-setting/4.png)
![](/images/20250316_csharp-env-setting/5.png)

3. 設定ファイルを削除して実行

![](/images/20250316_csharp-env-setting/6.png)
![](/images/20250316_csharp-env-setting/7.png)


## コード例

1. 埋め込みリソース名は、`名前空間.フォルダ名.ファイル名` のような形式

※埋め込みリソース名が分からない場合は、以下のコードで埋め込みリソース名を一覧取得可能

```csharp
String[] embeddingResourceNames = Assembly.GetExecutingAssembly().GetManifestResourceNames();
Console.WriteLine($"埋め込みリソース名一覧: {String.Join(Environment.NewLine, embeddingResourceNames)}");
```

2. 埋め込みリソースのデータ取得方法

```csharp
String envResourceName = "EnvConfigTest.env.json";
Stream resourceStream = Assembly.GetExecutingAssembly().GetManifestResourceStream(envResourceName)
```

3. 出力ディレクトリにコピーした外部ファイルのパス取得方法

```csharp
String exeFolderPath = AppDomain.CurrentDomain.BaseDirectory;
Console.WriteLine($"exeフォルダのパス: {exeFolderPath}");

String envPath = Path.Combine(exeFolderPath, "env.json");
Console.WriteLine($"env.jsonのパス: {envPath}");
```

全コード

```csharp
using System.Reflection;
using System.Text.Json;

namespace EnvConfigTest;

public static class JsonOptionsProvider
{
    public static readonly JsonSerializerOptions Options = new JsonSerializerOptions { WriteIndented = true };
}

class Program
{
    /// <summary>
    /// エントリーポイント
    /// </summary>
    static void Main()
    {
        try
        {
            // 埋め込みリソースからの設定を取得
            Person embeddingConfig = GetEmbeddingResourceData();
            Console.WriteLine();
            // 外部ファイルからの設定を取得
            Person? overrideConfig = GetOverrideData();
            Console.WriteLine();
            // 最終的な設定を取得
            Person finalConfig = overrideConfig ?? embeddingConfig;
            Console.WriteLine("最終的に使用される設定:");
            Console.WriteLine(finalConfig);
        }
        catch(JsonException ex)
        {
            Console.WriteLine($"JSONデシリアライズエラー: {ex}");
        }
        catch(Exception ex)
        {
            Console.WriteLine($"予期しないエラー: {ex}");
        }
    }


    /// <summary>
    /// 埋め込みリソースからデフォルト設定を取得する
    /// </summary>
    private static Person GetEmbeddingResourceData()
    {
        String[] embeddingResourceNames = Assembly.GetExecutingAssembly().GetManifestResourceNames();
        Console.WriteLine($"埋め込みリソース名一覧: {String.Join(Environment.NewLine, embeddingResourceNames)}");

        String envResourceName = "EnvConfigTest.env.json";
        Stream resourceStream = Assembly.GetExecutingAssembly().GetManifestResourceStream(envResourceName)
            ?? throw new InvalidOperationException($"埋め込みリソース {envResourceName} が見つかりません");

        using(resourceStream)
        {
            Person defaultConfig = JsonSerializer.Deserialize<Person>(resourceStream, JsonOptionsProvider.Options)
                ?? throw new InvalidOperationException("埋め込みリソースのデータのデシリアライズに失敗しました");

            Console.WriteLine("埋め込みリソースから取得したデータ:");
            Console.WriteLine(defaultConfig);
            return defaultConfig;
        }
    }


    /// <summary>
    /// 外部ファイルから設定を取得する
    /// </summary>
    private static Person? GetOverrideData()
    {
        String exeFolderPath = AppDomain.CurrentDomain.BaseDirectory;
        Console.WriteLine($"exeフォルダのパス: {exeFolderPath}");

        String envPath = Path.Combine(exeFolderPath, "env.json");
        Console.WriteLine($"env.jsonのパス: {envPath}");

        if(File.Exists(envPath))
        {
            Stream overrideStream = File.OpenRead(envPath);
            using(overrideStream)
            {
                Person overrideConfig = JsonSerializer.Deserialize<Person>(overrideStream, JsonOptionsProvider.Options)
                    ?? throw new InvalidOperationException("外部ファイルのデータのデシリアライズに失敗しました");

                Console.WriteLine("外部ファイルから取得したデータ:");
                Console.WriteLine(overrideConfig);
                return overrideConfig;
            }
        }
        else
        {
            Console.WriteLine($"外部ファイル {envPath} が見つかりません");
            return null;
        }
    }
}


class Person
{
    public String Name { get; set; } = String.Empty;
    public Int32 Age { get; set; } = -1;
    public String Email { get; set; } = String.Empty;

    public override String ToString() => JsonSerializer.Serialize(this, JsonOptionsProvider.Options);
}
```

## おわりに

運用チームのことを考えたり、インターネットに接続できない環境での開発など、実務でしか経験しないようなことに学びが多いですね！