---
title: "AIからも叩けるUnity用コマンドパレット「LiminalPalette」"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "ClaudeCode", "AI", "個人開発"]
published: false
---

## はじめに

Unity 開発中、「デバッグ用のチートコマンドを整備したい」「Claude Code などの AI Agent にゲームの動作確認まで任せたい」と思ったことはないでしょうか。

私は最近、コマンドパレット風 UI を持つ Unity 用のデバッグコンソール / コマンド実行ライブラリ **LiminalPalette** を開発し、活用しています。
ある程度運用が安定してきたので、LiminalPalette がどのようなものか、どのように活用しているかを紹介したいと思います。

@[card](https://github.com/void2610/liminal-palette)

## LiminalPalette とは

VS Code のコマンドパレット風 UI を持つ、Unity 用のデバッグコンソール / コマンド実行ライブラリです。

具体的には以下のことができます。

- ファジー検索付きコマンドパレット: `Cmd/Ctrl + K` で開く VS Code ライクな UI (Editor / Runtime 両対応、UI Toolkit 製)
- 型解決済み引数 UI: `int` / `float` / `string` / `bool` / `enum` / `Vector2/3/4` / `Color` / `[Flags] enum` / `UnityEngine.Object` 派生を自動で入力フォーム化
- コマンド列 + assert によるシナリオ: `[LiminalScenario]` で宣言し、UI / HTTP / CI から回帰テストとして実行
- HTTP API: localhost で `/api/v1/{health, commands, execute, logs, state, scenarios, scenarios/run}` を提供
- Production ビルド除外: asmdef の `defineConstraints` で Player ビルドにデバッグコードが混入しない
- Claude Code Skills 同梱: メニューから 1 クリックで AI Agent 連携をセットアップ

`[LiminalCommand]` 属性をつけるだけで、任意の C# メソッドを **Editor / Runtime / HTTP API の 3 経路から統一的に実行できる**ことが最大の特徴です。

単純なデバッグコンソールとして利用するだけでも便利ですが、HTTP API 経由で AI Agent (Claude Code 等) や CLI からコマンドを呼び出せるため、
AI Agent が自律的に実装の動作確認をする、複数のコマンドを組み合わせて E2E テストを実装するといったユースケースにも活用できます。

## 設計思想: デバッグ機能を使い捨てにしない

デバッグ機能の失敗には二通りあります。
一つは、そもそも残さないこと。再利用できる形で残すコストが高いためで、デバッグメニューの UI を組んで配線するくらいなら、と一時的な `if (Input.GetKeyDown(...))` / `Debug.Log(...)` に流れた経験のある方も多いのではないでしょうか。
もう一つは、残したのに使われないこと。書いた本人しか存在を知らないまま、ただの汚い検証コードとして腐っていきます。

LiminalPalette は次の3点でこの問題を解決します。

| 特徴 | 効果 |
| --- | --- |
| 属性 1 つで登録が完了する | 「ちゃんと残す」コストが「その場しのぎ」と同じになる |
| 登録した時点でパレットから検索できる | 書いた本人しか存在を知らない、という腐り方を防げる |
| 同じコマンドを機械からも叩ける | 人間が押すだけの機能で終わらず、テストの部品になる |

1 つ目だけでは、単に手軽なだけで使い捨てを量産する方向にも働きます。
コストが下がったうえで、残したものが最初から「人間の操作 UI」「AI の操作 API」「E2E テストの部品」を兼ねていることに意味があります。

この設計は、「日々の動作確認を `[LiminalScenario]` として資産化し、E2E 回帰テストを貯め続ける」という運用ルールと組み合わせたときに効いてきます。
実プロジェクトでその運用をどう回しているかは別記事にまとめているので、ライブラリの機能より運用のほうに興味がある方はそちらへどうぞ。

@[card](https://zenn.dev/void2610/articles/unity-e2e-scenario-26y07m20d)

## 基本の使い方

### 1. コマンドを書く

デバッグしたいメソッドに属性を付けるだけです。

```csharp:Player.cs
using R3;
using UnityEngine;
using Void2610.LiminalPalette;

public class Player : MonoBehaviour
{
    [LiminalObservableField("Player/Health")]
    public ReactiveProperty<int> Hp { get; } = new(100);  // 現在値が UI に常時表示される

    [LiminalCommand("Player/Health/Set", Description = "プレイヤーの HP を設定する")]
    public void SetHealth(int value) => Hp.Value = value;
}
```

`[LiminalObservableField]` を付けた R3 の `ReactiveProperty<T>` は、パレット UI に現在値が push 駆動で常時表示されます。
属性はプロパティとフィールドのどちらにも付けられ、`ReadOnlyReactiveProperty<T>` / `Observable<T>` も対象です (public メンバーであることが条件)。
「HP を書き換えるコマンド」と「現在の HP の監視」がワンセットで手に入るイメージです。

### 2. VContainer に登録

```csharp:GameLifetimeScope.cs
public class GameLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        builder.RegisterComponentInHierarchy<Player>();
        builder.RegisterEntryPoint<LiminalPaletteEntryPoint>();   // 1 行で接続完了
    }
}
```

インスタンスメソッドのコマンドは VContainer 経由で解決されるので、static メソッドに限定されません。

### 3. パレットから実行

- **Editor**: `Cmd/Ctrl + K` でエディタウィンドウとしてパレットを開き、"Player Set" などとファジー検索 → 引数を入力して Run。
- **Play Mode**: 同じ操作で半透明オーバーレイとして実際のゲーム画面の上に重なってパレットが開きます。

:::message
ショートカット機能には Unity のエディタ API を使っているため、Unity エディタの GUI からショートカット割り当てを変更することも可能です。
:::

### 4. HTTP API から実行

Editor 起動時に Bearer トークンが `~/.liminal-palette/token` へ自動生成され、localhost に HTTP サーバーが立ち上がります。

```bash
TOKEN=$(cat ~/.liminal-palette/token)

curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -X POST http://127.0.0.1:7610/api/v1/execute \
     -d '{"path":"Player/Health/Set","args":{"value":100}}'
# → {"success":true,"value":null,...}
```

シェルから叩けるということは、CI スクリプトや AI Agent からもゲームを操作できるということです。

## シナリオ機能

単発コマンドに加えて、`[LiminalScenario]` 属性で「コマンドを順次実行するシナリオ」を C# で宣言できます。
「敵スポーン → アイテム所持 → 特定ステージへワープ」のような連続操作をボタン 1 つで再現できるほか、HTTP API (`/api/v1/scenarios/run`) 経由で CI から走らせて統合テストとしても使えます。

```csharp:CombatScenarios.cs
public static class CombatScenarios
{
    [LiminalScenario("Combat/EnemyTakesDamage", Description = "敵にダメージを与えて HP が減ることを検証")]
    public static IEnumerable<ScenarioStep> EnemyTakesDamage()
    {
        yield return ScenarioStep.Run("Enemy/Spawn", new() { ["type"] = "Goblin" });
        yield return ScenarioStep.AssertEquals("Enemy/Hp", 100, "spawn 直後は満タン");
        yield return ScenarioStep.Run("Enemy/Damage", new() { ["amount"] = 30 });
        yield return ScenarioStep.WaitFrames(1);
        yield return ScenarioStep.AssertEquals("Enemy/Hp", 70, "30 ダメージ後は 70");
    }
}
```

これで `Cmd/Ctrl + K` → Scenario タブに `Combat/EnemyTakesDamage` が並び、Run Scenario で全ステップが順次実行されて、各ステップの ✓ / ✗ と所要時間が表示されます。

ステップは `ScenarioStep` の static ファクトリで組み立てます。

| ステップ | 説明 |
| --- | --- |
| `Run(path, args)` | `[LiminalCommand]` を呼ぶ。失敗で fail-fast |
| `WaitSeconds(sec)` / `WaitFrames(n)` | 実時間 / フレーム数で待機 |
| `AssertEquals(fieldPath, expected)` / `AssertNotEquals(...)` | `[LiminalObservableField]` の現在値を検証 |
| `AssertEventually(fieldPath, expected, timeoutSeconds)` | 値が期待値になるまで待ってから検証 |
| `AssertCommandReturns(path, args, expected)` | コマンドの戻り値を検証 |
| `LoadScene(sceneName)` | シーンをロード |

全ステップの末尾には任意引数 `description` があり、実行結果に表示されるラベルになります (上のコード例の `"spawn 直後は満タン"` がこれです)。

ただの `yield return` の列なので、C# の制御構文 (ループや条件分岐、ヘルパーメソッド) をそのまま使ってシナリオを組めるのがポイントです。

## AI Agent 連携 (Claude Code Skills)

Claude Code から HTTP API を叩くための Agent Skills を package に同梱しています。Unity のメニューから

```text
Tools > LiminalPalette > Install AI Skills...
```

を選ぶと、プロジェクトルートの `.claude/skills/` に 8 個の `liminal-*` skill がコピーされます。Claude Code を再起動すれば、

> 「LP のコマンド一覧を見せて」
> 「Player/Health/Set を 50 で実行して」

のような自然言語の指示で、Claude Code が適切な操作を自動選択してくれます。

AI Agent との統合方法として MCP サーバーではなく「Agent Skills + CLI ツール」という構成を採っているのは、同じ構成をとっている [uLoopMCP](https://github.com/hatayama/uLoopMCP) の使用感の良さに影響を受けています。
skill は「いつ・どう使うか」の知識だけを持ち、実際の操作は CLI に任せるこの構成は、MCP と違って常駐プロセスが不要で、skill の中身がただのテキストなのでプロジェクトごとのカスタマイズも容易です。

その CLI 側として、HTTP API をラップする Rust 製シングルバイナリ **liminal-cli** も開発中です。
トークンの読み込みやポートの特定を自動でやってくれるので、curl を直接組み立てるより人間にもスクリプトにも扱いやすくなります。

```bash
liminal exec Player/Health/Set value=100
liminal run "Title/*" --report out.xml  # シナリオを glob で一括実行 + JUnit XML 出力
```

@[card](https://github.com/void2610/liminal-cli)

:::details インストールされる 8 個の skill 一覧

| skill | 用途 |
| --- | --- |
| `liminal-overview` | 全 skill のエントリーポイント |
| `liminal-find-port` | LP サーバーの生存確認・ポート特定 |
| `liminal-list-commands` | コマンドと引数スキーマの一覧取得 |
| `liminal-execute` | コマンドの単発実行 |
| `liminal-get-state` | `[LiminalObservableField]` の現在値取得 |
| `liminal-get-logs` | コマンド実行履歴の取得 |
| `liminal-list-scenarios` | シナリオの一覧取得 |
| `liminal-run-scenario` | シナリオの実行 (glob 一括対応) |

:::

## uLoopMCP との使い分け

Unity を AI Agent から操作するツールとしては [uLoopMCP](https://github.com/hatayama/uLoopMCP) が有名です。
LiminalPalette はこれを置き換えるものではなく、責務が異なるので併用するのが前提です。私も両方を使っています。

| | uLoopMCP | LiminalPalette |
| --- | --- | --- |
| 対象 | Unity Editor そのもの | ゲーム内ロジック |
| 主な用途 | コンパイル確認 / テスト実行 / シーン階層取得 / Inspector 配線 / スクショ / Play モード切替 | コマンド実行 / 状態観測 / シナリオによる回帰テスト |
| 抽象度 | Editor API を直接叩く汎用操作 | ゲーム側が属性で公開した意味のある API |

一言でいうと、uLoopMCP は「Unity Editor のリモコン」、LiminalPalette は「ゲームが自分で公開する操作 / 観測 API」です。
前者は何も準備せずに使える代わりに、操作がゲームの意味を持ちません。
後者はゲーム側に属性を書く手間がかかる代わりに、「HP を 50 にする」「Day1 まで進める」といったゲームの語彙で AI に操作を渡せます。

なので、コード編集後のコンパイル確認やシーン構造の把握は uLoopMCP、ランタイムのゲーム動作確認やテストは LiminalPalette、という住み分けになります。

## おわりに

この記事で扱っているソースコードは以下のリポジトリで公開しています。
動作要件やインストール手順の詳細はリポジトリの README を参照してください。
フィードバックや Issue も歓迎です。

@[card](https://github.com/void2610/liminal-palette)

LiminalPalette を実際のプロジェクトで運用している事例と、シナリオを資産化していく具体的な運用ルールについては、別記事にまとめています。
よければこちらもご覧ください。

@[card](https://zenn.dev/void2610/articles/unity-e2e-scenario-26y07m20d)

## 参考文献

- [hatayama/uLoopMCP](https://github.com/hatayama/uLoopMCP)
- [Cysharp/R3](https://github.com/Cysharp/R3)
- [hadashiA/VContainer](https://github.com/hadashiA/VContainer)

-----
