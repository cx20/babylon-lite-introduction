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

**目的**：メッシュの位置を調整して地面の上に載せる。`createBox` は既定で**中心が原点**に生成されるため、
`position.y` を指定しないと**半分が地面にめり込みます**。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createGround, createStandardMaterial`

> ⚠️ **`createBox` の第 2 引数は数値**（一辺の長さ）です。Babylon.js の `MeshBuilder.CreateBox("box", {})` に
> 引きずられて `createBox(engine, { size: 1 })` のようにオプションオブジェクトを渡すと、
> 頂点が `NaN` 化して**無言で描画されなくなります**。既定値（1）でよければ `createBox(engine)` でも構いません。

### A. 位置を指定しない場合（めり込む）

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

// box の原点は中心。position.y を指定しないと高さ1の半分（0.5）が地面に埋まる
const box = createBox(engine, 1);
box.material = createStandardMaterial();   // ★マテリアル必須（無いと描画されない）
addToScene(scene, box);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/2

### B. 地面の上に載せる（`position.y` を指定）

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

const box = createBox(engine, 1);
box.material = createStandardMaterial();
box.position.y = 0.5;   // 高さの半分だけ上げて地面に載せる
addToScene(scene, box);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/3
>
> Babylon.js と同じく **原点はメッシュの中心**。`position` は `ObservableVec3` ですが、
> `box.position.y = 0.5` のように成分へ直接代入できます（[README の落とし穴](./README.md#重要な落とし穴共通)）。

### C. 横に伸ばす（スケール）

```typescript
box.position.x = 2;
box.scaling.x = 2;      // 横に伸ばす
```

---

## 2-04 基本的な家 (A Basic House) — ○

**目的**：box を家体、cylinder（`tessellation: 3` の三角柱）を屋根にして組み合わせる。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createCylinder, createGround, createStandardMaterial`

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

// 家体 —— サイズ1の box を接地
const house = createBox(engine, 1);   // createBox の第2引数は数値（一辺の長さ）
house.material = createStandardMaterial();
house.position.y = 0.5;
addToScene(scene, house);

