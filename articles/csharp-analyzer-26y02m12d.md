---
title: "Unity開発者はC#オレオレアナライザーを作ろう"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "dotnet", "Roslyn"]
published: false
---

## はじめに

AIが生成するコードがプロジェクトの規約に沿わなかったり、チーム制作でコードスタイルがバラバラになったりすることがあると思います。
C#でカスタムアナライザーを作れば、基本的な命名規則や改行ルールに加えて、独自のルールで違反を検出・自動修正できます。
最近はAIでアナライザーを作ることも容易になってきたので、ぜひ作ってみるのはいかがでしょうか。

参考に、私が実際に使用しているUnity向けカスタムアナライザーのリポジトリを置いておきます。
https://github.com/void2610/unity-analyzers

## 他のフォーマットツールとの比較

C# のコードフォーマット・静的解析ツールにはいくつかの選択肢があります。

| ツール | 特徴 | Unity対応 |
|---|---|---|
| **dotnet-format** | .NET 公式のフォーマッター。`.editorconfig` でルール設定可能。カスタムアナライザーで拡張できる | ◎ |
| **StyleCop.Analyzers** | Microsoft が提供する定番のスタイルチェッカー | ○ |
| **Roslynator** | 500以上のアナライザーとリファクタリングを提供 | ○ |
| **ReSharper CLI** | JetBrains 製。高機能で CLI ツールは無料 | ○ |

`dotnet-format` + カスタムアナライザーの組み合わせは、**自分のプロジェクトに合ったルールを自由に定義できる**という点で最も柔軟です。

## カスタムアナライザーとは

Roslyn アナライザーは、C# コンパイラ（Roslyn）のAPIを使ってコードを解析し、診断メッセージを出力するコンポーネントです。
アナライザーに加えて **CodeFix Provider** を実装すれば、`dotnet-format` で自動修正も可能になります。

アナライザーが動作する流れは以下の通りです。

```
ソースコード → Roslyn コンパイラ → 構文木(Syntax Tree) 
→ アナライザーが解析 → 診断結果 → CodeFix が自動修正
```

## AIでアナライザーを作る

Roslyn API は構造化されたパターンが多く、AIが得意な領域です。
慣例的にテストコードもセットで用意する必要があるため、AI生成コードの品質も担保しやすいです。
筆者のリポジトリの8つのルールも、全てClaudeCodeが実装しました。

## カスタムアナライザーの作成手順

以下に、AIを活用してアナライザーを作成する上での最低限の知識をまとめます。

### プロジェクト構成

アナライザープロジェクトは以下のような構成になります。

```
MyAnalyzers/
├── MyAnalyzers/          # アナライザー本体
│   └── Rules/            # 各ルールの実装
├── MyAnalyzers.Tests/    # テストプロジェクト
└── nuget/                # NuGet パッケージ出力先
```

ポイント:

- ターゲットフレームワークは `netstandard2.0`（Roslyn アナライザーの要件）
- `Microsoft.CodeAnalysis.CSharp` パッケージを参照
- NuGet パッケージとして配布するための設定が必要（`IsRoslynComponent`、`PackagePath` の指定など）

### .editorconfig での設定

Unity プロジェクトのルートに `.editorconfig` を配置して、アナライザーのルールを制御できます。

```ini:.editorconfig
[*.cs]
# カスタムアナライザーのルールを有効化
dotnet_diagnostic.VUA0001.severity = warning
dotnet_diagnostic.VUA0002.severity = warning
```

### ビルドと利用

```bash
# アナライザーをビルド・パッケージ化
dotnet pack MyAnalyzers/MyAnalyzers.csproj -c Release -o ./nuget
```

ビルドしたアナライザーは `dotnet-format` 経由で利用します。

```bash
# 自動修正
dotnet format Assembly-CSharp.csproj
```

活用例:
- **AI生成コードをフォーマットする**: AI のルールに `dotnet format` の実行を組み込む
- **CIで自動チェック**: GitHub Actions 等で `dotnet format --verify-no-changes` を実行し、規約違反のある PR を検出する
- **エディタの保存時アクション**: 保存のたびにフォーマットを実行して常に規約に揃える

## アナライザーの実装例

以下に、簡単なアナライザーの実装例を示します。AIを使用する際はここの部分は不要かと思います。
カスタムアナライザーは主に2つのコンポーネントで構成されます。

### DiagnosticAnalyzer（検出）

構文木を解析し、ルール違反を検出して診断メッセージを報告します。

```csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class MyAnalyzer : DiagnosticAnalyzer
{
    // 診断ルールの定義
    private static readonly DiagnosticDescriptor Rule = new(
        id: "VUA0001",
        title: "ルールのタイトル",
        messageFormat: "違反メッセージ: '{0}'",
        category: "Design",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true
    );

    public override void Initialize(AnalysisContext context)
    {
        // 解析対象のノード種別を登録
        context.RegisterSyntaxNodeAction(Analyze, SyntaxKind.FieldDeclaration);
    }
}
```

### CodeFixProvider（自動修正）

検出された違反を自動修正するコードを提供します。`dotnet-format` はこの CodeFix を使って一括修正を行います。

```csharp
[ExportCodeFixProvider(LanguageNames.CSharp), Shared]
public class MyCodeFix : CodeFixProvider
{
    public override ImmutableArray<string> FixableDiagnosticIds
        => ImmutableArray.Create("VUA0001");

    public override async Task RegisterCodeFixesAsync(CodeFixContext context)
    {
        // 修正アクションを登録
    }
}
```

## 実装したルールの例

Unity 開発では `[SerializeField]` の有無でフィールドの命名規則を変えたいケースがあります。
例えば、private フィールドには `_camelCase` を強制しつつ、`[SerializeField]` 付きのprivateフィールドにはプレフィックスを付けないようにする、
といったルールはUnity公式が推奨しているものの、有名どころのフォーマッタでは対応していません。

こうしたマイナーな規則にはカスタムアナライザーの出番です。
筆者のリポジトリではこのようなルールを含む8つのルールを実装しています。詳細は[リポジトリ](https://github.com/void2610/unity-analyzers)を参照してください。


## まとめ

- カスタムアナライザーで**プロジェクト専用のルール**を定義できる
- CodeFix Provider で `dotnet-format` による**自動修正**が可能
- AIのルールやCIに組み込んで**コード品質を継続的に維持**できる
- **AIの得意分野**なので、Roslyn API の深い知識がなくても始められる

カスタムアナライザーは一度作ってしまえば、チーム全体のコード品質を継続的に維持するための強力なツールになります。ぜひ[リポジトリ](https://github.com/void2610/unity-analyzers)を参考に、自分のプロジェクトに合ったアナライザーを AI と一緒に作ってみてください。

## 宣伝

今回紹介した内容は、以下のゲーム開発にも活用しています。
よければウィッシュリストに追加していただけると嬉しいです！

### VOID RED
「あなただけ」の記憶構築ADV。記憶を失った少女が記憶を取り戻すためオークションに挑む。
https://store.steampowered.com/app/3997140/

### 庭小人の庭
植物育成とターン制バトルを組み合わせたデッキ構築型ローグライクゲーム。種から育てたカードで戦略的に戦おう。
https://store.steampowered.com/app/4105720/
