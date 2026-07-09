# 第3部：アニメーション

> 共通テンプレート・凡例は [README](./README.md) を参照。
> このパートの終着点が [ゴール完成版](./99-goal-final.md) です。

---

## 3-00 村のアニメーション — 導入

**—** 親子関係 → 車の組み立て → 車輪/車の動き → キャラの歩行、と進みます。
Lite のアニメは 2 系統：**glTF 由来のスケルタル/クリップ**（`AnimationGroup`）と、**プロパティアニメ**（`createPropertyAnimationClip`）や `onBeforeRender` での自前更新です。

---

## 3-01 メッシュの親子関係 (Mesh Parents) — ○

### 基本：親子付けの2種類

Lite の親子付けには**2 系統**あり、本章の要になります。

- **`node.parent = parent`（素の親子付け）** — 本家の `mesh.parent =` 相当。**ワールド変換を保持しない**ので、
  子は親のローカル原点に置かれます（`children` 配列にも手動で push する必要あり → 下記 `attachToParent`）。
- **`setParent(child, parent)`** — 本家 `TransformNode.setParent()` 相当。**ワールド変換を保持**したまま付け替え、
  `children` 配列も維持します。子が原点にいる時点で呼べば両者は等価です。

`addToScene(scene, parent)` は children を再帰的にたどるので、親 1 つ追加すれば子孫もシーンへ入ります。

```typescript
const parent = createBox(engine, 1);   // createBox の第2引数は数値（一辺の長さ）
parent.position.y = 0.5;
parent.material = createStandardMaterial();   // ★マテリアル必須

const child = createSphere(engine, { diameter: 0.5 });
child.position.x = 1.5;        // 親からの相対位置
setParent(child, parent);      // 親子付け（親の移動/回転に追従）
child.material = createStandardMaterial();     // ★マテリアル必須

addToScene(scene, parent);     // child も再帰的に追加される

// 親を回すと子も一緒に回る
let a = 0;
onBeforeRender(scene, () => {
  a += 0.02;
  parent.rotationQuaternion.y = Math.sin(a / 2);
  parent.rotationQuaternion.w = Math.cos(a / 2);
});
```

### 応用：親子ボックス＋ローカル/ワールド座標軸ビューア

**目的**：本家チュートリアルと同じく、`faceColors` 付き box の親子（`boxParent` / `boxChild`）に、
**子に追従するローカル座標軸**と、**ラベル付きワールド座標軸**を重ねて親子変換を可視化する。
本家が使う `CreateLines` / `faceColors` / `DynamicTexture` / `billboardMode` はいずれも Lite に無いため、
公開 API で代替します。

**Lite 移植時の注意点**（v1.8.0 の d.ts / lib ソースで確認）：

- **`CreateLines`（線分描画）が無い** — line-list トポロジは公開 API から指定できません。Lite 自身のギズモも
  細円柱でワイヤーを構成しているため、本移植も **区間ごとの細い `createCylinder`＋無照明マテリアル**（`createAxisLine`）で軸線を引きます。
- **`CreateBox` に `faceColors` が無い**、かつ **Standard パスは頂点カラーを参照しない**（参照するのは Node/PBR 系）。
  そこで **6×1 ピクセルのパレットテクスチャ**を `createTexture2DFromPixels` で作り、各面の UV 4 頂点をパレット画素の中心へ固定して面ごとに色分けします（既定サンプラが nearest なのでパレット用途にそのまま合う）。
- **`DynamicTexture`（`drawText`）が無い** — canvas 2D に文字を描いて `getImageData` → `createTexture2DFromPixels({ srgb: true })` で代替。
- **`mesh.billboardMode` が無い** — カメラ正対表示は専用の **`FacingBillboardSystem`**（`BILLBOARDMODE_ALL` 相当）を使います
  （`createGridSpriteAtlas` → `createFacingBillboardSystem` → `addFacingBillboardSystem` → `addBillboardSprite`）。
  文字が上下反転する環境では `BillboardSpriteInit.flipY = true` で調整。
- API 名の対応：`camera.minZ` → **`camera.nearPlane`**、`wheelPrecision` は同名、`scene.clearColor` は **`GPUColorDict`（`{ r, g, b, a }`）**。
  `position` / `rotation` の一括設定は **`ObservableVec3.set(x,y,z)` / `EulerProxy.set(x,y,z)`** が使えます。
- **ローカル軸だけは素の親子付け**（`attachToParent` = `.parent =`）で子ボックスのローカル原点へ接続します。
  `setParent` はワールド変換を保持してしまい、軸がワールド原点に残るためです。

追加 import：`createCylinder, createMeshFromData, createTransformNode, createTexture2DFromPixels, createGridSpriteAtlas, createFacingBillboardSystem, addFacingBillboardSystem, addBillboardSprite, setParent`（＋型 `SceneNode, StandardMaterialProps, FacingBillboardSpriteSystem, BillboardSpriteHandle`）

