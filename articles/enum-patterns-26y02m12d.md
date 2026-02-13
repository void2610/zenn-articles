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

Enumに直接メソッドを追加することはできませんが、**拡張メソッド**を使えばEnumに色々なメソッドを定義できます。
Enumを引数に取るヘルパーメソッドを定義することもできますが、拡張メソッドにすることで、呼び出し側のコードがより直感的になります。

ここで重要なのは、拡張メソッドで定義すべきなのは **Enumの値と1対1で対応する情報**だという点です。
日本語名や色など、そのEnum値に固有の属性が適しています。
HPや攻撃力のようなゲームバランスに関わるパラメータは、ScriptableObjectやマスターデータなど別の仕組みで管理するのが良いでしょう。

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
従来の`switch`文では以下のようになります。

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

スイッチ式は値を返すことに特化しており、`break` や `return` が不要です。
Enum の拡張メソッドとの相性が非常に良いので、積極的に活用すると良いでしょう。

## パターン2: サマリーコメントでIDEでの開発体験を向上させる

RiderをはじめとするIDEでは、サマリーコメント（`<summary>`）を書いておくと、**ホバー時にツールチップで説明が表示**されます。
これにより、コードを読まなくても各値の意味が分かるようになります。
これはEnumが難しい英単語になってしまっていたり、Enumでの動作を説明しておく必要があったりする場合に特に有効です。

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

## パターン4: 全値ループで UI を生成する

`Enum.GetValues()` を使うと、Enum のすべての値をループできます。
設定画面のドロップダウンやデバッグ用リスト、図鑑の全項目表示など、**Enum の全値を一覧表示したい場面**で便利です。

```csharp
// 全 Enum 値をループしてドロップダウンの選択肢を生成
foreach (EnemyType type in Enum.GetValues(typeof(EnemyType)))
{
    dropdown.options.Add(new TMP_Dropdown.OptionData(type.ToString()));
}
```

### パターン1と組み合わせる

パターン1の拡張メソッドと組み合わせると、日本語名でのドロップダウン生成が簡潔に書けます。

```csharp
// 拡張メソッドと組み合わせて日本語名でドロップダウンを生成
foreach (EnemyType type in Enum.GetValues(typeof(EnemyType)))
{
    dropdown.options.Add(new TMP_Dropdown.OptionData(type.ToJapaneseName()));
}
```

ドロップダウンの選択肢を手動で追加・管理する必要がなくなり、Enum に新しい値を追加するだけで UI にも自動的に反映されます。

## パターン5: カスタム属性でメタデータを付与する

Enum の各値にカスタム属性（Attribute）を付与し、リフレクションで取得するパターンです。
アイテム情報やスキル情報など、**Enum 値に複数のメタデータをまとめて宣言的に定義したい場合**に便利です。

### カスタム属性を定義する

```csharp
[AttributeUsage(AttributeTargets.Field)]
public class ItemInfoAttribute : Attribute
{
    public string DisplayName { get; }
    public int MaxStack { get; }

    public ItemInfoAttribute(string displayName, int maxStack)
    {
        DisplayName = displayName;
        MaxStack = maxStack;
    }
}
```

### Enum に属性を付与する

```csharp
public enum ItemType
{
    [ItemInfo("回復薬", 99)]
    HealthPotion,

    [ItemInfo("毒消し", 99)]
    Antidote,

    [ItemInfo("伝説の剣", 1)]
    LegendarySword,
}
```

### リフレクションで属性を取得する

```csharp
public static class EnumAttributeHelper
{
    /// <summary>
    /// Enum 値に付与されたカスタム属性を取得する
    /// </summary>
    public static T GetAttribute<T>(this Enum value) where T : Attribute
    {
        var field = value.GetType().GetField(value.ToString());
        return field?.GetCustomAttribute<T>();
    }
}

// 使い方
var info = ItemType.HealthPotion.GetAttribute<ItemInfoAttribute>();
Debug.Log(info.DisplayName); // "回復薬"
Debug.Log(info.MaxStack);    // 99
```

### パターン1（拡張メソッド）との使い分け

| | 拡張メソッド | カスタム属性 |
|---|---|---|
| 向いている用途 | 処理ロジック（変換、判定など） | データの宣言的な付与 |
| 定義場所 | 別クラスに記述 | Enum 定義に直接記述 |
| パフォーマンス | 高速（直接呼び出し） | リフレクションのコストあり |

カスタム属性はデータを Enum 定義の近くにまとめて書ける利点がありますが、リフレクションによる取得にはコストがかかります。
頻繁にアクセスする場合は、起動時に Dictionary にキャッシュしておくと良いでしょう。

## まとめ

| パターン | 用途 |
|---|---|
| 拡張メソッド + スイッチ式 | Enum に日本語名やイメージカラーなど1対1の情報を紐づける |
| サマリーコメント | IDE 上で説明を表示する |
| Flags 属性 | 複数の状態を組み合わせる |
| 全値ループ | ドロップダウンやリストを Enum から自動生成する |
| カスタム属性 | Enum 値にメタデータを宣言的に付与する |

Enum は単純な機能ですが、工夫次第でゲーム開発の様々な場面で活用できます。
拡張メソッドや全値ループ、カスタム属性などを組み合わせることで、保守性と可読性の高いコードが書けます。

## 宣伝

よければウィッシュリストに追加していただけると嬉しいです！

### VOID RED
「あなただけ」の記憶構築ADV。記憶を失った少女が記憶を取り戻すためオークションに挑む。
https://store.steampowered.com/app/3997140/

### 庭小人の庭
植物育成とターン制バトルを組み合わせたデッキ構築型ローグライクゲーム。種から育てたカードで戦略的に戦おう。
https://store.steampowered.com/app/4105720/
