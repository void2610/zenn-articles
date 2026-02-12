---
title: "ゲーム開発でよく使う C# Enum 便利パターン集"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "ゲーム開発", "設計パターン"]
published: false
---

## はじめに

ゲーム開発では、アイテム種類、キャラクター状態、ステージタイプなど、**固定の選択肢**を扱う場面が数多くあります。
こうした場面で活躍するのが C# の列挙型（Enum）です。

この記事では、Enum をただ定義するところから一歩進んだ、ゲーム開発で役立つ便利なパターンを紹介します。

## そもそもなぜ Enum を使うのか

### マジックナンバーの排除

```csharp
// Bad: 何を表しているのか分からない
if (state == 0) { /* ... */ }
if (state == 1) { /* ... */ }

// Good: 意味が明確
if (state == CharacterState.Idle) { /* ... */ }
if (state == CharacterState.Running) { /* ... */ }
```

### Enum を使うメリット

- **型安全**: 無効な値を代入できない
- **可読性**: コードの意図が明確になる
- **IDE サポート**: 補完が効き、タイプミスを防げる
- **switch の網羅性チェック**: すべてのケースを処理しているか検出できる

### Enum を使う際の注意点

- **値の追加は末尾に**: 途中に値を追加すると、シリアライズされたデータとの整合性が壊れる
- **0 には "None" や "Unknown" を**: デフォルト値として安全に使える
- **intやstringへのキャストは最小限に**: 型安全のメリットが失われる

## パターン1: 拡張メソッドを定義する

Enum に直接メソッドを追加することはできませんが、**拡張メソッド**を使えば Enum に色々なメソッドを定義できます。
Enumを引数に取るヘルパーメソッドを定義することもできますが、拡張メソッドにすることで、呼び出し側のコードが直感的になります。

ここで重要なのは、拡張メソッドで定義すべきなのは **Enum の値と1対1で対応する情報**だという点です。
日本語名や色など、そのEnum値に固有の属性が適しています。
HP や攻撃力のようなゲームバランスに関わるパラメータは、ScriptableObject やマスターデータなど別の仕組みで管理するのが良いでしょう。

### 基本の拡張メソッド

```csharp
public enum EnemyType
{
    Slime,
    Goblin,
    Dragon,
}

public static class EnemyTypeExtensions
{
    /// <summary>
    /// 敵の日本語名を取得する
    /// </summary>
    public static string ToJapaneseName(this EnemyType type) => type switch
    {
        EnemyType.Slime => "スライム",
        EnemyType.Goblin => "ゴブリン",
        EnemyType.Dragon => "ドラゴン",
        _ => throw new ArgumentOutOfRangeException(nameof(type)),
    };

    /// <summary>
    /// 敵のイメージカラーを取得する
    /// </summary>
    public static Color GetColor(this EnemyType type) => type switch
    {
        EnemyType.Slime => Color.blue,
        EnemyType.Goblin => Color.green,
        EnemyType.Dragon => Color.red,
        _ => throw new ArgumentOutOfRangeException(nameof(type)),
    };
}
```

### 使い方

```csharp
var enemy = EnemyType.Dragon;
Debug.Log(enemy.ToJapaneseName()); // "ドラゴン"
Debug.Log(enemy.GetColor());       // RGBA(1.000, 0.000, 0.000, 1.000)
```

### スイッチ式について

上記のコード例では C# 8.0 以降で使える**スイッチ式（switch expression）**を使っています。
従来の `switch` 文と比較してみましょう。

```csharp
// 従来の switch 文
public static string ToJapaneseName(this EnemyType type)
{
    switch (type)
    {
        case EnemyType.Slime:
            return "スライム";
        case EnemyType.Goblin:
            return "ゴブリン";
        case EnemyType.Dragon:
            return "ドラゴン";
        case EnemyType.DarkKnight:
            return "暗黒騎士";
        default:
            throw new ArgumentOutOfRangeException(nameof(type));
    }
}
```

スイッチ式は**式**なので、値を返すことに特化しており、`break` や `return` が不要です。
Enum の拡張メソッドとの相性が非常に良いので、積極的に活用しましょう。

また、複数の条件をまとめることもできます。

```csharp
// 装備品かどうかを判定
public static bool IsEquipment(this ItemType type) => type switch
{
    ItemType.Weapon or ItemType.Armor => true,
    _ => false,
};
```

## パターン2: サマリーコメントで IDE の開発体験を向上させる

Rider や Visual Studio などの IDE では、サマリーコメント（`<summary>`）を書いておくと、**ホバー時にツールチップで説明が表示**されます。
これにより、コードを読まなくても各値の意味が分かるようになります。

![Riderでサマリーコメントがツールチップ表示される例](/images/enum-patterns/rider-summary-tooltip.png)
*Rider でサマリーコメントがツールチップとして表示される*

```csharp
/// <summary> ゲーム内のバフ効果の種類 </summary>
public enum BuffType
{
    /// <summary> 効果なし </summary>
    None = 0,
    /// <summary> 攻撃力上昇 </summary>
    AttackUp = 1,
    /// <summary> 防御力上昇 </summary>
    DefenseUp = 2,
    /// <summary> 移動速度上昇 </summary>
    SpeedUp = 3,
    /// <summary> HP自動回復 </summary>
    Regeneration = 4,
    /// <summary> 無敵 </summary>
    Invincible = 5,
}
```

## パターン3: Flags 属性で複数の状態を組み合わせる

ゲーム開発では、キャラクターが複数の状態を同時に持つことがあります。`[Flags]` 属性を使えば、ビット演算で複数の状態を組み合わせられます。

```csharp
[Flags]
public enum StatusEffect
{
    None      = 0,
    Poison    = 1 << 0,  // 0001
    Burn      = 1 << 1,  // 0010
    Freeze    = 1 << 2,  // 0100
    Paralysis = 1 << 3,  // 1000
}
```

### 使い方

```csharp
// 複数の状態異常を付与
var effects = StatusEffect.Poison | StatusEffect.Burn;

// 特定の状態異常を持っているか判定
if (effects.HasFlag(StatusEffect.Poison))
{
    Debug.Log("毒状態です");
}

// 状態異常を解除
effects &= ~StatusEffect.Poison;
```

## まとめ

| パターン | 用途 |
|---|---|
| 拡張メソッド + スイッチ式 | Enum に日本語名やイメージカラーなど1対1の情報を紐づける |
| サマリーコメント | IDE 上で説明を表示する |
| Flags 属性 | 複数の状態を組み合わせる |

Enum は単純な機能ですが、工夫次第でゲーム開発の様々な場面で活用できます。
特に拡張メソッド + スイッチ式の組み合わせは、可読性が高く保守もしやすいのでおすすめです。

## 宣伝

よければウィッシュリストに追加していただけると嬉しいです！

### VOID RED
「あなただけ」の記憶構築ADV。記憶を失った少女が記憶を取り戻すためオークションに挑む。
https://store.steampowered.com/app/3997140/

### 庭小人の庭
植物育成とターン制バトルを組み合わせたデッキ構築型ローグライクゲーム。種から育てたカードで戦略的に戦おう。
https://store.steampowered.com/app/4105720/
