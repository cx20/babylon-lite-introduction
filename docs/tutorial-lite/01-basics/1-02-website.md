# 1-02 あなたの web サイトをかっこよくしよう — △

> [第1部：基礎](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：3D モデルを Web ページに埋め込む。

Babylon.js の `<babylon-viewer>` Web コンポーネント（ゼロコード埋め込み）は Lite にありません。代わりに **canvas を置いて `loadGltf` するだけ**で同等の見せ方になります。

```html
<canvas id="renderCanvas" style="width:100%; height:480px"></canvas>
<script type="module" src="./main.js"></script>
```

```typescript
// 追加 import: createDefaultCamera, attachControl, createHemisphericLight, loadGltf
addToScene(scene, await loadGltf(engine, "https://playground.babylonjs.com/scenes/BoomBox.glb"));
const camera = createDefaultCamera(scene);   // 読み込んだモデルを自動フレーミング
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));
```

> ブログ等への埋め込みは [Lite Playground の embed 機能](https://doc.babylonjs.com/lite/04-playground)（`?embed=runner` の iframe ＋ `postMessage`）も使えます。

---

← [1-01 ハローワールド！最初のシーンとモデル](./1-01-hello-world.md) ・ [1-03 モデルを扱う](./1-03-models.md) →
