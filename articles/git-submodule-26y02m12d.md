---
title: "Unity開発でも Git サブモジュールを使おう"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "Git", "サブモジュール", "ゲーム開発"]
published: false
---

## はじめに

Unity で複数のプロジェクトを開発していると、**汎用的なスクリプト（Utils）** を使い回したくなることがあります。
Unity のパッケージマネージャー（UPM）で管理する方法もありますが、もっと手軽な選択肢として **Git サブモジュール**があります。
この記事では、Git サブモジュールを使って汎用スクリプト群を管理する方法を紹介します。

:::message
この記事では Unity プロジェクトでのサブモジュールの活用方法に焦点を当てています。Git サブモジュール自体の基本的な使い方（追加・更新・削除など）については、他の記事を参考にしてください。
:::

## UPMとどっちを使うべき？

### Git サブモジュールと UPM の比較

| | Git サブモジュール | UPM (Git URL) |
|---|---|---|
| **セットアップ** | `git submodule add` で追加 | Package Manager ウィンドウから Git URL を指定、または manifest.json に追記 |
| **更新方法** | `git submodule update` | Packages ウィンドウから GUI で更新 |
| **バージョン管理** | サブモジュールのコミットで固定 | Git タグやコミットハッシュで固定 |
| **編集のしやすさ** | 直接編集してコミットできる | Packages 配下は読み取り専用 |
| **依存関係の解決** | 手動で管理 | UPM が自動で依存関係を解決 |
| **他パッケージとの統合** | Unity のパッケージシステムとは独立 | UPM エコシステムと統合されている |

### どちらを選ぶべきか？

UPMは現代的なパッケージマネージャーとしての機能が充実しています。
一方で、サブモジュールの最大のメリットは直接コードを編集してコミットし、普段のGit操作の延長で管理できる点です。
そのため、**大規模なライブラリの配布には UPM が適していますが、自分用の Utils のような小〜中規模の汎用コードにはサブモジュールが手軽ではないかと思います。**

## 注意点

サブモジュールを Unity プロジェクトで使う場合、いくつか注意すべき点があります。

### 1. メタファイル（.meta）も一緒にコミットする

Unity はアセットごとに `.meta` ファイルを生成し、その中に **GUID** が記録されています。
この GUID を使ってアセット間の参照を管理しているため、`.meta` ファイルもサブモジュールのリポジトリに含める必要があります。

:::message alert
`.meta` ファイルをサブモジュール側に含めないと、プロジェクトごとに異なる GUID が割り当てられ、参照が壊れてしまいます。
:::

### 2. シンボリックリンクを使って Assets 配下に配置する

`Assets/` の中にサブモジュールの実体を直接置くと、Unity が `.git` フォルダなどを認識してしまい問題が起きることがあります。
そのため、サブモジュールの実体はプロジェクトルートの `SubModules/` に配置し、`Assets/Scripts/Utils` は**シンボリックリンク**にします。
また、サブモジュールをルートに置くことで、GitHubなどからディレクトリを確認したときに、サブモジュールの存在がわかりやすくなるというメリットもあります。

:::message
シンボリックリンクの作成方法は他の記事を参照してください。
:::

### 3. Assembly Definition を定義する

サブモジュールのスクリプトが**メインプロジェクトのコードに依存しない**ように、Assembly Definition（`.asmdef`）を定義しましょう。

```json:Utils.asmdef
{
    "name": "Utils",
    "rootNamespace": "Utils",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

これにより以下のメリットがあります：
- サブモジュール側のコンパイルがメインプロジェクトから独立する
- 依存関係が明確になる
- コンパイル時間が短縮される

メインプロジェクト側からサブモジュールの機能を使いたい場合は、メインの `.asmdef` の `references` に `Utils` を追加します。

### 4. 名前空間（namespace）を切る

Assembly Definition と合わせて、サブモジュールのコードには**専用の名前空間**を設定しましょう。

名前空間を切ることで以下のメリットがあります：
- メインプロジェクトのクラスとの**名前の衝突を防げる**
- `using Utils.Extensions;` のように、**どのコードがサブモジュール由来なのか明確**になる
- サブモジュールを複数のプロジェクトに導入しても安全

## 具体的なプロジェクト構成

以下のようなディレクトリ構成をおすすめします。

### 全体像

```
MyUnityProject/
├── .gitmodules
├── SubModules/              # サブモジュールの実体（プロジェクトルートに配置）
│   └── Utils/
│       ├── Runtime/
│       │   ├── Extensions/
│       │   │   ├── TransformExtensions.cs
│       │   │   └── TransformExtensions.cs.meta
│       │   ├── Helpers/
│       │   │   ├── MathHelper.cs
│       │   │   └── MathHelper.cs.meta
│       │   └── Utils.asmdef
│       └── .git
├── Assets/
│   └── Scripts/
│       ├── Utils -> ../../SubModules/Utils/Runtime  # シンボリックリンク
│       ├── Utils.meta
│       ├── Game/
│       │   └── ...
│       └── ...
└── ...
```

## 自分の Utils リポジトリの紹介

参考として、私が実際に使っている Utils リポジトリを公開しています。

https://github.com/void2610/my-unity-utils

このリポジトリをサブモジュールとして追加するだけで、新しい Unity プロジェクトですぐに使い始めることができます。

## まとめ

- Git サブモジュールは**普段の Git 操作の延長で管理できる**ので、特別なワークフローが不要
- **直接編集してコミットできる**ので、開発中の改善がスムーズ
- `.meta` ファイル、Assembly Definition、名前空間の設定を忘れずに
- **シンボリックリンク**を使えば、Unity のフォルダ構成をすっきり保てる
- 小〜中規模の汎用コードの共有にはサブモジュールが手軽でおすすめ

## 宣伝

この記事で紹介したサブモジュール管理の手法は、実際に以下のゲーム開発で活用しています。
よければウィッシュリストに追加していただけると嬉しいです！

### VOID RED
「あなただけ」の記憶構築ADV。記憶を失った少女が記憶を取り戻すためオークションに挑む。
https://store.steampowered.com/app/3997140/

### 庭小人の庭
植物育成とターン制バトルを組み合わせたデッキ構築型ローグライクゲーム。種から育てたカードで戦略的に戦おう。
https://store.steampowered.com/app/4105720/
