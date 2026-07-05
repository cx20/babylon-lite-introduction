# 第2部：村の構築

> 共通テンプレート・凡例は [README](./README.md) を参照。
> 以降のサンプルはカメラ・ライトを省略していることがあります。1-01 の骨組みに足してください。

---

## 2-00 村を作る — 導入

**—** 地面 → メッシュ → テクスチャ → コピー、と積み上げて村を作ります。

---

## 2-01 地面 (Grounding) — ○

追加 import：`createGround, createStandardMaterial, loadTexture2D`

```typescript
const ground = createGround(engine, { width: 24, height: 24 });
const groundMat = createStandardMaterial();
groundMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/grass.png");
ground.material = groundMat;
addToScene(scene, ground);
```

---

## 2-02 サウンドを追加 — ✕

**Lite には音声エンジンがありません**（`KHR_audio` も非対応）。どうしても鳴らす場合は、エンジン外で **Web Audio API を直接**使います。

```typescript
// エンジン機能ではない代替（ブラウザ標準）
const audio = new Audio("https://playground.babylonjs.com/sounds/cellolong.wav");
audio.loop = true;
addEventListener("pointerdown", () => audio.play(), { once: true }); // 自動再生制限のためユーザー操作後に
```

> 3D 定位（`BABYLON.Sound` の spatial）に相当する機能はありません。

---

## 2-03 メッシュを設置 (Place and Scale) — ○

追加 import：`createBox`

```typescript
const box = createBox(engine, { size: 1 });
box.position.x = 2;
box.position.y = 0.5;   // 地面に載せる
box.scaling.x = 2;      // 横に伸ばす
addToScene(scene, box);
```

---

## 2-04 基本的な家 (A Basic House) — ○

追加 import：`createBox, createStandardMaterial, loadTexture2D`

```typescript
const house = createBox(engine, { width: 1, height: 1, depth: 1 });
house.position.y = 0.5;
const mat = createStandardMaterial();
mat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/floor.png");
house.material = mat;
addToScene(scene, house);
```

> **面ごとに別画像**（ドア面・窓面など、Babylon.js の `faceUV`）は box 生成器では未対応の可能性があります → [2-06](#2-06-マテリアル面ごと--) 参照。単一テクスチャを貼るだけならこの通りで OK です。

---

## 2-05 テクスチャを貼る (Add Texture) — ○

追加 import：`loadTexture2D`

```typescript
const mat = createStandardMaterial();
mat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/bricktile.jpg");
// UV スケール/オフセットは対応（フィールド名は IntelliSense で確認：要確認）
// mat.diffuseTexture.uScale = 2; mat.diffuseTexture.vScale = 2;
house.material = mat;
```

Lite は「UV Scaling / Offset」「Bump / Normal」「Emissive」など主要テクスチャに対応しています。

---

## 2-06 マテリアル（面ごと/faceUV） — △

**目的**：家の面ごとに異なる画像（正面＝ドア、側面＝窓）を貼る。

box 生成器の `faceUV` 相当は未確認です（faceUV は cylinder / polyhedron 生成器には存在）。代替は次のいずれか：

1. **面ごとに別プレーンを貼る**（`createPlane` を 6 枚、各面に別マテリアル）
2. **マルチマテリアル**（`.babylon` ローダー経由なら submesh + multiMaterial に対応）
3. 面ごとの UV を持つ **カスタムメッシュ / ShaderMaterial**

```typescript
// 代替例：正面だけドア画像のプレーンを重ねる
// 追加 import: createPlane, loadTexture2D, createStandardMaterial
const door = createPlane(engine, { width: 1, height: 1 });   // 要確認: createPlane の有無/引数
door.position.z = -0.51;
const doorMat = createStandardMaterial();
doorMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/door.png");
door.material = doorMat;
addToScene(scene, door);
```

---

## 2-07 リファクタリング（関数化） — ○

**目的**：家を作る処理を関数にまとめる。純粋な TypeScript なのでそのまま可能。

```typescript
async function createHouse(engine, scene, x: number, z: number) {
  const house = createBox(engine, { size: 1 });
  house.position.x = x; house.position.y = 0.5; house.position.z = z;
  const mat = createStandardMaterial();
  mat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/floor.png");
  house.material = mat;
  addToScene(scene, house);
  return house;
}

await createHouse(engine, scene, -2, 0);
await createHouse(engine, scene,  2, 0);
```

---

## 2-08 メッシュを結合 (Combining Meshes) — △

Babylon.js の `Mesh.MergeMeshes` 相当のヘルパーは **ありません**。目的（ドローコール削減 / 一体化）別の代替：

- **同一メッシュを大量配置** → thin instances（[2-09](#2-09-メッシュをコピー-copying-meshes--)）
- **ブーリアンで一体化** → `CSG2`（union / subtract / intersect、Scenes 90-91）
- どうしても頂点結合が必要なら手動で頂点バッファを結合

---

## 2-09 メッシュをコピー (Copying Meshes) — ○

**目的**：家を大量に複製する。Babylon.js の `createInstance()` は Lite では **thin instances** に置き換えます。

追加 import：`createBox, setThinInstances`

```typescript
// 列ごとに家を並べる（各インスタンスは 16 float の列優先 4x4 行列）
function translation(x: number, y: number, z: number): Float32Array {
  const m = new Float32Array(16);
  m[0] = 1; m[5] = 1; m[10] = 1; m[15] = 1;
  m[12] = x; m[13] = y; m[14] = z;
  return m;
}

const house = createBox(engine, { size: 1 });
house.material = createStandardMaterial();

const N = 8;
const data = new Float32Array(N * 16);
for (let i = 0; i < N; i++) data.set(translation(i * 2 - 7, 0.5, 4), i * 16);
setThinInstances(house, data, N);
addToScene(scene, house);
```

> Babylon.js の `InstancedMesh` API（🚫）は非対応。thin instances（`setThinInstances` / `setThinInstanceColors`）を使います。

---

## 2-10 Viewer のカメラ変更 — ○

**目的**：カメラの種類・パラメータを変える。

追加 import：`createArcRotateCamera, createFreeCamera, attachControl, attachFreeControl`

```typescript
// オービット（周回）カメラ
const arc = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 8, { x: 0, y: 1, z: 0 });
scene.camera = arc;
attachControl(arc, canvas, scene);

// ── もしくは一人称（WASD）カメラ ──
// const free = createFreeCamera({ x: 0, y: 2, z: -8 }, { x: 0, y: 1, z: 0 });
// scene.camera = free;
// attachFreeControl(free, canvas, scene);
```

`FollowCamera` は未対応（—）。追従は [3-07](./03-animation.md) のように毎フレーム target を更新するか、カメラをキャラに `parent` して実現します。

---

## 2-11 Web アプリ レイアウト — ○

**目的**：3D canvas を Web ページ内にレイアウトする。HTML/CSS の話でエンジン制約なし。

```html
<div class="app">
  <header>My Village</header>
  <canvas id="renderCanvas"></canvas>
</div>
<style>
  .app { display: grid; grid-template-rows: auto 1fr; height: 100vh; }
  #renderCanvas { width: 100%; height: 100%; display: block; }
</style>
```
