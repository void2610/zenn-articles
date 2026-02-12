---
title: "Unity開発向け C# オレオレアナライザーを作ろう"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "dotnet", "GithubActions", "Roslyn"]
published: false
---

## はじめに

Unity 開発で C# のフォーマットやコーディング規約チェックを行いたいと思ったことはありませんか？
チームで開発していると、コーディングスタイルがバラバラになってレビューで指摘が増えたり、可読性が下がったりすることがあります。

この記事では、`dotnet-format` のために独自の Roslyn アナライザーを実装して利用する方法を解説します。
プロジェクト構成やディレクトリ構成、他のフォーマットツールとの比較、GitHub Actions での自動チェック、そして AI の出力を修正する活用法まで紹介します。

## 他のフォーマットツールとの比較

C# のコードフォーマット・静的解析ツールにはいくつかの選択肢があります。

| ツール | 特徴 | Unity対応 |
|---|---|---|
| **dotnet-format** | .NET 公式のフォーマッター。`.editorconfig` でルール設定可能。カスタムアナライザーで拡張できる | ◎ |
| **StyleCop.Analyzers** | Microsoft が提供する定番のスタイルチェッカー | ○ |
| **Roslynator** | 500以上のアナライザーとリファクタリングを提供 | ○ |
| **ReSharper CLI** | JetBrains 製。高機能だが有料 | ○ |

`dotnet-format` + カスタムアナライザーの組み合わせは、**自分のプロジェクトに合ったルールを自由に定義できる**という点で最も柔軟です。

## カスタムアナライザーとは

Roslyn アナライザーは、C# コンパイラ（Roslyn）のAPIを使ってコードを解析し、診断メッセージを出力するコンポーネントです。
アナライザーに加えて **CodeFix Provider** を実装すれば、`dotnet-format` で自動修正も可能になります。

### アナライザーの仕組み

```
ソースコード → Roslyn コンパイラ → 構文木(Syntax Tree) → アナライザーが解析 → 診断結果
                                                         → CodeFix が自動修正
```

## プロジェクト構成

以下のようなディレクトリ構成でアナライザープロジェクトを作成します。

```
MyAnalyzers/
├── MyAnalyzers.sln
├── MyAnalyzers/
│   ├── MyAnalyzers.csproj        # アナライザー本体
│   ├── Rules/
│   │   ├── FieldNamingAnalyzer.cs
│   │   └── FieldNamingCodeFix.cs
│   └── DiagnosticIds.cs
├── MyAnalyzers.Tests/
│   ├── MyAnalyzers.Tests.csproj  # テストプロジェクト
│   └── FieldNamingAnalyzerTests.cs
└── nuget/                         # NuGet パッケージ出力先
```

### アナライザーのプロジェクトファイル

```xml:MyAnalyzers/MyAnalyzers.csproj
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
    <IsRoslynComponent>true</IsRoslynComponent>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
    <PackageId>MyAnalyzers</PackageId>
    <PackageVersion>1.0.0</PackageVersion>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <!-- NuGet パッケージとしてアナライザーを配布するための設定 -->
    <DevelopmentDependency>true</DevelopmentDependency>
    <SuppressDependenciesWhenPacking>true</SuppressDependenciesWhenPacking>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.4">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" />
  </ItemGroup>

  <ItemGroup>
    <!-- パッケージ化時にアナライザー DLL を正しい場所に配置 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll"
          Pack="true"
          PackagePath="analyzers/dotnet/cs"
          Visible="false" />
  </ItemGroup>

</Project>
```

## アナライザーの実装例

例として、**フィールド名が `_camelCase` であることを強制するアナライザー**を作ってみましょう。

### 診断IDの定義

```csharp:MyAnalyzers/DiagnosticIds.cs
namespace MyAnalyzers
{
    // 診断IDを一元管理するクラス
    public static class DiagnosticIds
    {
        public const string FieldNaming = "MY001";
    }
}
```

### アナライザー本体