```typescript
type Color4 = readonly [number, number, number, number];  // 本家 Color4 相当
type Vec3Tuple = readonly [number, number, number];

// 面番号は本家 boxBuilder 準拠: 0=+Z 1=-Z 2=+X 3=-X 4=+Y(top) 5=-Y(bottom)
const FACE_COLORS: Color4[] = [
  [0, 0, 1, 1],     // Blue
  [0, 0.5, 0.5, 1], // Teal
  [1, 0, 0, 1],     // Red
  [0.5, 0, 0.5, 1], // Purple
  [0, 1, 0, 1],     // Green
  [1, 1, 0, 1],     // Yellow
];

// 面ごとに色の違う box（本家 CreateBox({ size, faceColors }) 相当）。
// 頂点配置は 2-06 の createWrappedBox と同じ面順。各面の UV 4頂点をパレット画素 f の中心 ((f+0.5)/6, 0.5) に固定する。
function createFaceColorBox(engine: EngineContext, size = 1): Mesh {
  const s = size / 2;
  // base[]（24頂点）/ indices[] は 2-06 A の createWrappedBox と同一（省略）
  // ...
  const uvs = new Float32Array(48);
  for (let f = 0; f < 6; f++) {
    const u = (f + 0.5) / 6;
    uvs.set([u, 0.5, u, 0.5, u, 0.5, u, 0.5], f * 8);   // 面 f 全頂点をパレット画素 f の中心へ
  }
  return createMeshFromData(engine, "faceColorBox", positions, normals, indices, uvs);
}

// faceColors から 6×1 パレットテクスチャ付きマテリアル
function createFaceColorMaterial(engine: EngineContext, faceColors: Color4[]): StandardMaterialProps {
  const pixels = new Uint8Array(6 * 4);
  for (let f = 0; f < 6; f++) {
    const [r, g, b, a] = faceColors[f] ?? [1, 1, 1, 1];
    pixels.set([r, g, b, a].map((v) => Math.round(v * 255)), f * 4);
  }
  const mat = createStandardMaterial();
  mat.diffuseTexture = createTexture2DFromPixels(engine, pixels, 6, 1, { srgb: true }); // 既定 nearest がパレット向き
  return mat;
}

// 1本の軸（折れ線）を細い円柱の連結で描く（本家 CreateLines 代替）
function createAxisLine(engine: EngineContext, name: string, points: Vec3Tuple[],
                        color: readonly [number, number, number], thickness: number): TransformNode {
  const axis = createTransformNode(name);
  const mat = createStandardMaterial();
  mat.diffuseColor = [color[0], color[1], color[2]];
  mat.emissiveColor = [1, 1, 1];
  mat.disableLighting = true;                 // 本家 LinesMesh 同様に無照明の単色
  for (let i = 0; i + 1 < points.length; i++) {
    const a = points[i]!, b = points[i + 1]!;
    const [dx, dy, dz] = [b[0] - a[0], b[1] - a[1], b[2] - a[2]];
    const len = Math.hypot(dx, dy, dz);
    if (len < 1e-6) continue;
    const seg = createCylinder(engine, { height: len, diameter: thickness, tessellation: 6 });
    seg.material = mat;
    seg.position.set((a[0] + b[0]) / 2, (a[1] + b[1]) / 2, (a[2] + b[2]) / 2);
    orientYAxisTo(seg, [dx / len, dy / len, dz / len]);   // +Y をセグメント方向へ向ける最短弧クォータニオン
    setParent(seg, axis);                     // axis は原点なので setParent と素の親子付けは等価
  }
  return axis;
}

// 素の親子付け（本家 `.parent =` 相当）。ワールド変換を保持しない
function attachToParent(child: SceneNode, parent: SceneNode): void {
  child.parent = parent;
  parent.children.push(child);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  scene.clearColor = { r: 0.04, g: 0.04, b: 0.06, a: 1.0 };   // GPUColorDict

  const camera = createArcRotateCamera(-Math.PI / 2.2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  camera.wheelPrecision = 50;
  camera.nearPlane = 0.1;                     // 本家 minZ 相当
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));

  // faceColors 付き box（親・子でパレットマテリアルを共有）
  const faceColorMat = createFaceColorMaterial(engine, FACE_COLORS);
  const boxParent = createFaceColorBox(engine, 1);
  boxParent.material = faceColorMat;
  const boxChild = createFaceColorBox(engine, 0.5);
  boxChild.material = faceColorMat;

  setParent(boxChild, boxParent);             // 両者とも原点なので本家 setParent と等価

  boxChild.position.set(0, 2, 0);             // 親を基準としたローカル座標
  boxChild.rotation.set(Math.PI / 4, Math.PI / 4, Math.PI / 4);
  boxParent.position.set(2, 0, 0);            // ワールド座標
  boxParent.rotation.set(0, 0, -Math.PI / 4);

  // 子のローカル座標軸 —— ローカル原点に置きたいので「素の親子付け」で接続
  const boxChildAxes = createLocalAxes(engine, 1);
  attachToParent(boxChildAxes, boxChild);

  showWorldAxes(engine, scene, 6);            // ラベル付きワールド座標軸（下記）

  addToScene(scene, boxParent);               // boxChild・ローカル軸も再帰追加される
  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/0
>
> **省略した補助関数**（完全版は Playground 参照）：
> - `orientYAxisTo(node, dir)` — `+Y` 軸を単位ベクトル `dir` へ向ける最短弧クォータニオンを設定（`rotationQuaternion.set`）。Lite に `fromUnitVectors` 相当が無いため自前実装。
> - `arrowPoints(size, "x"|"y"|"z")` — 矢じり付き軸のポイント列。
> - `createLocalAxes(engine, size)` / `showWorldAxes(engine, scene, size)` — 3 本の `createAxisLine` を束ねる。ワールド版は X/Y/Z ラベルも表示。
> - `createAxisLabelSystem(engine, scene)` — canvas 2D で「X/Y/Z」を 3 コマ描画 → `createTexture2DFromPixels` → `createGridSpriteAtlas` → `createFacingBillboardSystem`（常にカメラ正対）。ラベルは `addBillboardSprite(system, { position, sizeWorld, frame })` で 1 スプライトずつ生成。
>
> ポイントは **`setParent`（ワールド変換保持）と `attachToParent`＝`.parent =`（保持しない）の使い分け**です。
> 子ボックスは原点で `setParent` するので本家と等価、ローカル軸は子のローカル原点へ置きたいので素の親子付けにします。

---

## 3-02 車を組み立てる (Building the Car) — ○

**目的**：本家どおり、車体輪郭を厚み付き多角形メッシュにして、4 輪を複製で組み立てる。

Lite には `ExtrudePolygon` が無く、earcut 相当の多角形三角形分割も公開 API にありません。
このサンプルの輪郭は **凸多角形** なので、`createMeshFromData` で
**上下キャップ＋側面クアッド** を直接組み立てます。

追加 import：`createCylinder, createMeshFromData, cloneTransformNode, createStandardMaterial`（＋型 `Mesh, SceneNode`）

```typescript
type OutlinePoint = readonly [number, number];

// 本家 MeshBuilder.ExtrudePolygon({ shape, depth }) 相当（凸多角形限定）
function extrudePolygonXoZ(engine: EngineContext, name: string, outline: OutlinePoint[], depth: number): Mesh {
  const n = outline.length;
  const capTris = n - 2;

  const edges: [number, number][] = [];
  for (let i = 0; i < n; i++) {
    const j = (i + 1) % n;
    const [ax, az] = outline[i]!;
    const [bx, bz] = outline[j]!;
    if (Math.hypot(bx - ax, bz - az) > 1e-9) edges.push([i, j]);
  }

  const vertCount = n * 2 + edges.length * 4;
  const positions = new Float32Array(vertCount * 3);
  const normals = new Float32Array(vertCount * 3);
  const uvs = new Float32Array(vertCount * 2);
  const indices = new Uint32Array(capTris * 2 * 3 + edges.length * 6);

  for (let i = 0; i < n; i++) {
    const [x, z] = outline[i]!;
    positions.set([x, 0, z], i * 3);
    normals.set([0, 1, 0], i * 3);
    uvs.set([x + 0.5, z + 0.5], i * 2);

    positions.set([x, -depth, z], (n + i) * 3);
    normals.set([0, -1, 0], (n + i) * 3);
    uvs.set([x + 0.5, z + 0.5], (n + i) * 2);
  }

  let idx = 0;
  for (let i = 1; i < n - 1; i++) {
    indices.set([0, i, i + 1], idx);
    indices.set([n, n + i + 1, n + i], idx + 3);
    idx += 6;
  }

  let v = n * 2;
  for (const [i, j] of edges) {
    const [ax, az] = outline[i]!;
    const [bx, bz] = outline[j]!;
    const dx = bx - ax;
    const dz = bz - az;
    const len = Math.hypot(dx, dz);
    const nx = dz / len;
    const nz = -dx / len;

    positions.set([
      ax, -depth, az,
      bx, -depth, bz,
      bx, 0, bz,
      ax, 0, az,
    ], v * 3);
    for (let k = 0; k < 4; k++) normals.set([nx, 0, nz], (v + k) * 3);
    uvs.set([0, 0, len, 0, len, 1, 0, 1], v * 2);
    indices.set([v, v + 1, v + 2, v, v + 2, v + 3], idx);
    v += 4;
    idx += 6;
  }

  return createMeshFromData(engine, name, positions, normals, indices, uvs);
}

function attachToParent(child: SceneNode, parent: SceneNode): void {
  child.parent = parent;
  parent.children.push(child);
}

function buildCar(engine: EngineContext): Mesh {
  const outline: OutlinePoint[] = [
    [-0.3, -0.1],
    [0.2, -0.1],
  ];

  for (let i = 0; i < 20; i++) {
    outline.push([0.2 * Math.cos((i * Math.PI) / 40), 0.2 * Math.sin((i * Math.PI) / 40) - 0.1]);
  }

  outline.push([0, 0.1]);
  outline.push([-0.3, 0.1]);

  const car = extrudePolygonXoZ(engine, "car", outline, 0.2);
  car.material = createStandardMaterial();
  return car;
}

const car = buildCar(engine);

