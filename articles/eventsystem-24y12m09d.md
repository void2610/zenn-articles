---
title: "[Unity, R3]ローグライクゲームのレリックを実装する"
emoji: "⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "R3", "UniRx", "csharp"]
published: true
---

## 背景
私はローグライクゲームをよく遊んでおり、実際に自分でローグライクゲームを作る際に、PeglinやSlay the Spireなどのようなレリックの効果をどのように実装するべきか悩みました。
この際にR3(旧UniRx)というライブラリを使えばレリックの複雑な相互作用を比較的綺麗に実装できるのではないかと思い、実装してみました。
(R3(UniRx)に関しては既に色々な解説記事があるので、解説は省略します。)

:::message
この記事で紹介するのはあくまでR3を初めて触った自分が実装してみたものです。
この方法が絶対に良いというわけではありませんし、他にいい実装法をご存知の方が居られましたら是非コメントなどで教えて頂きたいです！🙇
:::

## 実装のイメージ
### 概要
ゲーム内のレリックが発動しそうなタイミング(プレイヤーが攻撃した時、コインを入手した時等)に、`値を紐付けたイベント`を発行します。
この際に`イベントに紐付けられたレリック`の処理を呼び出し、`紐づけられた値`を任意に上書きし、上書き後の値をゲーム進行に使うという流れです。

### 処理の流れ
以下に、`「プレイヤーの体力が8割以上なら獲得するコイン量が2倍になる」`というレリックの効果が発動する流れを説明します。

1.`OnCoinGain`イベントを定義する
2.レリック処理を定義し、`OnCoinGain`イベントに紐付ける
3.コインを2枚入手
3.`OnCoinGain`イベントが`Value=2`で発行される
4.レリック処理が発動し、条件を満たしていたら`OnCoinGain`イベントの`Value`を2倍にする
5.`OnCoinGain`イベントの`Value`を参照し、実際にコインを4枚獲得
6.`OnCoinGain`イベントの`Value`をリセットする

## コード
以下に実際のコードを示します
### GameEvent<T>
:::details コード全文
@[gist](https://gist.github.com/void2610/017cfc4e2f9540b27a0600d9978bfd12)
:::
### IRelicBehavior
:::details コード全文
@[gist](https://gist.github.com/void2610/b1890735525c0efcbd2c87204108f668)
:::
### EventManager
:::details コード全文
@[gist](https://gist.github.com/void2610/21eee3fc9904fd7ed2b77a5faf809a8b)
:::
### レリック
:::details コード全文
@[gist](https://gist.github.com/void2610/3197f9ef37099877786151f032d82ac3)
:::
### コイン獲得処理
:::details コード全文
@[gist](https://gist.github.com/void2610/4a668ccb5c881180184c61114e6c6574)
:::

:::message
`EventManager`内で、`ResetEventManager()`メソッドで手動でイベントを初期化していますが、これはUnityエディターでPlay Modeに入るまでの時間を短縮するため、Domainのリロードを無効化しており、staticな変数が初期化されないためです。
詳しくは以下のUnity公式の動画をご覧ください。

[【Unity】開発効率アップ！ Enter Play Mode オプション解説](https://www.youtube.com/watch?v=ThoWjnNR6F4)
:::

## 使用方法
`EventManager`に任意のイベントを定義し、任意のレリックの`IRelicBehavior.ApplyEffect()`内でレリックの処理をするメソッドをイベントの`GameEvent<T>.Subscribe()`を使って登録します。
任意のタイミングで、`GameEvent<T>.Trigger()`を呼び出すことでレリックの処理が発動するので、その後に`GameEvent<T>.GetValue()`で値を読み取ってゲーム進行に使います。

## 最後に
冒頭でも書きましたが、この実装はあくまでR3を初めて触った自分が思いつきで実装してみたものです。
他にいい実装法をご存知の方が居られましたら是非コメントなどで教えて頂きたいです！🙇

