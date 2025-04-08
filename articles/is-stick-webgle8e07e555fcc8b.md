---
title: "[Unity, InputSystem] WebGLでスティックの上下操作が効かなくなった時の対処法"
emoji: "🕹️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Unity, InputSystem, webgl]
published: true
---

## 問題点

Unity で InputSystem を使ってコントローラー操作を実装していた時に、エディタ上では正しく動作するのに、WebGL ビルドをした際にスティックの上下操作が効かなくなってしまいました。
ただ上下操作が効かないだけではなく、スティックの左右操作やコントローラー十字ボタンでの操作は正常に動作していました。

## 対処法

この現象の原因は、以下のように`Vector2`の`InputAction`において、コントローラーのスティックをバインディングする際に`Composite`を使っていることにありました。

以下の画像のように、本来、`Left Stick[Gamepad]`をバインディングするだけでいいのに、`Composite`を使用したバインディングをしてしまっていました。

![](/images/is-stick-webgl/1.png)
_間違ったバインディング_

以下の画像のように、`Composite`を使用せずに、`Left Stick[Gamepad]`をバインディングすることで、WebGL ビルドでも正常に動作するようになりました。

![](/images/is-stick-webgl/2.png)
_正しいバインディング_

キーボードの場合は、WASD や矢印キーを`Composite`でバインディングする必要がありますが、コントローラーのスティックは`Composite`を使用しないようにしましょう。
