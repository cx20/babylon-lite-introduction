# 4-00 コリジョン回避 (Avoiding Collisions) — 導入

> [第4部：衝突回避](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**—** Babylon.js チュートリアルは `mesh.intersectsMesh()` / `moveWithCollisions()` を使いますが、**Lite にこれらの組み込みヘルパーはありません**。代わりに **境界ボックス（AABB）や距離判定を自前**で行うか、**Ray Casting / Picking**（✅）を使います。物理は Havok V2 のサブセット（⚡）。

---

← [3-07 村を歩き回る (A Walk Around The Village)](../03-animation/3-07-walk-around-village.md) ・ [4-01 車の衝突事故を回避する (Avoiding a Car Crash)](./4-01-car-crash.md) →
