---
title: "AI Agentに動作確認を任せるほどE2Eテストが増えるUnity開発運用"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "ClaudeCode", "AI", "testing"]
published: false
---

## はじめに

Unity のゲーム開発で、こんな経験はないでしょうか。

- Claude Code に実装を任せたが、動作確認は「Unity を開いて確認してください」と人間に投げ返されてしまう
- 動作確認はしているのに、それが毎回その場で消費されて、テスト資産として何も残らない
- E2E テストを書こうとしたが、MonoBehaviour / Prefab / DI が絡んで mock だらけになり、書くコストが見合わない

この記事では、自作の Unity 用デバッグコンソール **LiminalPalette** を軸に、**「AI Agent に動作確認を任せれば任せるほど、E2E 回帰テストが勝手に増えていく」** 運用を実際のプロジェクトでどう組んでいるかを紹介します。

LiminalPalette 自体の紹介は別記事に書いたので、コマンドやシナリオの基本的な書き方はそちらを参照してください。

@[card](https://zenn.dev/void2610/articles/liminal-palette-26y07m20d)

:::message
ざっくり前提だけ書くと、LiminalPalette は `[LiminalCommand]` を付けた C# メソッドを Editor / Runtime / HTTP API から実行でき、`[LiminalScenario]` でコマンド列 + assert を「シナリオ」として宣言できるライブラリです。AI Agent は HTTP API 経由でこれらを叩きます。
:::

## 対象プロジェクト

現在開発中のゲーム「庭小人の庭」「カラーリコレクション」の 2 プロジェクトで実運用しています。

@[card](https://store.steampowered.com/app/4105720/)

@[card](https://store.steampowered.com/app/4848670/)

ここからは「カラーリコレクション」での運用を紹介します。まだプロトタイプ段階のプロジェクトですが、この時点ですでに以下の規模のコマンド・テストが整備できています。

| 種別 | 数 |
| --- | --- |
| デバッグコマンド用クラス | 41 ファイル (`Assets/Scripts/Debug/` に集約) |
| `[LiminalCommand]` | 171 個 |
| `[LiminalScenario]` | 54 個 |
| `[LiminalObservableField]` | 6 個 |

プロトタイプの初期からこの密度でテストが貯まるのは、後述する「検証のシナリオ化」運用の効果が大きいです。

## テストの種類と配置場所を固定する

まず前提として、`Docs/test-strategy.md` に「どこに何のテストを書くか」を固定しています。迷いをなくすことがテストを増やし続けるための第一条件です。

| 種類 | 配置 | 主な役割 |
| --- | --- | --- |
| LiminalScenario | `Assets/Scripts/Debug/*Scenarios.cs` | end-to-end の挙動検証 (シーン込み) |
| PlayMode テスト | `Assets/Tests/PlayMode/*E2ETests.cs` | LiminalScenario を Test Runner で回す薄いランナー |
| EditMode テスト | `Assets/Tests/EditMode/*Tests.cs` | 純粋な C# クラスのロジック単体検証 |
| LiminalCommand | `Assets/Scripts/Debug/*DebugCommands.cs` | シナリオ / 手動デバッグ両用の操作・観測 API |

**「ゲームプレイ挙動の検証はシナリオで書く」が中心線**で、EditMode の単体テストはモックなしで完結する純粋ロジック (Model / パズル判定など) 専用と割り切っています。

MonoBehaviour で物理や Update を持ち、Prefab の SerializeField を前提に動き、VContainer で組み立てられるオブジェクトを EditMode でカバーしようとすると、fake / mock の boilerplate が膨大になり、書くコストも壊れたときの修正コストも見合いません。**実シーンを動かして外側から観測する**ほうが、不変条件をそのままステップの列で書けて、リファクタリングしても外向きの挙動が変わらない限りテストが壊れません。

:::message
シナリオは本番の `MainScene` 上でのみ回し、パート別のテストシーンは廃止しました。手置きのテストシーンは本番構成から drift し、「本番が壊れているのにテストだけ通る」偽の安心を生むためです。
:::

## CLAUDE.md で AI Agent の開発ワークフローを固定する

Claude Code が毎回同じ手順を踏むよう、`CLAUDE.md` に開発ワークフローを明文化しています。

```markdown:CLAUDE.md
## 開発ワークフロー

1. コードを変更する
2. `uloop-compile` (ForceRecompile=true) でコンパイル
3. `uloop-get-logs` (LogType=Error) でコンパイルエラーがないことを確認
4. ロジック変更があれば `uloop-run-tests` (EditMode / 必要なら PlayMode) を回す
5. コードを編集した場合は、必ず `run-format.sh` を実行する
6. ツール出力は末尾サマリだけで判断せず必ず全文を確認する
7. 結果をユーザーに報告する
```

さらに、ツールの使い分けの原則をこう書いています。

> Unity 側の操作は **uloop / LiminalPalette skill を第一選択** とする。
> 手動操作を依頼する前にまず skill で完結できないか検討すること。

これがないと AI は「Unity を開いて確認してください」と人間に投げ返してきます。**「人間に手動操作を頼む前に skill で完結できないか検討する」を明文化しておくことが、自律的な動作確認を回すうえで一番効きました。**

なお、詳細な運用方針は CLAUDE.md に直接書かず、**unity-coding-standards** リポジトリで管理している自作の Claude Code プラグイン `unity-standards` の skill (`uloop-guide` / `liminal-palette-guide`) に集約し、CLAUDE.md からは参照だけしています。複数プロジェクトで同じ方針を共有するためです。

@[card](https://github.com/void2610/unity-coding-standards)

このプラグインには運用ガイドのほか、コーディング規約や Roslyn アナライザー、UI View の Prefab 構築手順なども入れています。

## 観測点とコマンドをワンセットで生やす

新しいゲームプレイ系のクラスを実装したら、あわせてデバッグコマンドクラスを `Debug/` に足します。
ポイントは「状態を強制するコマンド」と「状態を観測するコマンド / フィールド」を対で用意することです。

```csharp:GameStateDebugCommands.cs
public sealed class GameStateDebugCommands
{
    // R3 の ReadOnlyReactiveProperty をそのまま公開 → UI に常時表示 + assert のポーリング対象になる
    [LiminalObservableField("Game/State/PhaseField", Description = "現在のゲームフェーズ")]
    public ReadOnlyReactiveProperty<GamePhase> PhaseField => _session.CurrentPhase;

    // 観測コマンドは string 返しにして、シナリオの期待値比較に使う
    [LiminalCommand("Game/State/Phase", Description = "現在のゲームフェーズ名を返す")]
    public string GetPhase() => _session.CurrentPhase.CurrentValue.ToString();
}
```

デバッグ用 asmdef は本番ビルドから丸ごと除外されるので、製品版にこれらのコードは含まれません。

## 検証をその場で消費せず、シナリオとして資産化する

この運用の核心です。単発のコマンド実行で動作確認を済ませることもできますが、それだと検証はその場で消費されて終わりです。
そこで、`liminal-palette-guide` skill に以下のルールを書いて Claude Code に守らせています。

:::message
- **その場限りの `liminal-execute` 連打で確認を済ませない**。同じ手順を 2 回踏みそうな気配があれば、迷わず `[LiminalScenario]` に切り出す
- **バグを直したら、その再現手順をシナリオにする**。同じ退行が二度と通らないようにする最小コスト
- **新機能を実装したら、その happy path と境界値をシナリオにする**。実装と同じ PR で増やす
- **シナリオは細かく多くて良い**。1 シナリオ 1 確認の粒度で、`liminal-run-scenario "Battle/*"` のような glob で束ねて回す
- **シナリオが書けない = 観測点 / コマンドが足りないサイン**。属性を生やすか、ライブラリ側に汎用 API を足す
:::

実際のシナリオはこのような形になります。

```csharp:TitleScenarios.cs
[LiminalScenario(
    "Title/Scenario/ContinueLoadsAutoSave",
    Scene = "TitleScene",
    Description = "つづきから → AUTO 読込で本編シーンへ遷移し Phase が復元される")]
public static IEnumerable<ScenarioStep> ContinueLoadsAutoSave()
{
    yield return ScenarioStep.Run("Game/Debug/ResetAll");
    yield return ScenarioStep.Run("Game/Debug/GoToPhase", Args("phase", GamePhase.Day1Novel));

    // 実 Raycast クリック (フェード中は正しく失敗するため成功までポーリング)
    yield return ClickStep("TitleView/continueButton", "つづきから実クリック");

    yield return ScenarioStep.Run("SaveLoad/Load", Args("slot", -1));
    // 固定待ちだと CI 負荷時に flaky になるため、観測フィールドを AssertEventually でポーリング
    yield return ScenarioStep.AssertEventually("Game/State/PhaseField", "Day1Novel", 10f, "Phase 復元待ち");
}
```

ポイントは **「検証作業」と「テスト資産の追加」を別タスクにしない**ことです。前者を後者の形式で書く、というだけの話にしておくと、テストを書く心理的コストがほぼゼロになります。

この運用にすると、バグ修正・機能追加のたびに「今回の検証手順」をシナリオに 1 本足すだけで、**回帰テストのコーパスが副作用的に増え続けます**。カラーリコレクションでは、この積み重ねだけで 54 本の E2E シナリオが貯まりました。

逆に「シナリオが書けない」と感じたら、それは観測点かコマンドが足りないサインなので、`[LiminalObservableField]` / `[LiminalCommand]` を生やしに戻ります。

## シナリオを Unity Test Runner に流し込む

貯めたシナリオは、PlayMode テストとして Unity Test Runner からも回せます。ランナーはプレフィックス指定で全シナリオを拾うだけの薄い実装で済みます。

```csharp:TitleScenariosE2ETests.cs
public sealed class TitleScenariosE2ETests
{
    [UnityTest]
    public IEnumerator Run([ValueSource(nameof(Paths))] string scenarioPath)
        => LiminalPaletteTestRunner.RunScenario(scenarioPath);

    public static IEnumerable<string> Paths
        => LiminalPaletteTestRunner.GetScenariosWithPrefix("Title/Scenario/");
}
```

`"Title/Scenario/"` プレフィックスのシナリオを足せば**自動で Test Runner に出現する**ので、テスト追加のたびにランナー側を触る必要がありません。カラーリコレクションでは Cinematic / Puzzle / Novel / Decision / Report / Title / Noema / Integration の 8 ファイルをこの形で置いています。

## 決定性を保つためのパターン

E2E は放っておくと flaky になります。カラーリコレクションで効果があったパターンをいくつか挙げます。

- **基準フェーズへの明示遷移**: `MainScene` の起動フェーズは前テストのオートセーブ復元に依存してしまうため、シナリオ冒頭で `ResetToPhase(GamePhase.Day1Novel)` を呼び、`ResetAll` + 基準フェーズ入場までを決定化する。前テストの状態への暗黙依存を新規シナリオに持ち込まない
- **固定待ちを使わない**: `WaitSeconds` で決め打ちすると CI 負荷時に落ちるので、`AssertCommandEventually` / `AssertEventually` で観測点をポーリングする
- **操作コマンドは受理を返す**: 「操作したが内部で握り潰された」ケースで成功を返すと、シナリオが誤って成功と判定して順序依存の flaky を生む。受理できたかどうかを返し、シナリオ側は受理までリトライする
- **`Assert*` コマンドを作らない**: `World/Assert/EnemyHp` のような domain-specific な assert コマンドは足さず、**戻り値 string の観測コマンドだけ**を用意してシナリオ側で `AssertCommandReturns` の期待値と比較する。観測と判定の責務を分けることで、観測コマンドはパレット UI からも human-readable に確認できる

## uLoopMCP との棲み分け

Unity を AI Agent から操作するツールとしては [uLoopMCP](https://github.com/hatayama/uLoopMCP) があり、カラーリコレクションでは **LiminalPalette と併用**しています。
両者は競合ではなく、責務がきれいに分かれます。

| | uLoopMCP | LiminalPalette |
| --- | --- | --- |
| 対象 | **Unity Editor そのもの** | **ゲーム内ロジック** |
| 主な用途 | コンパイル確認 / テスト実行 / シーン階層取得 / Inspector ワイヤリング / スクリーンショット / Play モード切替 | コマンド実行 / 状態観測 / シナリオによる回帰テスト |
| 抽象度 | Editor API を直接叩く汎用操作 | ゲーム側が属性で公開した意味のある API |

運用ルールはこうしています。

- **Editor 自体の操作** (コード編集後の `uloop-compile` → エラー確認、`uloop-get-hierarchy` でのシーン構造把握、`uloop-execute-dynamic-code` での SerializeField 設定など) は **uLoopMCP**。
- **ランタイムのゲーム動作確認・回帰テスト** (コマンド実行、`[LiminalObservableField]` の状態観測、シナリオ実行) は **LiminalPalette**。
- uLoopMCP にはキーボード / マウス入力のシミュレーションや入力の録画再生もありますが、**回帰テスト用途では使いません**。
録画再生は壊れやすくレビューもしづらいため、テスト内容は `[LiminalScenario]` としてコードで表現します。
- `uloop-execute-dynamic-code` で C# を直接流すのは、該当する `[LiminalCommand]` が無い場合の最後の手段。
まず属性を生やせないかを検討します。

一言でいうと、**uLoopMCP は「Unity Editor のリモコン」、LiminalPalette は「ゲームが自分で公開する操作・観測 API」** です。
前者は汎用的ですが操作がゲームの意味を持たず、後者はゲーム側の実装が必要な代わりに「HP を 50 にする」「Day1 まで進める」といった意味のある単位で AI に操作を渡せます。
AI Agent に開発を任せるなら、この 2 段構えがいまのところ一番安定しています。

## おわりに

この運用の要点をまとめます。

1. **テストの種類と配置場所を固定する** — 迷わないことがテストを増やし続ける条件
2. **CLAUDE.md / skill にワークフローと判断基準を書く** — 「人間に手動操作を頼む前に skill を検討する」が効く
3. **検証をシナリオとして資産化する** — 検証作業とテスト追加を別タスクにしない
4. **決定性のパターンを共有する** — 固定待ちを避け、観測点をポーリングする

「変更後にゲームを起動して目視で確認してください」を CI に置き換えつつ、日々の動作確認がそのまま回帰テストとして積み上がっていく状態は、一度作るとかなり快適です。

## リポジトリ

この記事で扱っている LiminalPalette は以下で公開しています。

@[card](https://github.com/void2610/liminal-palette)

## 参考文献

- [void2610/unity-coding-standards](https://github.com/void2610/unity-coding-standards)
- [hatayama/uLoopMCP](https://github.com/hatayama/uLoopMCP)
- [Cysharp/R3](https://github.com/Cysharp/R3)
- [hadashiA/VContainer](https://github.com/hadashiA/VContainer)