// 屋根 —— tessellation: 3 の三角柱シリンダーを横倒しにして載せる
const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
roof.material = createStandardMaterial();
roof.scaling.x = 0.75;
roof.rotation.z = Math.PI / 2;   // rotation（Euler）は rotationQuaternion への代入プロキシとしても使える
roof.position.y = 1.22;
addToScene(scene, roof);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/4
>
> `createCylinder` はオプションオブジェクト（`{ diameter, height, tessellation }`）をそのまま渡せます（`tessellation` は最小 3 にクランプ）。
> `createBox` だけは第2引数が数値（→ [2-03 の注記](#2-03-メッシュを設置-place-and-scale--)）。
>
> **面ごとに別画像**（ドア面・窓面など、Babylon.js の `faceUV`）は box 生成器では未対応の可能性があります → [2-06](#2-06-マテリアル面ごと--) 参照。テクスチャを貼る例は次の [2-05](#2-05-テクスチャを貼る-add-texture--) を参照してください。

---

## 2-05 テクスチャを貼る (Add Texture) — ○

**目的**：家（box＋屋根）にテクスチャを、地面に単色を割り当てる。

追加 import：`loadTexture2D`

```typescript
// テクスチャ2枚を registerScene 前にまとめてロード
// ★ Lite の loadTexture2D は async。registerScene / startEngine 前に await し終える必要がある
//   （起動後の後入れはレンダーループのクラッシュ要因）
const [roofTex, floorTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
  loadTexture2D(engine, "https://www.babylonjs-playground.com/textures/floor.png"),
]);

// 色マテリアル（Color3 → 配列 [r, g, b]）
const groundMat = createStandardMaterial();
groundMat.diffuseColor = [0, 1, 0];
ground.material = groundMat;

// テクスチャマテリアル
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;
roof.material = roofMat;

const houseMat = createStandardMaterial();
houseMat.diffuseTexture = floorTex;
house.material = houseMat;
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/5
>
> Lite は「UV Scaling / Offset」「Bump / Normal」「Emissive」など主要テクスチャに対応しています
> （フィールド名は IntelliSense で確認：要確認）。

---

## 2-06 マテリアル（面ごと/faceUV） — △

**目的**：1 枚のテクスチャアトラス（横に並んだ正面/右/背面/左の 4 コマ）を、box の各側面に別々に貼る。
Babylon.js の `MeshBuilder.CreateBox({ faceUV, wrap: true })` に相当する機能は **`createBox` には無い**ことが確認できました
（`createBox(engine, size)` は数値の一辺長のみ。`faceUV` / `wrap` / `width` オプションは存在しません）。

### 技法：`createMeshFromData` で本家 box ビルダーを逐語移植する

Lite は公開 API に **`createMeshFromData(engine, name, positions, normals, indices, uvs)`** という、
頂点データを直接渡してメッシュを組み立てる低レベル関数を持っています。これを使い、Babylon.js 本家の
`boxBuilder`（`wrap: true` パス）の頂点配置・UV 計算式・インデックスをそのまま移植した `createWrappedBox` ヘルパーを自作します。

- **面番号**は本家と同じ：`0=+Z`（背面）`1=-Z`（正面）`2=+X`（右）`3=-X`（左）`4=+Y`（上）`5=-Y`（底）
- **`faceUV`** は本家 `Vector4(x, y, z, w)` の代わりに `[uMin, vMin, uMax, vMax]` の配列で表現
- 面ごとの UV は本家と同じ式（頂点順に `(z,w) (x,w) (x,y) (z,y)`）で 4 頂点に適用
- `width` はプリミティブオプションではなく**頂点座標に直接焼き込む**（非一様 scaling に頼らない）

これにより、Lite 組み込み box と同一の UV・巻き順規約を保ったまま、本家と見た目が一致する faceUV box を再現できます。

追加 import：`createCylinder, createGround, createStandardMaterial, createMeshFromData, loadTexture2D`

### A. 一軒家 (Detached House) — `cubehouse.png`

`cubehouse.png` は横 1 列に「正面/右/背面/左」の 4 コマが並んだアトラス。各面の U 範囲を `faceUV` で割り当てます。

```typescript
type FaceUV = readonly [number, number, number, number];   // [uMin, vMin, uMax, vMax]

function createWrappedBox(
  engine: EngineContext,
  options: { width?: number; height?: number; depth?: number; faceUV?: (FaceUV | undefined)[] } = {},
) {
  const w = (options.width ?? 1) / 2, h = (options.height ?? 1) / 2, d = (options.depth ?? 1) / 2;

  // prettier-ignore
  const base = [
    -1,  1,  1,   1,  1,  1,   1, -1,  1,  -1, -1,  1,   // 0: +Z（背面）
     1,  1, -1,  -1,  1, -1,  -1, -1, -1,   1, -1, -1,   // 1: -Z（正面）
     1,  1,  1,   1,  1, -1,   1, -1, -1,   1, -1,  1,   // 2: +X（右側面）
    -1,  1, -1,  -1,  1,  1,  -1, -1,  1,  -1, -1, -1,   // 3: -X（左側面）
     1,  1,  1,  -1,  1,  1,  -1,  1, -1,   1,  1, -1,   // 4: +Y（上面）
     1, -1, -1,  -1, -1, -1,  -1, -1,  1,   1, -1,  1,   // 5: -Y（底面）
  ];
  const positions = new Float32Array(72);
  for (let i = 0; i < 24; i++) {
    positions[i * 3 + 0] = base[i * 3 + 0]! * w;
    positions[i * 3 + 1] = base[i * 3 + 1]! * h;
    positions[i * 3 + 2] = base[i * 3 + 2]! * d;
  }

  const AXES = [[0,0,1],[0,0,-1],[1,0,0],[-1,0,0],[0,1,0],[0,-1,0]];
  const normals = new Float32Array(72);
  for (let f = 0; f < 6; f++) for (let v = 0; v < 4; v++) normals.set(AXES[f]!, (f * 4 + v) * 3);

  // faceUV 適用（本家と同じ式: 頂点順に (z,w) (x,w) (x,y) (z,y)）
  const uvs = new Float32Array(48);
  for (let f = 0; f < 6; f++) {
    const [x, y, z, wv] = options.faceUV?.[f] ?? [0, 0, 1, 1];
    uvs.set([z, wv, x, wv, x, y, z, y], f * 8);
  }

  // prettier-ignore
  const indices = new Uint32Array([
     2,  3,  0,   2,  0,  1,   // +Z
     4,  5,  6,   4,  6,  7,   // -Z
     9, 10, 11,   9, 11,  8,   // +X
    12, 14, 15,  12, 13, 14,   // -X
    17, 19, 16,  17, 18, 19,   // +Y
    20, 22, 23,  20, 21, 22,   // -Y
  ]);

  return createMeshFromData(engine, "box", positions, normals, indices, uvs);
}

const [houseTex, roofTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/cubehouse.png"),
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
]);
const boxMat = createStandardMaterial();
boxMat.diffuseTexture = houseTex;
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;

// アトラスの4分割を各側面へ（上面4・底面5は見えないので未指定）
const faceUV: (FaceUV | undefined)[] = [];
faceUV[0] = [0.5, 0.0, 0.75, 1.0]; // 背面
faceUV[1] = [0.0, 0.0, 0.25, 1.0]; // 正面
faceUV[2] = [0.25, 0.0, 0.5, 1.0]; // 右側面
faceUV[3] = [0.75, 0.0, 1.0, 1.0]; // 左側面

const box = createWrappedBox(engine, { faceUV });
box.material = boxMat;
box.position.y = 0.5;
addToScene(scene, box);

const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
roof.material = roofMat;
roof.scaling.x = 0.75;
roof.rotation.z = Math.PI / 2;
roof.position.y = 1.22;
addToScene(scene, roof);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/6

### B. Semi Detached House（2 戸 1 棟）— `semihouse.png`

`semihouse.png` は「正面(0〜0.4)・窓なし壁(0.4〜0.6)・背面(0.6〜1.0)」の 3 区画。側面 2 面（右/左）は同じ「窓なし壁」区画を共有します。
家体は `width: 2`（幅を頂点に直接焼き込み）、屋根は本家同様 `roof.scaling.y = 2` で**長さ**を 2 倍にします。

```typescript
const [houseTex, roofTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/semihouse.png"),
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
]);
const boxMat = createStandardMaterial();
boxMat.diffuseTexture = houseTex;
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;

const faceUV: (FaceUV | undefined)[] = [];
faceUV[0] = [0.6, 0.0, 1.0, 1.0]; // 背面
faceUV[1] = [0.0, 0.0, 0.4, 1.0]; // 正面
faceUV[2] = [0.4, 0.0, 0.6, 1.0]; // 右側面（窓なし壁を共有）
faceUV[3] = [0.4, 0.0, 0.6, 1.0]; // 左側面（窓なし壁を共有）

const box = createWrappedBox(engine, { width: 2, faceUV });
box.material = boxMat;
box.position.y = 0.5;
addToScene(scene, box);

const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
roof.material = roofMat;
roof.scaling.x = 0.75;
roof.scaling.y = 2;   // TRS はスケール→回転の順で合成されるため、rotation.z 適用後も「長さ」が2倍になる
roof.rotation.z = Math.PI / 2;
roof.position.y = 1.22;
addToScene(scene, roof);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/7

> **まとめ**：box の `faceUV`/`wrap`/非一様 `width` は `createBox` 単体では非対応ですが、公開 API の
> `createMeshFromData` に本家 boxBuilder のロジックを逐語移植すれば、見た目を完全に一致させて再現できます。
> より単純に済ませたい場合は、代わりに以下も選択肢です：
> 1. **面ごとに別プレーンを貼る**（`createPlane` を複数枚、各面に別マテリアル）
> 2. **マルチマテリアル**（`.babylon` ローダー経由なら submesh + multiMaterial に対応）

---

## 2-07 リファクタリング（関数化） — ○

**目的**：[2-06 A の一軒家](#2-06-マテリアル面ごとfaceuv--) を、本家チュートリアルと同じく
`buildGround` / `buildBox` / `buildRoof` の「部品工場」関数へ分割する。`createScene` が「配置図」に徹する構成も本家と同じ。
純粋な TypeScript の関数化なのでそのまま可能ですが、Lite ならではの差分がいくつかあります。

**Lite 移植時の差分**：

- Lite の `create` 系 API は **engine を明示的に要求**するため、`build` 関数はすべて `engine` を引数に取る
  （本家のようなグローバル `scene` への暗黙登録は無い）。
- `loadTexture2D` は **await 必須**なので、テクスチャを使う `buildBox` / `buildRoof` は `async` 関数になる。
  `createScene` 側で `Promise.all` にまとめ、[2-06](#2-06-マテリアル面ごとfaceuv--) と同じ並列ロードを維持する。
- 本家はメッシュ生成時に自動でシーンへ登録されるが、Lite では `addToScene` が必要。
  `build` 関数は「**作って返す**」だけに徹し、シーンへの登録は `createScene` が一括で行う。
- `createBox` に `faceUV` / `wrap` が無い問題は 2-06 と同じく `createWrappedBox`（本家 `boxBuilder` の
  `wrap: true` パスの逐語移植）で対応。ヘルパー本体は [2-06 A](#2-06-マテリアル面ごとfaceuv--) を参照。

追加 import：`createCylinder, createGround, createStandardMaterial, createMeshFromData, loadTexture2D`

```typescript
// createWrappedBox は 2-06 A のものをそのまま流用（ここでは省略）

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement) {
  const scene = createSceneContext(engine);

  // カメラとライト（配置図側でセット）
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const ground = buildGround(engine);
  // buildBox / buildRoof はテクスチャロードを含む async 関数。Promise.all で並列ロードを維持
  const [box, roof] = await Promise.all([buildBox(engine), buildRoof(engine)]);

  // Lite はメッシュの自動登録が無いため、明示的にシーンへ追加
  addToScene(scene, ground);
  addToScene(scene, box);
  addToScene(scene, roof);

  return scene;
}

