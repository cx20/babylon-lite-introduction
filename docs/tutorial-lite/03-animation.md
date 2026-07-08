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

## 3-07 村を歩き回る (A Walk Around The Village) — ○

**目的**：キャラをルートに沿って歩かせ、カメラで追う。**これがゴールの中核**です。

ポイントは 2 つ：
1. **ウェイポイント方式**で道路に沿って前進（`movePOV` 相当を「ヨー角スカラー」で表現）
2. **カメラをキャラに `parent`** して向きごと追従（Babylon.js の `camera.parent = dude` と同じ見え方）

完全なコードは [ゴール完成版](./99-goal-final.md) を参照してください。核だけ抜粋：

```typescript
// heading（進行方向のヨー角）→ Y 軸クォータニオン
const applyPose = () => {
  const h = heading * 0.5;
  root.rotationQuaternion.x = 0; root.rotationQuaternion.y = Math.sin(h);
  root.rotationQuaternion.z = 0; root.rotationQuaternion.w = Math.cos(h);
};

onBeforeRender(scene, () => {
  const t = waypoints[wp];
  let dx = t.x - root.position.x, dz = t.z - root.position.z;
  const d = Math.hypot(dx, dz);
  if (d < 0.3) { wp = (wp + 1) % waypoints.length; }
  else {
    dx /= d; dz /= d;
    heading = Math.atan2(dx, dz);
    applyPose();
    root.position.x += dx * speed; root.position.z += dz * speed;
  }
  if (camera.beta > Math.PI / 2.2) camera.beta = Math.PI / 2.2;   // upperBetaLimit 代替
});
```

> `FollowCamera` は未対応のため、**`camera.parent = root`**（親子付け）で追従します。
> RH→LH 変換の影響で前後が反転するので、カメラ角は `alpha = -Math.PI/2`（背後）にします。
