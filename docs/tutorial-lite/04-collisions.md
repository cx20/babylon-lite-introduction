# 第4部：衝突回避

> 共通テンプレート・凡例は [README](./README.md) を参照。

---

## 4-00 コリジョン回避 (Avoiding Collisions) — 導入

**—** Babylon.js チュートリアルは `mesh.intersectsMesh()` / `moveWithCollisions()` を使いますが、Lite にこれらの**組み込みヘルパーは未確認**です。代わりに **境界ボックス（AABB）や距離判定を自前**で行うか、**Ray Casting / Picking**（✅）を使います。物理は Havok V2 のサブセット（⚡）。

---

## 4-01 車の衝突事故を回避する (Avoiding a Car Crash) — △

**目的**：接近を検知して停止／迂回する。

### 方法A：距離（円）判定 — 最も簡単

```typescript
const near = (a: {x:number;z:number}, b: {x:number;z:number}, r: number) =>
  Math.hypot(a.x - b.x, a.z - b.z) < r;

const obstacles = [{ x: 5, z: 0 }, { x: -3, z: 4 }];

onBeforeRender(scene, () => {
  const willHit = obstacles.some((o) => near(root.position, o, 1.5));
  if (!willHit) {
    root.position.x += Math.sin(heading) * speed;
    root.position.z += Math.cos(heading) * speed;
  } else {
    // 迂回：向きを少し変える等
    heading += 0.05;
    applyPose();
  }
});
```

### 方法B：AABB（境界ボックス）判定

読み込んだメッシュは `boundMin` / `boundMax`（ワールド AABB）を持ちます。2 つの AABB が重なるかで判定できます。

```typescript
const overlapAABB = (a: any, b: any) =>
  a.boundMin.x <= b.boundMax.x && a.boundMax.x >= b.boundMin.x &&
  a.boundMin.y <= b.boundMax.y && a.boundMax.y >= b.boundMin.y &&
  a.boundMin.z <= b.boundMax.z && a.boundMax.z >= b.boundMin.z;

// onBeforeRender 内で overlapAABB(carMesh, houseMesh) を判定
```

> より厳密な当たり判定が必要なら Ray Casting / Picking（GPU ID パス＋ CPU レイ/三角形、thin-instance・変形メッシュ対応）が使えます。
