---
title: "ゲーム開発でよく使う C# Enum 便利パターン集"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "csharp", "ゲーム開発", "設計パターン"]
published: false
---

## はじめに

ゲーム開発では、アイテムの種類、キャラクターの状態、ステージのタイプなど、**固定の選択肢**を扱う場面が数多くあります。
こうした場面で活躍するのが C# の列挙型（Enum）です。

この記事では、Enum をただ定義するだけでなく、**ゲーム開発の現場で本当に役立つ便利なパターン**を紹介します。

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
- **int へのキャストは最小限に**: 型安全のメリットが失われる

```csharp
// ゲーム開発でよくある Enum の定義例
public enum ItemType
{
    None = 0,      // デフォルト値
    Weapon = 1,
    Armor = 2,
    Potion = 3,
    Material = 4,
    // 新しい値は末尾に追加する
}
```

## パターン1: 拡張メソッドで日本語名や情報を定義する

Enum に直接メソッドを追加することはできませんが、**拡張メソッド**を使えば Enum にメソッドを生やすことができます。

### 基本の拡張メソッド

```csharp
public enum EnemyType
{
    Slime,
    Goblin,
    Dragon,
    DarkKnight,
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
        EnemyType.DarkKnight => "暗黒騎士",
        _ => throw new ArgumentOutOfRangeException(nameof(type)),
    };

    /// <summary>
    /// 敵の基本HPを取得する
    /// </summary>
    public static int GetBaseHp(this EnemyType type) => type switch
    {
        EnemyType.Slime => 10,
        EnemyType.Goblin => 30,
        EnemyType.Dragon => 500,
        EnemyType.DarkKnight => 200,
        _ => throw new ArgumentOutOfRangeException(nameof(type)),
    };

    /// <summary>
    /// 飛行する敵かどうかを判定する
    /// </summary>
    public static bool IsFlying(this EnemyType type) => type switch
    {
        EnemyType.Dragon => true,
        _ => false,
    };
}
```

### 使い方

```csharp
var enemy = EnemyType.Dragon;

Debug.Log(enemy.ToJapaneseName()); // "ドラゴン"
Debug.Log(enemy.GetBaseHp());      // 500
Debug.Log(enemy.IsFlying());       // true
```

拡張メソッドを使うことで、Enum の値に紐づく情報をまとめて管理でき、呼び出し側も直感的に書けます。

## パターン2: スイッチ式を活用する

C# 8.0 以降で使えるスイッチ式（switch expression）を使うと、従来の `switch` 文よりも簡潔に書けます。

### 従来の書き方

```csharp
// 従来の switch 文
public static string GetDescription(ItemType type)
{
    switch (type)
    {
        case ItemType.Weapon:
            return "攻撃力を上げる装備";
        case ItemType.Armor:
            return "防御力を上げる装備";
        case ItemType.Potion:
            return "HPを回復するアイテム";
        case ItemType.Material:
            return "合成に使う素材";
        default:
            return "不明なアイテム";
    }
}
```

### スイッチ式を使った書き方

```csharp
// スイッチ式（C# 8.0 以降）
public static string GetDescription(ItemType type) => type switch
{
    ItemType.Weapon => "攻撃力を上げる装備",
    ItemType.Armor => "防御力を上げる装備",
    ItemType.Potion => "HPを回復するアイテム",
    ItemType.Material => "合成に使う素材",
    _ => "不明なアイテム",
};
```

スイッチ式は**式**なので、値を返すことに特化しており、`break` や `return` が不要です。

### 複数の条件をまとめる

```csharp
// 装備品かどうかを判定
public static bool IsEquipment(this ItemType type) => type switch
{
    ItemType.Weapon or ItemType.Armor => true,
    _ => false,
};
```

## パターン3: サマリーコメントで IDE の開発体験を向上させる

Rider や Visual Studio などの IDE では、サマリーコメント（`<summary>`）を書いておくと、**ホバー時にツールチップで説明が表示**されます。
これにより、コードを読まなくても各値の意味が分かるようになります。

```csharp
/// <summary>
/// ゲーム内のバフ効果の種類
/// </summary>
public enum BuffType
{
    /// <summary>
    /// 効果なし
    /// </summary>
    None = 0,

    /// <summary>
    /// 攻撃力が一定時間上昇する。重複時はスタックする。
    /// </summary>
    AttackUp = 1,

    /// <summary>
    /// 防御力が一定時間上昇する。重複時は効果時間が延長される。
    /// </summary>
    DefenseUp = 2,

    /// <summary>
    /// 移動速度が一定時間上昇する。重複しない（上書き）。
    /// </summary>
    SpeedUp = 3,

    /// <summary>
    /// 一定時間、毎秒HPが回復する。重複時はスタックする。
    /// </summary>
    Regeneration = 4,

    /// <summary>
    /// 一定時間、被ダメージを無効化する。重複しない（効果時間がリセットされる）。
    /// </summary>
    Invincible = 5,
}
```

### 日本語名もサマリーに書く場合

拡張メソッドを使わずに、サマリーコメントだけで日本語名を管理する方法もあります。

```csharp
public enum WeaponType
{
    /// <summary>剣 - 近距離攻撃。バランス型の武器。</summary>
    Sword,

    /// <summary>弓 - 遠距離攻撃。クリティカル率が高い。</summary>
    Bow,

    /// <summary>杖 - 魔法攻撃。MP消費で強力な範囲攻撃が可能。</summary>
    Staff,

    /// <summary>槍 - 中距離攻撃。リーチが長く、貫通攻撃が可能。</summary>
    Spear,
}
```

こうしておくと、IDE 上で `WeaponType.Sword` にカーソルを合わせるだけで「剣 - 近距離攻撃。バランス型の武器。」と表示されます。

## パターン4: Flags 属性で複数の状態を組み合わせる

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

## パターン5: Enum とディクショナリでマスターデータを管理する

小規模なマスターデータであれば、Enum をキーにしたディクショナリで管理するのもシンプルで便利です。

```csharp
public static class EnemyMaster
{
    // 敵のパラメータを定義する構造体
    public readonly struct EnemyParam
    {
        public readonly string Name;
        public readonly int Hp;
        public readonly int Attack;
        public readonly int Defense;

        public EnemyParam(string name, int hp, int attack, int defense)
        {
            Name = name;
            Hp = hp;
            Attack = attack;
            Defense = defense;
        }
    }

    // 敵のマスターデータ
    private static readonly Dictionary<EnemyType, EnemyParam> Data = new()
    {
        [EnemyType.Slime] = new("スライム", 10, 3, 1),
        [EnemyType.Goblin] = new("ゴブリン", 30, 8, 5),
        [EnemyType.Dragon] = new("ドラゴン", 500, 50, 30),
        [EnemyType.DarkKnight] = new("暗黒騎士", 200, 35, 25),
    };

    /// <summary>
    /// 敵のパラメータを取得する
    /// </summary>
    public static EnemyParam Get(EnemyType type) => Data[type];
}
```

## まとめ

| パターン | 用途 |
|---|---|
| 拡張メソッド | Enum に日本語名やパラメータを紐づける |
| スイッチ式 | 簡潔に値のマッピングを記述する |
| サマリーコメント | IDE 上で説明を表示する |
| Flags 属性 | 複数の状態を組み合わせる |
| ディクショナリ | 小規模マスターデータの管理 |

Enum は単純な機能ですが、工夫次第でゲーム開発の様々な場面で活用できます。
特に拡張メソッド + スイッチ式の組み合わせは、可読性が高く保守もしやすいのでおすすめです。