const wheelRB = createCylinder(engine, { diameter: 0.125, height: 0.05 });
wheelRB.material = createStandardMaterial();
attachToParent(wheelRB, car);
wheelRB.position.x = -0.2;
wheelRB.position.y = 0.035;
wheelRB.position.z = -0.1;

const wheelRF = cloneTransformNode(wheelRB) as Mesh;
attachToParent(wheelRF, car);
wheelRF.position.x = 0.1;

const wheelLB = cloneTransformNode(wheelRB) as Mesh;
attachToParent(wheelLB, car);
wheelLB.position.y = -0.2 - 0.035;

const wheelLF = cloneTransformNode(wheelRF) as Mesh;
attachToParent(wheelLF, car);
wheelLF.position.y = -0.2 - 0.035;

const wheels = [wheelRB, wheelRF, wheelLB, wheelLF];

addToScene(scene, car);   // 4 輪も再帰的に追加される
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/1
>
> ポイントは 3 つです。
> 1. **`ExtrudePolygon` の代替**として、凸輪郭なら `createMeshFromData` で直接組み立てられる
> 2. **`cloneTransformNode`** は transform / material を複製し、メッシュは GPU バッファ共有の shallow clone になる
> 3. 本家の `wheel.parent = car` は **素の親子付け**なので、Lite では `setParent` ではなく `attachToParent` を使う

---

## 3-03 車のマテリアル (Car Material) — ○

**目的**：前章の押し出し車体に `car.png` を貼り、続けて車輪に `wheel.png` を貼る。車体は `ExtrudePolygon` 相当の `faceUV` / `wrap`、車輪は `CreateCylinder` 相当の `faceUV` を Lite 用に移植します。

### 車体に画像を張っていく

上面・側面・下面を `faceUV` で画像の各領域へ割り当て、`wrap: true` で側面テクスチャを車体一周に巻き付けます。

本家サンプルの要点はこの 3 行です。

```typescript
faceUV[0] = new Vector4(0, 0.5, 0.38, 1);  // 上面（天井）
faceUV[1] = new Vector4(0, 0, 1, 0.5);     // 側面（車体側面の帯）
faceUV[2] = new Vector4(0.38, 1, 0, 0.5);  // 下面（u 反転指定）
```

Lite には `ExtrudePolygon` が無いので、前章の `extrudePolygonXoZ` を `faceUV` / `wrap` 対応に拡張します。

追加 import：`createMeshFromData, createStandardMaterial, loadTexture2D`（＋型 `EngineContext, Mesh`）

```typescript
type OutlinePoint = readonly [number, number];
type FaceUV = readonly [number, number, number, number];

const DEFAULT_FACE_UV: FaceUV = [0, 0, 1, 1];

interface ExtrudePolygonXoZOptions {
  shape: OutlinePoint[];
  depth: number;
  faceUV?: FaceUV[];   // [0]=上面, [1]=側面, [2]=下面
  wrap?: boolean;      // true で側面テクスチャを全周に巻き付ける
}

function extrudePolygonXoZ(engine: EngineContext, name: string, options: ExtrudePolygonXoZOptions): Mesh {
  const { shape: outline, depth, wrap = false } = options;
  const fuvTop = options.faceUV?.[0] ?? DEFAULT_FACE_UV;
  const fuvSide = options.faceUV?.[1] ?? DEFAULT_FACE_UV;
  const fuvBottom = options.faceUV?.[2] ?? DEFAULT_FACE_UV;

  const n = outline.length;
  const capTris = n - 2;

  let xmin = Infinity;
  let xmax = -Infinity;
  let zmin = Infinity;
  let zmax = -Infinity;
  for (const [x, z] of outline) {
    xmin = Math.min(xmin, x);
    xmax = Math.max(xmax, x);
    zmin = Math.min(zmin, z);
    zmax = Math.max(zmax, z);
  }
  const width = xmax - xmin;
  const height = zmax - zmin;

  const edgeLen: number[] = [];
  const cumulate: number[] = [0];
  let totalLen = 0;
  for (let i = 0; i < n; i++) {
    const [ax, az] = outline[i]!;
    const [bx, bz] = outline[(i + 1) % n]!;
    const len = Math.hypot(bx - ax, bz - az);
    edgeLen.push(len);
    totalLen += len;
    cumulate.push(totalLen);
  }

  const edges: number[] = [];
  for (let i = 0; i < n; i++) {
    if (edgeLen[i]! > 1e-9) edges.push(i);
  }

  const vertCount = n * 2 + edges.length * 4;
  const positions = new Float32Array(vertCount * 3);
  const normals = new Float32Array(vertCount * 3);
  const uvs = new Float32Array(vertCount * 2);
  const indices = new Uint32Array(capTris * 2 * 3 + edges.length * 6);

  for (let i = 0; i < n; i++) {
    const [x, z] = outline[i]!;
    const u = (x - xmin) / width;
    const v = (z - zmin) / height;

    positions.set([x, 0, z], i * 3);
    normals.set([0, 1, 0], i * 3);
    uvs.set([fuvTop[0] + (fuvTop[2] - fuvTop[0]) * u, fuvTop[1] + (fuvTop[3] - fuvTop[1]) * v], i * 2);

    positions.set([x, -depth, z], (n + i) * 3);
    normals.set([0, -1, 0], (n + i) * 3);
    uvs.set(
      [fuvBottom[0] + (fuvBottom[2] - fuvBottom[0]) * (1 - u), fuvBottom[1] + (fuvBottom[3] - fuvBottom[1]) * (1 - v)],
      (n + i) * 2,
    );
  }

  let idx = 0;
  for (let i = 1; i < n - 1; i++) {
    indices.set([0, i, i + 1], idx);
    indices.set([n, n + i + 1, n + i], idx + 3);
    idx += 6;
  }

  let v0 = n * 2;
  for (const i of edges) {
    const j = (i + 1) % n;
    const [ax, az] = outline[i]!;
    const [bx, bz] = outline[j]!;
    const dx = bx - ax;
    const dz = bz - az;
    const len = edgeLen[i]!;
    const nx = dz / len;
    const nz = -dx / len;

    const uA = wrap ? fuvSide[0] + ((fuvSide[2] - fuvSide[0]) * cumulate[i]!) / totalLen : fuvSide[0];
    const uB = wrap ? fuvSide[0] + ((fuvSide[2] - fuvSide[0]) * cumulate[i + 1]!) / totalLen : fuvSide[2];
    const vTop = fuvSide[3];
    const vBottom = fuvSide[1];

    positions.set([
      ax, -depth, az,
      bx, -depth, bz,
      bx, 0, bz,
      ax, 0, az,
    ], v0 * 3);
    for (let k = 0; k < 4; k++) normals.set([nx, 0, nz], (v0 + k) * 3);
    uvs.set([uA, vBottom, uB, vBottom, uB, vTop, uA, vTop], v0 * 2);
    indices.set([v0, v0 + 1, v0 + 2, v0, v0 + 2, v0 + 3], idx);
    v0 += 4;
    idx += 6;
  }

  return createMeshFromData(engine, name, positions, normals, indices, uvs);
}

const outline: OutlinePoint[] = [
  [-0.3, -0.1],
  [0.2, -0.1],
];

for (let i = 0; i < 20; i++) {
  outline.push([0.2 * Math.cos((i * Math.PI) / 40), 0.2 * Math.sin((i * Math.PI) / 40) - 0.1]);
}

outline.push([0, 0.1]);
outline.push([-0.3, 0.1]);

const faceUV: FaceUV[] = [
  [0, 0.5, 0.38, 1],  // 上面
  [0, 0, 1, 0.5],     // 側面
  [0.38, 1, 0, 0.5],  // 下面
];

const carMat = createStandardMaterial();
carMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/car.png");

const car = extrudePolygonXoZ(engine, "car", { shape: outline, depth: 0.2, faceUV, wrap: true });
car.material = carMat;
addToScene(scene, car);
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/2
>
> `wrap: true` の側面 UV は、各辺ごとに同じ画像を繰り返すのではなく、累積周長を全周長で正規化して `faceUV[1]` の範囲へ割り当てます。輪郭中の重複点は長さ 0 の辺として累積長に影響しないため、ジオメトリ生成ではスキップして構いません。

### 車輪に画像を張っていく

本家 `CreateCylinder` の `faceUV` は、`[0]=下面キャップ, [1]=円柱側面, [2]=上面キャップ` です。`wheel.png` は上下キャップへ全体を貼り、側面は同じ UV 点を参照させて外周を単色にします。

```typescript
const wheelFaceUV: FaceUV[] = [
  [0, 0, 1, 1],     // 下面キャップ：wheel.png 全体
  [0, 0.5, 0, 0.5], // 円柱側面：同一点を参照し、外周を単色化
  [0, 0, 1, 1],     // 上面キャップ：wheel.png 全体
];
```

Lite で `CreateCylinder({ faceUV })` 相当を厳密に扱うため、円柱も `createMeshFromData` で生成します。

```typescript
interface CylinderYOptions {
  diameter: number;
  height: number;
  tessellation?: number;
  faceUV?: FaceUV[]; // [0]=下面, [1]=側面, [2]=上面
}

