# 第1部：基礎

> 共通テンプレート・凡例は [README](./README.md) を参照。

---

## 1-00 最初 (Firsts) — 導入

**—（コードなし）** Babylon Lite は WebGPU 専用。`createEngine → createSceneContext → …build… → registerScene → startEngine` という流れで 1 シーンを構築します。Babylon.js の「クラス＋コンストラクタ」に対し、Lite は「**ファクトリ関数＋プレーンデータ＋明示的な `addToScene`**」が基本です。

---

## 1-01 ハローワールド！最初のシーンとモデル — ○

**目的**：シーン・カメラ・ライト・球・地面を作る。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createSphere, createGround, createStandardMaterial`

```typescript
// カメラ（キャラ後方視点に合わせて alpha = -π/2）
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 4, { x: 0, y: 1, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);

// ライト
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));

// 球 —— ★マテリアルの明示割り当てが必須（無いと描画されない）
const sphere = createSphere(engine, { diameter: 2, segments: 16 });
sphere.position.y = 1;
const sphereMat = createStandardMaterial();
sphereMat.diffuseColor = [0.8, 0.8, 0.8];   // 色は [r, g, b] 配列
sphere.material = sphereMat;
addToScene(scene, sphere);

// 地面 —— 同様にマテリアルを割り当て
const ground = createGround(engine, { width: 6, height: 6 });
const groundMat = createStandardMaterial();
groundMat.diffuseColor = [0.5, 0.55, 0.5];
ground.material = groundMat;
addToScene(scene, ground);
```

> ⚠️ **マテリアル必須**：Lite にはデフォルトマテリアルが無く、`material` 未設定のメッシュは
> `addToScene` されても**描画されません（エラーも出ません）**。プリミティブには必ず
> `createStandardMaterial()` / `createPbrMaterial()` を割り当ててください（→ [README の落とし穴](./README.md#重要な落とし穴共通)）。
>
> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/0
>
> `position` などは `ObservableVec3`。**成分（`.x/.y/.z`）で代入**します（丸ごと別オブジェクトを代入すると dirty 追跡が切れます）。

---

## 1-02 あなたの web サイトをかっこよくしよう — △

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

## 1-03 モデルを扱う — ○

**目的**：外部 3D モデルを読み込んで操作する。

追加 import：`createDefaultCamera, attachControl, createHemisphericLight, loadGltf`

```typescript
addToScene(scene, await loadGltf(engine, "https://playground.babylonjs.com/scenes/BoomBox.glb"));

const camera = createDefaultCamera(scene);   // auto-framing
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));
```

- `loadGltf` は **glTF / GLB** を読み込み `AssetContainer` を返す → `addToScene` で登録。
- Babylon.js チュートリアルの `.babylon`（例：Dude.babylon）は **スキン/アニメ非対応**なので、アニメ付きモデルは **glTF に変換**するか glTF モデル（例：`Xbot.glb`）を使います（→ [3-06](./03-animation.md)）。

---

## 1-04 最初の web アプリのセットアップ — ○

**目的**：Playground でなく実プロジェクトとして構築する。

Lite は **TypeScript / Vite ネイティブ**。エンジン側の制約はなく、パッケージ名が変わるだけです。

```bash
npm create vite@latest my-app -- --template vanilla-ts
cd my-app
npm install @babylonjs/lite
```

```typescript
// src/main.ts … 共通テンプレートと同じ（import 元は "@babylonjs/lite"）
```

`index.html` に `<canvas id="renderCanvas"></canvas>` を置けば、Playground と同じコードがそのまま動きます。
