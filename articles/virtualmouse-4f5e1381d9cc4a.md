---
title: "[Unity, InputSystem] 仮想マウスの実装で詰まった点"
emoji: "🐭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Unity", "InputSystem", "csharp"]
published: true
---

## 背景

現在 Unity で制作中のゲームでコントローラーからマウスを動かすために仮想マウスを実装する際に、日本語の情報があまり存在せず詰まってしまったので、問題点と解決方法を共有します。

:::message
本記事の内容は、`M1 Macbook Pro`, `Unity6 6000.0.42f1`, `Input System 1.13.1 · February 18, 2025`で動作確認をしています。
異なる環境、バージョンでは動作が変わる可能性もありますのでご了承ください。
:::

## 基本的な仮想マウスの実装方法

InputSystem は`VirtualMouseInput`というクラスを提供しており、`VirtualMouse`コンポーネントを任意のオブジェクトにアタッチし、いくつかの設定をすることで簡単に仮想マウスを実装することができます。

基本的な仮想マウスの実装方法は以下の記事を参考にするのがいいと思います。
https://nekojara.city/unity-input-system-virtual-mouse

## VirtualMouseInput の問題点

上の記事でも触れられていますが、公式が提供している`VirtualMouseInput`クラスには以下のような問題点があります。

1. `CanvasScaler`の`UI Scale Mode`が`Constant Pixel`以外の場合、仮想マウスの位置が正しく表示されない。

2. 仮想マウスの有効/無効化が行えない。

3. `InputAction`の`ActionType`を`PassThrough`にする必要がある。

## 問題点の解決方法

### 1. `CanvasScaler`の`UI Scale Mode`が`Constant Pixel`以外の場合、仮想マウスの位置が正しく表示されない。

この点は[Unity のフォーラム](https://discussions.unity.com/t/cant-interact-with-ui-when-using-a-virtual-mouse/813075)でも議論されていた内容で、`VirtualMouseInput`内の座標を更新する部分を修正する必要があります。

以下に変更点を示します。

```diff cs:VirtualMouseInput.cs
@@ class VirtualMouseInput : MonoBehaviour
        private void UpdateMotion()
        {
+            // キャンバスを取得できていない場合はキャンバスを探す
+           if (m_Canvas == null)
+               TryFindCanvas();
            if (m_VirtualMouse == null)
                return;
@@ private void UpdateMotion()
            // Update software cursor transform, if any.
            if (m_CursorTransform != null &&
                (m_CursorMode == CursorMode.SoftwareCursor ||
                    (m_CursorMode == CursorMode.HardwareCursorIfAvailable && m_SystemMouse == null)))
-               m_CursorTransform.position = newPosition;
+               {
+                   // 仮想マウスのスクリーン座標をキャンバスのローカル座標に変換
+                   Vector2 localPos;
+                   RectTransformUtility.ScreenPointToLocalPointInRectangle(
+                       m_CursorTransform.parent as RectTransform,
+                       newPosition,
+                       m_Canvas.worldCamera,
+                       out localPos);
+                   m_CursorTransform.anchoredPosition = localPos;
+               }
```

また、この記事の最後に、私が実際に使用している`MyVirtualMouseInput`クラスのコードを載せておきますので、参考にしてください。

:::message
`VirtualMouseInput`をそのままコピーしても動作しない部分があったので、不要だと思われる部分を数箇所削除しています。
(チュートリアルへのリンクやエディタ専用のディレクティブなど)
:::

### 2. 仮想マウスの有効/無効化が行えない。

この点も Unity のフォーラムで議論されていた内容で、色々なアプローチがあるようですが、いくつかの方法を試してもクリックやマウスホバーの判定が残ってしまっていました。
私は、`VirtualMouseInput`に`isActive`というフィールドを追加し、`isActive`が`false`の時は仮想マウスの座標を`-9999, -9999`に固定するようにしました。

以下に変更点を示します。

```diff cs:VirtualMouseInput.cs
@@ class VirtualMouseInput : MonoBehaviour
+       public bool isActive = true;

@@ class VirtualMouseInput : MonoBehaviour
        private void UpdateMotion()
        {
+            // 無効状態なら画面外に固定して早期リターンする
+            if (!isActive)
+            {
+                Vector2 offScreenPos = new Vector2(-9999, -9999);
+                InputState.Change(m_VirtualMouse.position, offScreenPos);
+                if (m_CursorTransform != null)
+                    m_CursorTransform.anchoredPosition = offScreenPos;
+                return;
+            }
```

実際に使用する際は、`VirtualMouseInput`の`isActive`と、設定している`Image`コンポーネントの表示/非表示を切り替えることで仮想マウスの有効/無効化を行うことができます。

### 3. `InputAction`の`ActionType`を`PassThrough`にする必要がある。

これは私の単純な確認不足で、`VirtualMouseInput`に渡す`InputAction`の`ActionType`を`PassThrough`にする必要があるようでした。
実際に、`InputSystem`を追加した際に自動生成される`InputActionAsset`内の UI 関連の`ActionType`は`PassThrough`になっていました。(新しい`InputAction`を作成すると`Value`になりますが)
この点の確認不足で仮想マウスをうまく動かすことができなかったので、困っている方は確認してみてください。

:::message
この`ActionType`が`PassThrough`でない場合に仮想マウスを操作できないという動作は、新規プロジェクトで再現を試みたところ、再現できませんでした。
そのためこの動作は、私のプロジェクトの設定や他のスクリプトが影響している可能性があります。
:::

## MyVirtualMouseInput の内容

以下が私が実際に使用している`MyVirtualMouseInput`クラスのコードです。
このクラスを任意のオブジェクトにアタッチし、通常の`VirtualMouseInput`と同じように設定して下さい。
:::details MyVirtualMouseInput.cs
@[gist](https://gist.github.com/void2610/5138920294f4204b4d67bea31a4e854f)
:::
