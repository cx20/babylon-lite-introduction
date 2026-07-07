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

追加 import：`createStandardMaterial, loadTexture2D`

```typescript
const bodyMat = createStandardMaterial();
bodyMat.diffuseColor = [0.8, 0.1, 0.1];       // 赤いボディ
car.material = bodyMat;

// 車輪ロゴ：cylinder は faceUV に対応しているので端面にテクスチャを貼れる
const wheelMat = createStandardMaterial();
wheelMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/logo.png");
for (const w of wheels) w.material = wheelMat;
```

---

## 3-04 車輪のアニメーション (Wheel Animation) — ○

**目的**：車輪を回す。`onBeforeRender` で車軸（X 軸）まわりに回転。

```typescript
let spin = 0;
onBeforeRender(scene, () => {
  spin += 0.15;
  for (const w of wheels) {
    // 車軸(Z で寝かせた後の回転)まわりに回す簡易版：X 軸ヨー成分を更新
    w.rotationQuaternion.x = Math.sin(spin / 2);
    // 寝かせ(Z)成分は維持したいので、厳密には合成が必要。回転見せが目的なら下記の別法が簡単
  }
});
```

> 厳密な合成が面倒なら、**キーフレームのプロパティアニメ**が使えます：
> ```typescript
> // 追加 import: createAnimationManager, createPropertyAnimationClip, createPropertyAnimationGroup
> const mgr = createAnimationManager({ engine });
> const clip = createPropertyAnimationClip("spin", [
>   { path: "rotationQuaternion", keys: [ /* 0→360° のクォータニオン列 */ ], quaternion: true }
> ]);
> // createPropertyAnimationGroup(mgr, wheel, clip, { loop: true });
> ```
> 回転を見せるだけなら `onBeforeRender` 版で十分です。

---

## 3-05 車のアニメーション (Car Animation) — ○

**目的**：車を経路に沿って動かす。`onBeforeRender` で位置を更新（キーフレームでも可）。

```typescript
let t = 0;
onBeforeRender(scene, () => {
  t += 0.02;
  car.position.z = Math.sin(t) * 5;   // 前後に往復
});
```

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