function remapFaceUV(faceUV: FaceUV, u: number, v: number): readonly [number, number] {
  return [
    faceUV[0] + (faceUV[2] - faceUV[0]) * u,
    faceUV[1] + (faceUV[3] - faceUV[1]) * v,
  ];
}

function createCylinderY(engine: EngineContext, name: string, options: CylinderYOptions): Mesh {
  const { diameter, height, tessellation = 32 } = options;
  const fuvBottom = options.faceUV?.[0] ?? DEFAULT_FACE_UV;
  const fuvSide = options.faceUV?.[1] ?? DEFAULT_FACE_UV;
  const fuvTop = options.faceUV?.[2] ?? DEFAULT_FACE_UV;

  const radius = diameter / 2;
  const halfHeight = height / 2;
  const sideVertCount = (tessellation + 1) * 2;
  const capVertCount = tessellation + 2;
  const vertCount = sideVertCount + capVertCount * 2;

  const positions = new Float32Array(vertCount * 3);
  const normals = new Float32Array(vertCount * 3);
  const uvs = new Float32Array(vertCount * 2);
  const indices = new Uint32Array(tessellation * 4 * 3);

  for (let i = 0; i <= tessellation; i++) {
    const ratio = i / tessellation;
    const angle = ratio * Math.PI * 2;
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    const x = cos * radius;
    const z = sin * radius;
    const bottomIndex = i * 2;
    const topIndex = bottomIndex + 1;

    positions.set([x, -halfHeight, z], bottomIndex * 3);
    positions.set([x, halfHeight, z], topIndex * 3);
    normals.set([cos, 0, sin], bottomIndex * 3);
    normals.set([cos, 0, sin], topIndex * 3);
    uvs.set(remapFaceUV(fuvSide, ratio, 0), bottomIndex * 2);
    uvs.set(remapFaceUV(fuvSide, ratio, 1), topIndex * 2);
  }

  let idx = 0;
  for (let i = 0; i < tessellation; i++) {
    const bottom0 = i * 2;
    const top0 = bottom0 + 1;
    const bottom1 = (i + 1) * 2;
    const top1 = bottom1 + 1;
    indices.set([bottom0, bottom1, top1, bottom0, top1, top0], idx);
    idx += 6;
  }

  const bottomStart = sideVertCount;
  const bottomCenter = bottomStart;
  positions.set([0, -halfHeight, 0], bottomCenter * 3);
  normals.set([0, -1, 0], bottomCenter * 3);
  uvs.set(remapFaceUV(fuvBottom, 0.5, 0.5), bottomCenter * 2);

  for (let i = 0; i <= tessellation; i++) {
    const ratio = i / tessellation;
    const angle = ratio * Math.PI * 2;
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    const vertexIndex = bottomStart + 1 + i;
    positions.set([cos * radius, -halfHeight, sin * radius], vertexIndex * 3);
    normals.set([0, -1, 0], vertexIndex * 3);
    uvs.set(remapFaceUV(fuvBottom, 0.5 - cos * 0.5, 0.5 + sin * 0.5), vertexIndex * 2);
  }

  for (let i = 0; i < tessellation; i++) {
    const current = bottomStart + 1 + i;
    const next = current + 1;
    indices.set([bottomCenter, next, current], idx);
    idx += 3;
  }

  const topStart = bottomStart + capVertCount;
  const topCenter = topStart;
  positions.set([0, halfHeight, 0], topCenter * 3);
  normals.set([0, 1, 0], topCenter * 3);
  uvs.set(remapFaceUV(fuvTop, 0.5, 0.5), topCenter * 2);

  for (let i = 0; i <= tessellation; i++) {
    const ratio = i / tessellation;
    const angle = ratio * Math.PI * 2;
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    const vertexIndex = topStart + 1 + i;
    positions.set([cos * radius, halfHeight, sin * radius], vertexIndex * 3);
    normals.set([0, 1, 0], vertexIndex * 3);
    uvs.set(remapFaceUV(fuvTop, 0.5 + cos * 0.5, 0.5 + sin * 0.5), vertexIndex * 2);
  }

  for (let i = 0; i < tessellation; i++) {
    const current = topStart + 1 + i;
    const next = current + 1;
    indices.set([topCenter, current, next], idx);
    idx += 3;
  }

  return createMeshFromData(engine, name, positions, normals, indices, uvs);
}

const wheelMat = createStandardMaterial();
wheelMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/wheel.png");

const createWheel = (name: string, x: number, y: number, z: number): Mesh => {
  const wheel = createCylinderY(engine, name, {
    diameter: 0.125,
    height: 0.05,
    tessellation: 32,
    faceUV: wheelFaceUV,
  });

  wheel.material = wheelMat;
  wheel.parent = car;
  wheel.position.x = x;
  wheel.position.y = y;
  wheel.position.z = z;
  addToScene(scene, wheel);

  return wheel;
};

const wheelRB = createWheel("wheelRB", -0.2, 0.035, -0.1);
const wheelRF = createWheel("wheelRF", 0.1, 0.035, -0.1);
const wheelLB = createWheel("wheelLB", -0.2, -0.2 - 0.035, -0.1);
const wheelLF = createWheel("wheelLF", 0.1, -0.2 - 0.035, -0.1);
const wheels = [wheelRB, wheelRF, wheelLB, wheelLF];

car.rotation.x = -Math.PI / 2; // 押し出し方向を車幅方向へ向ける
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/3
>
> `wheelFaceUV[1] = [0, 0.5, 0, 0.5]` は u/v の範囲を持たないため、側面の全頂点が `wheel.png` 上の同一点を参照します。これによりタイヤ外周は画像を引き伸ばさず、黒い帯として見せられます。

---

## 3-04 車輪のアニメーション (Wheel Animation) — ○

**目的**：まず 1 つの車輪だけを作り、画像付きタイヤが回転することを確認する。本家の `Animation("rotation.y", 30fps)` と `scene.beginAnimation(..., true)` を、Lite では `requestAnimationFrame` と `performance.now()` で置き換えます。

本家サンプルの要点は、`frame 0 = rotation.y: 0`、`frame 30 = rotation.y: 2π`、`fps = 30` なので **1 秒で 1 回転**することです。

