# 3-02 車を組み立てる (Building the Car) — ○

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

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

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-02 車を組み立てる"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/1
>
> ポイントは 3 つです。
> 1. **`ExtrudePolygon` の代替**として、凸輪郭なら `createMeshFromData` で直接組み立てられる
> 2. **`cloneTransformNode`** は transform / material を複製し、メッシュは GPU バッファ共有の shallow clone になる
> 3. 本家の `wheel.parent = car` は **素の親子付け**なので、Lite では `setParent` ではなく `attachToParent` を使う

---

← [3-01 メッシュの親子関係 (Mesh Parents)](./3-01-mesh-parents.md) ・ [3-03 車のマテリアル (Car Material)](./3-03-car-material.md) →
