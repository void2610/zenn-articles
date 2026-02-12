---
title: "Unity開発でも Git サブモジュールを使おう"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "Git", "サブモジュール", "ゲーム開発"]
published: false
---

## はじめに

Unity で複数のプロジェクトを開発していると、**汎用的なスクリプト（Utils）** を使い回したくなることがあります。
コピペで管理していると更新が大変ですし、Unity のパッケージマネージャー（UPM）で管理する方法もありますが、セットアップや更新に手間がかかります。

この記事では、**Git サブモジュール**を使って汎用スクリプトを管理する方法を紹介します。
普段の Git 操作だけで完結するため、更新時の手間が少ないのが大きな利点です。

## Unity パッケージマネージャー（UPM）との比較

| | Git サブモジュール | UPM (Git URL) |
|---|---|---|
| **セットアップ** | `git submodule add` だけ | `package.json` の作成、manifest への追記が必要 |
| **更新方法** | `git submodule update` | Packages ウィンドウから更新 or manifest 書き換え |
| **バージョン管理** | サブモジュールのコミットで固定 | Git タグやコミットハッシュで固定 |
| **編集のしやすさ** | 直接編集してコミットできる | Packages 配下は読み取り専用 |
| **CI/CD 対応** | `--recurse-submodules` で取得可能 | UPM の解決が必要 |
| **学習コスト** | Git の基本操作で完結 | UPM のパッケージ仕様の理解が必要 |

サブモジュールの最大のメリットは、**普段の Git 操作だけで管理できる**ことと、**直接コードを編集してコミットできる**ことです。

## 注意点

サブモジュールを Unity プロジェクトで使う場合、いくつか注意すべき点があります。

### 1. メタファイル（.meta）も一緒にコミットする

Unity はアセットごとに `.meta` ファイルを生成し、その中に **GUID** が記録されています。
この GUID を使ってアセット間の参照を管理しているため、`.meta` ファイルもサブモジュールのリポジトリに含める必要があります。

```
# .gitignore で .meta を除外しないこと！
# ❌ *.meta  ← これはやらない
```

:::message alert
`.meta` ファイルをサブモジュール側に含めないと、プロジェクトごとに異なる GUID が割り当てられ、参照が壊れます。
:::

### 2. アセンブリ・ディフィニッションを定義する

サブモジュールのスクリプトが**メインプロジェクトのコードに依存しない**ように、アセンブリ・ディフィニッション（`.asmdef`）を定義しましょう。

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

これにより：
- サブモジュール側のコンパイルがメインプロジェクトから独立する
- 依存関係が明確になる
- コンパイル時間が短縮される

メインプロジェクト側からサブモジュールの機能を使いたい場合は、メインの `.asmdef` の `references` に `Utils` を追加します。

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

### ポイント

**(a) スクリプトフォルダの中にはシンボリックリンクだけを残す**

`Assets/Scripts/` の中にサブモジュールの実体を置くと、Unity が `.git` フォルダなどを認識してしまい問題が起きることがあります。
そのため、`Assets/Scripts/Utils` は**シンボリックリンク**にして、実体は別の場所に置きます。

**(b) 実際のサブモジュールはプロジェクトルートにインストールする**

サブモジュールの実体は `SubModules/` ディレクトリに配置します。
`Assets/` フォルダの外にあるため、Unity が余計なファイルをインポートしようとする心配がありません。

## セットアップ手順

### 1. サブモジュールを追加する

```bash
# プロジェクトルートで実行
git submodule add https://github.com/your-name/Utils.git SubModules/Utils
```

### 2. シンボリックリンクを作成する

```bash
# Assets/Scripts ディレクトリに移動
cd Assets/Scripts

# シンボリックリンクを作成
# macOS / Linux
ln -s ../../SubModules/Utils/Runtime Utils

# Windows (管理者権限が必要)
mklink /D Utils ..\..\SubModules\Utils\Runtime
```

:::message
Windows の場合、Git の設定で `core.symlinks = true` にしておく必要があります。
```bash
git config core.symlinks true
```
:::

### 3. .gitignore の設定

```gitignore
# シンボリックリンクの .meta は追跡する（除外しない）
# ただし、シンボリックリンク先の .meta はサブモジュール側で管理する
```

### 4. Unity で確認

Unity エディタを開くと、`Assets/Scripts/Utils` 配下にサブモジュールのスクリプトが表示されるはずです。
アセンブリ・ディフィニッションが正しく設定されていれば、コンパイルも通ります。

## 日常的な Git 操作

### サブモジュールを最新に更新する

```bash
# サブモジュールを最新のコミットに更新
git submodule update --remote

# メインプロジェクトでサブモジュールの変更をコミット
git add SubModules/Utils
git commit -m "Update Utils submodule"
```

### サブモジュール内で直接編集する

```bash
# サブモジュールのディレクトリに移動
cd SubModules/Utils

# ブランチに切り替え（デフォルトでは detached HEAD）
git checkout main

# 編集してコミット・プッシュ
git add .
git commit -m "Add new utility method"
git push

# メインプロジェクトに戻って更新をコミット
cd ../..
git add SubModules/Utils
git commit -m "Update Utils submodule"
```

### クローン時にサブモジュールも取得する

```bash
# 初回クローン時
git clone --recurse-submodules https://github.com/your-name/MyUnityProject.git

# すでにクローン済みの場合
git submodule update --init --recursive
```

## CI/CD での注意点

GitHub Actions などの CI 環境では、`checkout` 時にサブモジュールを取得する設定が必要です。

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

## 自分の Utils リポジトリの紹介

参考として、私が実際に使っている Utils リポジトリを公開しています。
主に以下のような汎用スクリプトを含んでいます：

- Transform / Vector の拡張メソッド
- シングルトンの基底クラス
- オブジェクトプールの汎用実装
- コルーチンのユーティリティ

このリポジトリをサブモジュールとして追加するだけで、新しい Unity プロジェクトですぐに使い始めることができます。

## まとめ

- Git サブモジュールは**普段の Git 操作だけで管理できる**ので学習コストが低い
- **直接編集してコミットできる**ので、開発中の改善がスムーズ
- `.meta` ファイルと `.asmdef` の管理を忘れずに
- **シンボリックリンク**を使えば、Unity のフォルダ構成をすっきり保てる
- UPM と比べてセットアップが簡単で、小〜中規模のチーム開発に向いている