前章の `FaceUV` / `remapFaceUV` / `createCylinderY` をそのまま使い、まず `wheelRB` 1 本だけを生成します。

```typescript
function animateWheelRotationY(wheel: Mesh, durationSeconds = 1): void {
  const startTime = performance.now();

  const update = (currentTime: number): void => {
    const elapsedSeconds = (currentTime - startTime) / 1000;
    const progress = (elapsedSeconds / durationSeconds) % 1;

    // frame 0 の 0 から frame 30 の 2π までを線形補間
    wheel.rotation.y = progress * 2 * Math.PI;

    requestAnimationFrame(update);
  };

  requestAnimationFrame(update);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  scene.clearColor = [1, 1, 1, 1]; // 元サンプルの白背景に合わせる

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 1.5, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));

  const wheelUV: FaceUV[] = [
    [0, 0, 1, 1],     // 下面キャップ：wheel.png 全体
    [0, 0.5, 0, 0.5], // 円柱側面：同一点を参照し、タイヤ外周を単色化
    [0, 0, 1, 1],     // 上面キャップ：wheel.png 全体
  ];

  const wheelMat = createStandardMaterial();
  wheelMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/wheel.png");

  const wheelRB = createCylinderY(engine, "wheelRB", {
    diameter: 0.125,
    height: 0.05,
    tessellation: 32,
    faceUV: wheelUV,
  });

  wheelRB.material = wheelMat;
  addToScene(scene, wheelRB);

  // frame 0 = 0、frame 30 = 2π、30fps と同じく 1 秒で 1 回転
  animateWheelRotationY(wheelRB, 1);

  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/4
>
> `requestAnimationFrame` の時刻から角度を計算するので、30fps / 60fps / 120fps のどれでも 1 秒あたり 1 回転になります。本家の `rotation.y` アニメーションと同じく、この段階では車体へ取り付けず、単体の車輪で動きを確認します。

### 他の車輪も回す

次は車体と 4 輪を生成し、`wheelRB` / `wheelRF` / `wheelLB` / `wheelLF` を同じ回転角で同期させます。本家は `scene.beginAnimation(...)` を 4 回呼びますが、Lite では 1 つの `requestAnimationFrame` 更新で `wheels` 配列をまとめて回します。

```typescript
interface CarParts {
  car: Mesh;
  wheels: Mesh[];
}

async function buildCar(engine: EngineContext, scene: SceneContext): Promise<CarParts> {
  // 3-03 と同じ car.png 付き車体を生成
  const car = extrudePolygonXoZ(engine, "car", {
    shape: outline,
    depth: 0.2,
    faceUV: carFaceUV,
    wrap: true,
  });
  car.material = carMat;
  addToScene(scene, car);

  const wheels: Mesh[] = [];

  const createWheel = (name: string, x: number, y: number, z: number): Mesh => {
    const wheel = createCylinderY(engine, name, {
      diameter: 0.125,
      height: 0.05,
      tessellation: 32,
      faceUV: wheelFaceUV,
    });

    wheel.material = wheelMat;
    wheel.parent = car;
    wheel.position.x = x;
    wheel.position.y = y;
    wheel.position.z = z;
    addToScene(scene, wheel);
    wheels.push(wheel);

    return wheel;
  };

  createWheel("wheelRB", -0.2, 0.035, -0.1);
  createWheel("wheelRF", 0.1, 0.035, -0.1);
  createWheel("wheelLB", -0.2, -0.2 - 0.035, -0.1);
  createWheel("wheelLF", 0.1, -0.2 - 0.035, -0.1);

  car.rotation.x = -Math.PI / 2;

  return { car, wheels };
}