```csharp:MyAnalyzers/Rules/FieldNamingAnalyzer.cs
using System.Collections.Immutable;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;

namespace MyAnalyzers.Rules
{
    [DiagnosticAnalyzer(LanguageNames.CSharp)]
    public class FieldNamingAnalyzer : DiagnosticAnalyzer
    {
        // 診断ルールの定義
        private static readonly DiagnosticDescriptor Rule = new(
            id: DiagnosticIds.FieldNaming,
            title: "フィールド名はアンダースコア + camelCase にする",
            messageFormat: "フィールド '{0}' は '_camelCase' 形式にしてください",
            category: "Naming",
            defaultSeverity: DiagnosticSeverity.Warning,
            isEnabledByDefault: true
        );

        public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics
            => ImmutableArray.Create(Rule);

        public override void Initialize(AnalysisContext context)
        {
            // 生成されたコードは解析対象外にする
            context.ConfigureGeneratedCodeAnalysis(
                GeneratedCodeAnalysisFlags.None);
            context.EnableConcurrentExecution();

            // フィールド宣言を監視対象に登録
            context.RegisterSyntaxNodeAction(
                AnalyzeFieldDeclaration,
                SyntaxKind.FieldDeclaration);
        }

        // フィールド宣言を解析するメソッド
        private static void AnalyzeFieldDeclaration(
            SyntaxNodeAnalysisContext context)
        {
            var fieldDeclaration = (FieldDeclarationSyntax)context.Node;

            // public フィールドは対象外
            if (fieldDeclaration.Modifiers.Any(SyntaxKind.PublicKeyword))
                return;

            // const フィールドは対象外
            if (fieldDeclaration.Modifiers.Any(SyntaxKind.ConstKeyword))
                return;

            foreach (var variable in fieldDeclaration.Declaration.Variables)
            {
                var name = variable.Identifier.Text;

                // アンダースコアで始まり、2文字目が小文字であることをチェック
                if (IsValidFieldName(name))
                    continue;

                var diagnostic = Diagnostic.Create(
                    Rule,
                    variable.Identifier.GetLocation(),
                    name);
                context.ReportDiagnostic(diagnostic);
            }
        }

        // フィールド名が _camelCase 形式かどうかを判定
        private static bool IsValidFieldName(string name)
        {
            if (name.Length < 2) return false;
            if (name[0] != '_') return false;
            if (!char.IsLower(name[1])) return false;
            return true;
        }
    }
}
```

### CodeFix Provider

```csharp:MyAnalyzers/Rules/FieldNamingCodeFix.cs
using System.Collections.Immutable;
using System.Composition;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CodeActions;
using Microsoft.CodeAnalysis.CodeFixes;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.Rename;

namespace MyAnalyzers.Rules
{
    [ExportCodeFixProvider(LanguageNames.CSharp), Shared]
    public class FieldNamingCodeFix : CodeFixProvider
    {
        public override ImmutableArray<string> FixableDiagnosticIds
            => ImmutableArray.Create(DiagnosticIds.FieldNaming);

        public override FixAllProvider GetFixAllProvider()
            => WellKnownFixAllProviders.BatchFixer;

        public override async Task RegisterCodeFixesAsync(
            CodeFixContext context)
        {
            var root = await context.Document
                .GetSyntaxRootAsync(context.CancellationToken);
            var diagnostic = context.Diagnostics[0];
            var span = diagnostic.Location.SourceSpan;

            // 対象のトークンを取得
            var token = root.FindToken(span.Start);
            var oldName = token.Text;
            var newName = ConvertToUnderscoreCamelCase(oldName);

            context.RegisterCodeFix(
                CodeAction.Create(
                    title: $"'{newName}' にリネーム",
                    createChangedSolution: async ct =>
                    {
                        var semanticModel = await context.Document
                            .GetSemanticModelAsync(ct);
                        var symbol = semanticModel
                            .GetDeclaredSymbol(token.Parent, ct);
                        // Renamer を使って参照箇所も含めてリネーム
                        return await Renamer.RenameSymbolAsync(
                            context.Document.Project.Solution,
                            symbol,
                            new SymbolRenameOptions(),
                            newName,
                            ct);
                    },
                    equivalenceKey: DiagnosticIds.FieldNaming),
                diagnostic);
        }

        // フィールド名を _camelCase 形式に変換するメソッド
        private static string ConvertToUnderscoreCamelCase(string name)
        {
            // 既存のアンダースコアを除去
            var trimmed = name.TrimStart('_');
            if (string.IsNullOrEmpty(trimmed))
                return "_field";

            // 先頭を小文字にして _ を付ける
            return "_" + char.ToLower(trimmed[0]) + trimmed.Substring(1);
        }
    }
}
```

