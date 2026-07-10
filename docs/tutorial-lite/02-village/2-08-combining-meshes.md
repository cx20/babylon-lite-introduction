# 2-08 メッシュを結合 (Combining Meshes) — △

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

Babylon.js の `Mesh.MergeMeshes` 相当のヘルパーは **ありません**。目的（ドローコール削減 / 一体化）別の代替：

- **同一メッシュを大量配置** → thin instances（`setThinInstances`、下記）
- **家体+屋根を1ユニットに** → `createTransformNode` + `setParent` で親子付け（[2-09](#2-09-メッシュをコピー-copying-meshes--)）
- **ブーリアンで一体化** → `CSG2`（union / subtract / intersect、Scenes 90-91）
- どうしても頂点結合が必要なら手動で頂点バッファを結合

**同一メッシュを大量配置する場合**（追加 import：`createBox, setThinInstances`）：

```typescript
// 各インスタンスは 16 float の列優先 4x4 行列。ここでは平行移動のみ
function translation(x: number, y: number, z: number): Float32Array {
  const m = new Float32Array(16);
  m[0] = 1; m[5] = 1; m[10] = 1; m[15] = 1;
  m[12] = x; m[13] = y; m[14] = z;
  return m;
}

const box = createBox(engine, 1);   // createBox の第2引数は数値（一辺の長さ）
box.material = createStandardMaterial();

const N = 8;
const data = new Float32Array(N * 16);
for (let i = 0; i < N; i++) data.set(translation(i * 2 - 7, 0.5, 4), i * 16);
setThinInstances(box, data, N);
addToScene(scene, box);
```

> Babylon.js の `InstancedMesh` API（🚫）は非対応。thin instances（`setThinInstances` / `setThinInstanceColors`）を使います。

---

← [2-07 リファクタリング（関数化）](./2-07-refactoring.md) ・ [2-09 メッシュをコピー (Copying Meshes)](./2-09-copying-meshes.md) →