function animateWheelsRotationY(wheels: readonly Mesh[], durationSeconds = 1): void {
  const startTime = performance.now();

  const update = (currentTime: number): void => {
    const elapsedSeconds = (currentTime - startTime) / 1000;
    const progress = (elapsedSeconds / durationSeconds) % 1;
    const rotationY = progress * 2 * Math.PI;

    for (const wheel of wheels) {
      wheel.rotation.y = rotationY;
    }

    requestAnimationFrame(update);
  };

  requestAnimationFrame(update);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 2, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const { wheels } = await buildCar(engine, scene);
  animateWheelsRotationY(wheels, 1);

  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/5
>
> 4 輪に別々の `requestAnimationFrame` を設定せず、1 つの更新処理で同じ `rotation.y` を設定します。これで本家の `beginAnimation(wheelRB/RF/LB/LF, 0, 30, true)` と同じく、全タイヤが 1 秒 1 回転で同期します。

---

## 3-05 車のアニメーション (Car Animation) — ○

**目的**：車体を `X=-4` から `X=4` へ前進させながら、4 輪も同時に回す。本家の `Animation("position.x", 30fps)` / `beginAnimation` を、Lite の **Property Animation API** で置き換えます。

このステップでは、本家サンプルに近づけるため `car.babylon` を読み込みます。ただし現在の Lite ローダー向けに、読み込み前に `geometryId` 参照を各 mesh の頂点データへ展開し、テクスチャとアニメーションはコード側で設定します。

追加 import：`createAnimationManager, createPropertyAnimationClip, createPropertyAnimationGroup, getContainerMeshes, loadBabylon, onBeforeRender, updateAnimationManager`（＋型 `AnimationManager`）

```typescript
interface CarParts {
  car: Mesh;
  wheels: readonly Mesh[];
}

function findMeshByName(meshes: readonly Mesh[], name: string): Mesh {
  const mesh = meshes.find((candidate) => candidate.name === name);
  if (!mesh) throw new Error(`メッシュ "${name}" が見つかりません。`);
  return mesh;
}

async function loadCar(engine: EngineContext, scene: SceneContext): Promise<CarParts> {
  const sourceUrl = "https://assets.babylonjs.com/meshes/car.babylon";
  const convertedUrl = await createConvertedBabylonUrl(sourceUrl); // geometryId を mesh 頂点データへ展開

  try {
    const container = await loadBabylon(engine, convertedUrl, {
      loadTextures: false,
      loadCamera: false,
    });
    addToScene(scene, container);

    const meshes = getContainerMeshes(container);
    const car = findMeshByName(meshes, "car");
    const wheelRB = findMeshByName(meshes, "wheelRB");
    const wheelRF = findMeshByName(meshes, "wheelRF");
    const wheelLB = findMeshByName(meshes, "wheelLB");
    const wheelLF = findMeshByName(meshes, "wheelLF");

    const carMaterial = createStandardMaterial();
    carMaterial.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/car.png");
    car.material = carMaterial;

    const wheelMaterial = createStandardMaterial();
    wheelMaterial.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/textures/wheelcar.png");
    for (const wheel of [wheelRB, wheelRF, wheelLB, wheelLF]) wheel.material = wheelMaterial;

    return { car, wheels: [wheelRB, wheelRF, wheelLB, wheelLF] };
  } finally {
    URL.revokeObjectURL(convertedUrl);
  }
}

function createCarAnimation(manager: AnimationManager, car: Mesh): void {
  const carClip = createPropertyAnimationClip(
    "carAnimation",
    [{
      path: "position.x",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: -4 },
        { frame: 150, value: 4 },
      ],
    }],
    { frameRate: 30 },
  );

  createPropertyAnimationGroup(manager, car, carClip, {
    fromFrame: 0,
    toFrame: 150,
    loop: true,
  });
}

function createWheelAnimations(manager: AnimationManager, wheels: readonly Mesh[]): void {
  const wheelClip = createPropertyAnimationClip(
    "wheelAnimation",
    [{
      path: "rotation.y",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 0 },
        { frame: 30, value: 2 * Math.PI },
      ],
    }],
    { frameRate: 30 },
  );

  for (const wheel of wheels) {
    createPropertyAnimationGroup(manager, wheel, wheelClip, {
      fromFrame: 0,
      toFrame: 30,
      loop: true,
    });
  }
}

function setupAnimations(engine: EngineContext, scene: SceneContext, car: Mesh, wheels: readonly Mesh[]): void {
  const manager = createAnimationManager({ engine });

  createCarAnimation(manager, car);       // frame 0: x=-4 → frame 150: x=4（5秒）
  createWheelAnimations(manager, wheels); // frame 0: y=0 → frame 30: y=2π（1秒）

  onBeforeRender(scene, (deltaMs: number) => {
    updateAnimationManager(manager, deltaMs);
  });
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 2, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const { car, wheels } = await loadCar(engine, scene);
  setupAnimations(engine, scene, car, wheels);

  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/6
>
> 車体は `frame 0 = position.x: -4`、`frame 150 = position.x: 4`、`frameRate = 30` なので 5 秒で移動します。車輪は `frame 0 = rotation.y: 0`、`frame 30 = rotation.y: 2π` なので 1 秒で 1 回転します。`AnimationManager` は自動では進まないため、`onBeforeRender` で `updateAnimationManager(manager, deltaMs)` を呼びます。
>
> **補助関数**：`createConvertedBabylonUrl(sourceUrl)` は、`car.babylon` 内の `geometryId` が参照する `geometries.vertexData` を各 mesh の `positions` / `normals` / `uvs` / `indices` へ展開し、Blob URL として `loadBabylon` に渡します。完全版は Playground 参照。

### 村の中を走らせる

次は `village.glb` と `car.glb` を読み込み、村の道路上で車を走らせます。

> **訂正**：以前この節では「`car.glb` には車輪回転アニメーションが埋め込まれている」と説明していましたが、これは誤りでした。`car.glb` の glTF `animations` は空で、車輪回転は含まれていません。本家チュートリアルも *"car.glb has no built-in wheel animation, so we create one"* としてコード側で作成しています。そのため、**車輪回転もコード側で作成**し、車体の前進とあわせて `AnimationManager` に登録します。

- **車輪**：`rotation.y` を frame 0→30（1 秒）で `-2π` 回すクリップを 4 輪それぞれのノードへ割り当て、ループ再生します。
- **車体**：`position.z` のプロパティアニメーションで前進させます。

> **補足（`rotationQuaternion = null` は不要）**：本家では毎フレームの `rotation.y` 書き込みで劣化しないよう `wheel.rotationQuaternion = null` にしますが、Lite では不要です。Lite の `node.rotation` は quaternion 上の Euler プロキシで、書き込みは直接 quaternion に反映されます。プロキシは Euler 状態をキャッシュし、外部で quaternion が変更されない限り再抽出しないため、毎フレーム書き込んでもジンバル問題は起きません（`car.glb` の車輪ノードはレスト回転なしのため、初期姿勢も本家の null 化後と一致します）。

追加 import：`loadGltf`（＋型 `AssetContainer, PropertyAnimationClip, SceneNode`）

```typescript
function findNodeByName(container: AssetContainer, name: string): SceneNode {
  const visited = new Set<SceneNode>();

  const visit = (node: SceneNode): SceneNode | null => {
    if (visited.has(node)) return null;
    visited.add(node);
    if (node.name === name) return node;

    for (const child of node.children ?? []) {
      const found = visit(child as SceneNode);
      if (found) return found;
    }
    return null;
  };

  for (const entity of container.entities) {
    if (!entity || typeof entity !== "object" || !("name" in entity) || !("children" in entity)) continue;
    const found = visit(entity as SceneNode);
    if (found) return found;
  }

  throw new Error(`ノード "${name}" が見つかりません。`);
}

function addContainerEntitiesToScene(scene: SceneContext, container: AssetContainer): void {
  for (const entity of container.entities) {
    addToScene(scene, entity);
  }
}

// car.glb には車輪アニメーションが埋め込まれていないため、
// 本家チュートリアルと同様にコード側で作成する（4 輪でクリップを共有）。
function addWheelAnimations(manager: AnimationManager, container: AssetContainer): void {
  const wheelClip: PropertyAnimationClip = createPropertyAnimationClip(
    "wheelAnimation",
    [{
      path: "rotation.y",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 0 },
        { frame: 30, value: -2 * Math.PI },
      ],
    }],
    { frameRate: 30 },
  );

  const wheelNames = ["wheelRB", "wheelRF", "wheelLB", "wheelLF"] as const;
  for (const name of wheelNames) {
    const wheel = findNodeByName(container, name);
    createPropertyAnimationGroup(manager, wheel, wheelClip, {
      fromFrame: 0,
      toFrame: 30,
      loop: true,
      speedRatio: 1,
    });
  }
}

function addCarMovementAnimation(manager: AnimationManager, car: SceneNode): void {
  const carMoveClip = createPropertyAnimationClip(
    "carMovement",
    [{
      path: "position.z",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 8 },
        { frame: 150, value: -7 },
        { frame: 200, value: -7 },
      ],
    }],
    { frameRate: 30 },
  );

  createPropertyAnimationGroup(manager, car, carMoveClip, {
    fromFrame: 0,
    toFrame: 200,
    loop: true,
    speedRatio: 1,
  });
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [villageContainer, carContainer] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/village.glb"),
    loadGltf(engine, "https://assets.babylonjs.com/meshes/car.glb"),
  ]);

  addContainerEntitiesToScene(scene, villageContainer);
  addContainerEntitiesToScene(scene, carContainer);

  const car = findNodeByName(carContainer, "car");
  car.rotation.set(Math.PI / 2, 0, -Math.PI / 2);
  car.position.set(-3, 0.16, 8);

  const animationManager = createAnimationManager({ engine });
  addWheelAnimations(animationManager, carContainer);  // 4 輪の回転（コード側で作成）
  addCarMovementAnimation(animationManager, car);      // 村の中を Z 方向へ走行

  onBeforeRender(scene, (deltaMs: number) => {
    updateAnimationManager(animationManager, deltaMs);
  });

  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/8
>
> `car.glb` の描画メッシュ名は `gltf_mesh_0` などになるため、`getContainerMeshes()` ではなく `AssetContainer.entities` のノード階層から glTF ノード名 `"car"` / `"wheelRB"` 等を探します。車輪は `frame 0 = rotation.y: 0`、`frame 30 = -2π` なので 1 秒で 1 回転します。車体移動は `frame 0 = position.z: 8`、`frame 150 = -7`、`frame 200 = -7` で、5 秒走行して約 1.67 秒停止してからループします。

---

## 3-06 キャラクターのアニメーション (Character Animation) — ○（アセットは glTF）

**目的**：リグ付きキャラを読み込み歩行アニメを再生する。

Babylon.js は `Dude.babylon` を使いますが、Lite の `.babylon` ローダーは **スキン/アニメ非対応**（`animationGroups` は常に空）。
一方で **glTF 由来のスケルタルアニメは完全対応**なので、`Xbot.glb`（walk/run 検証済み）に置換します。

追加 import：`loadGltf, playAnimation, stopAnimation`

```typescript
const xbot = await loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb");
addToScene(scene, xbot);

// 全クリップを止めて walk だけ再生（idle/run が同時再生されるのを防ぐ）
for (const g of xbot.animationGroups ?? []) stopAnimation(g);
const walk = (xbot.animationGroups ?? []).find((g) => g.name === "walk") ?? xbot.animationGroups?.[0];
if (walk) { walk.loopAnimation = true; walk.speedRatio = 1.0; playAnimation(walk); }
```

