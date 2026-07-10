# 3-03 車のマテリアル (Car Material) — ○

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：前章の押し出し車体に `car.png` を貼り、続けて車輪に `wheel.png` を貼る。車体は `ExtrudePolygon` 相当の `faceUV` / `wrap`、車輪は `CreateCylinder` 相当の `faceUV` を Lite 用に移植します。

## 車体に画像を張っていく

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

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/2?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-03 車のマテリアル / 車体に画像を張っていく"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/2
>
> `wrap: true` の側面 UV は、各辺ごとに同じ画像を繰り返すのではなく、累積周長を全周長で正規化して `faceUV[1]` の範囲へ割り当てます。輪郭中の重複点は長さ 0 の辺として累積長に影響しないため、ジオメトリ生成ではスキップして構いません。

## 車輪に画像を張っていく

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

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-03 車のマテリアル / 車輪に画像を張っていく"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/3
>
> `wheelFaceUV[1] = [0, 0.5, 0, 0.5]` は u/v の範囲を持たないため、側面の全頂点が `wheel.png` 上の同一点を参照します。これによりタイヤ外周は画像を引き伸ばさず、黒い帯として見せられます。

---

← [3-02 車を組み立てる (Building the Car)](./3-02-building-the-car.md) ・ [3-04 車輪のアニメーション (Wheel Animation)](./3-04-wheel-animation.md) →