## .editorconfig での設定

Unity プロジェクトのルートに `.editorconfig` を配置して、アナライザーのルールを制御できます。

```ini:.editorconfig
[*.cs]
# カスタムアナライザーのルールを有効化
dotnet_diagnostic.MY001.severity = warning
```

## ビルドとパッケージ化

```bash
# アナライザーをビルド
dotnet build MyAnalyzers/MyAnalyzers.csproj -c Release

# NuGet パッケージを作成
dotnet pack MyAnalyzers/MyAnalyzers.csproj -c Release -o ./nuget
```

作成した NuGet パッケージは、ローカルの NuGet ソースとして登録して利用できます。

```bash
# ローカルの NuGet ソースとして登録
dotnet nuget add source ./nuget --name LocalAnalyzers
```

## Unity プロジェクトで使う

Unity プロジェクトで `dotnet-format` を実行する場合は、`.csproj` ファイルに対して実行します。

```bash
# フォーマットチェック（修正はしない）
dotnet format Assembly-CSharp.csproj --verify-no-changes --diagnostics MY001

# 自動修正
dotnet format Assembly-CSharp.csproj --diagnostics MY001
```

## GitHub Actions で動かす

GitHub Actions でプルリクエスト時に自動でコーディング規約チェックを行うワークフローの例です。

```yaml:.github/workflows/format-check.yml
name: Format Check

on:
  pull_request:
    paths:
      - '**.cs'

jobs:
  format-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: カスタムアナライザーを復元
        run: dotnet restore

      - name: フォーマットチェック
        run: |
          dotnet format --verify-no-changes --diagnostics MY001 \
            --verbosity diagnostic 2>&1 | tee format-result.txt

      - name: 結果をPRにコメント
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const result = fs.readFileSync('format-result.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ⚠️ フォーマットチェック失敗\n\n\`\`\`\n${result.slice(-3000)}\n\`\`\``
            });
```

## ローカルで AI の出力を修正する

LLM（GitHub Copilot や ChatGPT など）にコードを生成してもらった際、プロジェクトのコーディング規約に沿っていないことがよくあります。
`dotnet-format` + カスタムアナライザーを使えば、AI が生成したコードも一括で規約に揃えることができます。

### 使い方

1. AI にコードを生成してもらう
2. 生成されたコードをプロジェクトに配置
3. `dotnet-format` を実行して自動修正

```bash
# AI が生成したコードをフォーマット
dotnet format --diagnostics MY001

# 差分を確認
git diff
```

これをエディタの保存時アクションやpre-commitフックに組み込むと、常にコーディング規約に沿ったコードを維持できます。

```bash
# .git/hooks/pre-commit の例
#!/bin/sh
dotnet format --verify-no-changes --diagnostics MY001
if [ $? -ne 0 ]; then
    echo "フォーマットチェックに失敗しました。dotnet format を実行してください。"
    exit 1
fi
```

## まとめ

- Roslyn アナライザーを使えば、**自分のプロジェクト専用のコーディング規約チェッカー**を作れる
- CodeFix Provider を実装すれば、`dotnet-format` で**自動修正**も可能
- GitHub Actions に組み込めば、**PR 時の自動チェック**が実現できる
- AI が生成したコードも**一括でフォーマット**できる

カスタムアナライザーは一度作ってしまえば、チーム全体のコード品質を継続的に維持するための強力なツールになります。ぜひ試してみてください。