// 地面組み立て
function buildGround(engine: EngineContext) {
  const groundMat = createStandardMaterial();
  groundMat.diffuseColor = [0, 1, 0];
  const ground = createGround(engine, { width: 10, height: 10 });
  ground.material = groundMat;
  return ground;   // シーンへの登録は createScene が行う
}

// 家体組み立て（テクスチャロードを含むので async）
async function buildBox(engine: EngineContext) {
  const boxMat = createStandardMaterial();
  boxMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/cubehouse.png");

  // アトラスの4分割を各側面へ（上面4・底面5は見えないので未指定）
  const faceUV: (FaceUV | undefined)[] = [];
  faceUV[0] = [0.5, 0.0, 0.75, 1.0]; // 背面
  faceUV[1] = [0.0, 0.0, 0.25, 1.0]; // 正面
  faceUV[2] = [0.25, 0.0, 0.5, 1.0]; // 右側面
  faceUV[3] = [0.75, 0.0, 1.0, 1.0]; // 左側面

  const box = createWrappedBox(engine, { faceUV });   // 本家 CreateBox({ faceUV, wrap: true }) 相当
  box.material = boxMat;
  box.position.y = 0.5;
  return box;
}

// 屋根組み立て（テクスチャロードを含むので async）
async function buildRoof(engine: EngineContext) {
  const roofMat = createStandardMaterial();
  roofMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg");

  const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
  roof.material = roofMat;
  roof.scaling.x = 0.75;
  roof.rotation.z = Math.PI / 2;
  roof.position.y = 1.22;
  return roof;
}
```

> 見た目は [2-06 A の一軒家](#2-06-マテリアル面ごとfaceuv--)（`X79RM0/v/6`）と同一。本節はロジックを変えずに
> 関数へ分割しただけです。本家との唯一の構造的な違いは「**生成と登録の分離**」（`build` 関数は返すだけ、
> `addToScene` は `createScene` が一括）で、Lite にデフォルトマテリアル・自動シーン登録が無いことに由来します。

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

const house = createBox(engine, 1);   // createBox の第2引数は数値（一辺の長さ）
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