| 前提 | 判定 |
|---|---|
| `Dude.babylon` をそのまま | ✕（静止表示のみ） |
| glTF（`Xbot.glb` 等）に置換 | ○（歩行/走行/ブレンドまで対応） |

---

## 3-07 村を歩き回る (A Walk Around The Village) — △

**目的**：まず球を三角形の軌道に沿って走らせて「前進＋その場で回頭」の組み合わせを掴み、
そのうえでキャラをルートに沿って歩かせ、カメラで追う。**これがゴールの中核**です。

**Lite 移植時の注意点**：

- **`mesh.movePOV` / `mesh.rotate` が無い** — Lite の `Mesh` には視点基準の移動・ローカル軸回転のヘルパーがありません。
  本章の軌道は XZ 平面・ヨーのみなので、**進行方向のヨー角スカラー `yaw` を自前で保持**し、
  `position += (-sin yaw, 0, -cos yaw) * step` で `movePOV(0, 0, step)` を、`yaw += turn` で `rotate(Axis.Y, turn, LOCAL)` を置き換えます。
  本家 `movePOV` は `definedFacingForward = true` が既定なので **yaw = 0 のときの前方は `-Z`**。Lite の `+Z` 前提から符号を反転して規約を合わせます。
- **`CreateLines`（線分描画）が無い** — 3-01 と同じく、**細い `createCylinder` ＋無照明マテリアル**で線分を代用します。
- **`FollowCamera` が無い** — 後半のキャラ追従は `ArcRotateCamera` を `camera.parent = root` で親子付けして代替します。

### 移動と回転 (Combining Movements)

**例）三角形の辺に沿って移動する球** — 直線で `dist` だけ進んだら `turn` だけ回頭する、を繰り返します。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createSphere, createCylinder, createStandardMaterial, onBeforeRender`（＋型 `EngineContext, Mesh, SceneContext, StandardMaterialProps`）

```typescript
type Vec3Tuple = readonly [number, number, number];

/** CreateLines 相当のフラット白線マテリアル（ライト非依存） */
function createLineMaterial(): StandardMaterialProps {
  const mat = createStandardMaterial();
  mat.diffuseColor = [1, 1, 1];
  mat.emissiveColor = [1, 1, 1];  // 出力 = emissive * diffuse * baseColor = 白
  mat.disableLighting = true;     // 陰影計算をスキップしてフラットに
  return mat;
}

/** 2 点間に極細円柱を 1 本置く（線分の代用・全辺水平前提） */
function addSegment(scene: SceneContext, engine: EngineContext,
                    a: Vec3Tuple, b: Vec3Tuple, mat: StandardMaterialProps): void {
  const dx = b[0] - a[0];
  const dz = b[2] - a[2];
  const len = Math.hypot(dx, b[1] - a[1], dz);

  const seg = createCylinder(engine, { diameter: 0.03, height: len, tessellation: 6 });
  seg.material = mat;
  seg.position.set((a[0] + b[0]) / 2, (a[1] + b[1]) / 2, (a[2] + b[2]) / 2);

  // +Y を水平方向 (dx, 0, dz) へ向ける。Lite の Euler は XYZ 順なので
  // rz = π/2 で +Y を水平化 → ry で方位。rz 適用後の +Y は (-cos ry, 0, sin ry)
  seg.rotation.set(0, Math.atan2(dz / len, -dx / len), Math.PI / 2);
  addToScene(scene, seg);
}

/** points を順に線分でつないだ「線メッシュ」相当 */
function createLinePath(scene: SceneContext, engine: EngineContext, points: readonly Vec3Tuple[]): void {
  const mat = createLineMaterial();
  for (let i = 0; i < points.length - 1; i++) {
    addSegment(scene, engine, points[i]!, points[i + 1]!, mat);
  }
}

/** 本家 track と同じ「dist まで進んだら turn だけ回る」 */
interface Slide {
  readonly turn: number;
  readonly dist: number;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 1.5, Math.PI / 5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const START: Vec3Tuple = [2, 0, 2];
  const sphere: Mesh = createSphere(engine, { diameter: 0.25, segments: 16 });
  sphere.material = createStandardMaterial();     // ★マテリアル必須
  sphere.position.set(START[0], START[1], START[2]);
  addToScene(scene, sphere);

  // 三角形の軌道（本家 CreateLines 相当のフラット白線）
  createLinePath(scene, engine, [
    [2, 0, 2],
    [2, 0, -2],
    [-2, 0, -2],
    [2, 0, 2],   // 三角形を閉じる
  ]);

  const track: readonly Slide[] = [
    { turn: Math.PI / 2, dist: 4 },
    { turn: (3 * Math.PI) / 4, dist: 8 },
    { turn: (3 * Math.PI) / 4, dist: 8 + 4 * Math.sqrt(2) },
  ];

  let distance = 0;
  let yaw = 0;
  let p = 0;
  const SPEED = 0.05 * 30;   // 本家 step = 0.05 @30fps を deltaMs 非依存化

  onBeforeRender(scene, (deltaMs: number) => {
    const step = SPEED * (deltaMs / 1000);

    // movePOV(0, 0, step) 相当。yaw = 0 のときの前方は -Z
    sphere.position.x += -Math.sin(yaw) * step;
    sphere.position.z += -Math.cos(yaw) * step;
    distance += step;

    if (distance > track[p]!.dist) {
      yaw += track[p]!.turn;        // rotate(Axis.Y, turn, LOCAL) 相当
      sphere.rotation.y = yaw;
      p = (p + 1) % track.length;
      if (p === 0) {                // 一周したら初期状態へ戻す
        distance = 0;
        yaw = 0;
        sphere.position.set(START[0], START[1], START[2]);
        sphere.rotation.set(0, 0, 0);
      }
    }
  });

  return scene;
}
```

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/0
>
> ポイントは 2 つです。
> 1. **`movePOV` / `rotate` の代替**は、姿勢をクォータニオンで持たず **ヨー角スカラー `yaw` を単一の真実**として扱い、そこから位置更新と `rotation.y` の両方を導くこと（XZ 平面の軌道なのでこれで十分）
> 2. **線の質感**は `disableLighting = true` が要。Lite の Standard シェーダは無照明時に `clamp(emissiveColor * diffuseColor, 0, 1) * baseColor` を出力するため、両方を白にすればライト方向に依存しないフラットな白＝`CreateLines` と同じ見た目になる。円柱を十分細く（直径 0.03）すれば極細ハードエッジにも近づく
>
> 本家は毎フレーム固定の `step = 0.05` ですが、`onBeforeRender` の `deltaMs` を使って速度をフレームレート非依存にしています（30fps でも 120fps でも同じ速さ）。

### 村を歩かせる

`village.glb` の中を、キャラが `walk` アニメーションで巡回します。
本家は `Dude.babylon` を使いますが、3-06 と同じ理由（**Lite の `.babylon` ローダーはスキニング/アニメ非対応**）で
**`Xbot.glb` に置換**します。Xbot は等身大＋内部スケール 0.01＋直立用の X+90° 回転がルートノードに
**bake（焼き込み）済み**なので、本家の `scaling = 0.008` のような指定は不要です。

**Lite で押さえる 4 点**：

1. **向きは「bake クォータニオンへの合成」で与える** — Xbot ルートの `rotationQuaternion` には直立用の X+90° が入っています。
   ここに `rotation.y = yaw` を代入すると Euler プロキシが quaternion を**純 Y 回転で上書き**し、直立が消えてキャラが倒れます。
   起動時に bake を保存し、毎回 `rotationQuaternion = yaw(worldY) ⊗ bake` を合成して代入してください。
   world-Y は必ず鉛直なので、直立を保ったまま heading だけが変わります。`scaling` / `position` は `rotation` と独立なので、
   heading を毎フレーム上書きしても身長調整と併用できます。
2. **glTF 内蔵アニメは「コンテナ丸ごと」を `addToScene` する** — Lite は `addToScene` に**コンテナ全体**を渡したときだけ、
   その `animationGroups` を毎フレーム自動 tick します（`_beforeRender` に登録）。`container.entities` を個別に追加すると
   自動 tick が働かず、**スキンが bind（T）ポーズのまま止まります**。
   さらにロード直後は先頭クリップ（Xbot では `idle`）だけが再生状態なので、`stopAnimation` で全停止 → `playAnimation(walk)` へ切り替えます。
3. **移動は前節と同じ自前 `movePOV` / `rotate`** — 正面 = `-Z` 規約で `forward = (-sin yaw, 0, -cos yaw)`。
   本家 `track` の `turn` は**度数**なので `Math.PI / 180` でラジアン化します。
4. **メッシュ正面と移動方向のズレは「見た目専用オフセット」で吸収** — Xbot のメッシュ正面は `-Z` 規約と 180° ずれており、
   そのままだと進行方向と逆を向きます（＝ムーンウォーク）。移動方向自体は正しいので、**見た目の heading にだけ**
   `FRONT_OFFSET = π` を足します。移動に使う `yaw` に足すと軌道ごと反転してしまいます。

追加 import：`loadGltf, playAnimation, stopAnimation, onBeforeRender`（＋型 `AssetContainer, EngineContext, SceneContext, SceneNode`）

```typescript
interface Quat { x: number; y: number; z: number; w: number }

/** クォータニオン積 a⊗b（(a⊗b)·v = a(b(v)) の規約） */
function qmul(a: Quat, b: Quat): Quat {
  return {
    x: a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y,
    y: a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x,
    z: a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w,
    w: a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z,
  };
}

/** world-Y 軸まわり yaw のクォータニオン */
function yawQuat(yaw: number): Quat {
  return { x: 0, y: Math.sin(yaw / 2), z: 0, w: Math.cos(yaw / 2) };
}

/** 指定名のアニメーションだけを再生（他は停止） */
function playOnly(container: AssetContainer, name: string): void {
  const groups = container.animationGroups ?? [];
  let matched = false;
  for (const g of groups) {
    if (g.name === name) {
      playAnimation(g);
      g.loopAnimation = true;
      g.speedRatio = 1;
      matched = true;
    } else {
      stopAnimation(g);   // ロード直後は先頭クリップ（idle）が再生状態
    }
  }
  if (!matched) throw new Error(`アニメーション "${name}" が見つかりません。利用可能: ${groups.map((g) => g.name).join(", ")}`);
}

interface Walk {
  readonly turn: number;   // degrees
  readonly dist: number;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 1.5, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [villageContainer, xbotContainer] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/village.glb"),
    loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb"),
  ]);

  // ★ コンテナ丸ごと追加（これで animationGroups の自動 tick が有効になる）
  addToScene(scene, villageContainer);
  addToScene(scene, xbotContainer);

  playOnly(xbotContainer, "walk");   // idle を止めて walk を再生

  // Xbot ルート（直立 bake ＋ 内部スケール 0.01）。findNodeByName は 3-05 と同じもの
  const xbot = findNodeByName(xbotContainer, "Xbot");

  // 直立 bake を保存 → この後 heading を毎回これに合成する
  const q0 = xbot.rotationQuaternion;
  const bake: Quat = { x: q0.x, y: q0.y, z: q0.z, w: q0.w };

  // 身長調整：焼き込み済みの現在値へ乗算するので、内部スケールの実値に依存せず 1/3 になる
  const SIZE = 1 / 3;
  const s = xbot.scaling;
  s.set(s.x * SIZE, s.y * SIZE, s.z * SIZE);

  // 本家の歩行トラック（turn は度数）
  const track: readonly Walk[] = [
    { turn: 86, dist: 7 },
    { turn: -85, dist: 14.8 },
    { turn: -93, dist: 16.5 },
    { turn: 48, dist: 25.5 },
    { turn: -112, dist: 30.5 },
    { turn: -72, dist: 33.2 },
    { turn: 42, dist: 37.5 },
    { turn: -98, dist: 45.2 },
    { turn: 0, dist: 47 },
  ];

  const START: readonly [number, number, number] = [-6, 0, 0];
  const START_YAW = (-95 * Math.PI) / 180;   // 本家 rotate(Y, -95°) 相当の初期向き
  const FRONT_OFFSET = Math.PI;              // ★見た目専用。yaw 側に足すと軌道が壊れる

  let distance = 0;
  let yaw = START_YAW;
  let p = 0;
  const SPEED = 0.015 * 60;                  // 本家 step = 0.015 @60fps

  const applyHeading = (): void => {
    const q = qmul(yawQuat(yaw + FRONT_OFFSET), bake);   // 直立 bake を保ったまま heading だけ変える
    xbot.rotationQuaternion.set(q.x, q.y, q.z, q.w);
  };

  xbot.position.set(START[0], START[1], START[2]);
  applyHeading();

  onBeforeRender(scene, () => {
    // walk の tick はコンテナ追加時に自動登録済み。ここでは移動のみ駆動する
    const step = SPEED * (1 / 60);

    // movePOV(0, 0, step) 相当（正面 = -Z）
    xbot.position.x += -Math.sin(yaw) * step;
    xbot.position.z += -Math.cos(yaw) * step;
    distance += step;

    if (distance > track[p]!.dist) {
      yaw += (track[p]!.turn * Math.PI) / 180;   // rotate(Axis.Y, toRad(turn), LOCAL) 相当
      applyHeading();
      p = (p + 1) % track.length;
      if (p === 0) {                             // 一周したら初期状態へ戻す
        distance = 0;
        yaw = START_YAW;
        xbot.position.set(START[0], START[1], START[2]);
        applyHeading();
      }
    }
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 村を歩き回る Xbot"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/1
> （上のプレビューは [GitHub Pages 版](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/03-animation.html#村を歩かせる) でのみ表示されます）
>
> Xbot が持つクリップは `idle` / `agree` / `run` / `sad_pose` / `walk` / `sneak_pose` / `headShake` です。
> `stopAnimation` で全停止してから `walk` だけを再生しないと、`idle` が重なって足が動きません。
>
> 一番はまりやすいのは **(1) の bake 合成**と **(2) のコンテナ丸ごと追加**です。
> `xbot.rotation.y = yaw` と書くと Euler プロキシが quaternion を純 Y 回転で上書きするため、
> 直立用の X+90° が消えてキャラが床に倒れます。必ず `yaw(worldY) ⊗ bake` を合成してください。

### 三人称カメラで追う

キャラに**カメラを `parent`** すると、向きごと追従する三人称視点になります（Babylon.js の `camera.parent = dude` と同じ見え方）。
`FollowCamera` は Lite に無いため、この親子付けが代替手段です。

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 1, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
camera.parent = root;                                             // FollowCamera 代替

onBeforeRender(scene, () => {
  if (camera.beta > Math.PI / 2.2) camera.beta = Math.PI / 2.2;   // upperBetaLimit 代替
});
```

> RH→LH 変換の影響で前後が反転するので、カメラ角は `alpha = -Math.PI/2`（背後）にします。
> `upperBetaLimit` フィールドは無いので、毎フレーム `camera.beta` を clamp します。
> 追従カメラ＋ウェイポイント歩行を組み込んだ完全版は [ゴール完成版](./99-goal-final.md) を参照してください。
