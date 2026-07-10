# 5-03 木のスプライト (Sprite Trees) — △（部分対応）

> [第5部：環境改善](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：木を大量のスプライトで配置する。Babylon.js の `SpriteManager` / `Sprite` は、Lite では **軸ロック billboard** に置き換えます（Sprites は部分対応 ⚡）。

追加 import：`loadSpriteAtlas, createAxisLockedBillboardSystem, addBillboardSpriteIndex, addAxisLockedBillboardSystem`

```typescript
const palm = await loadSpriteAtlas(engine, "https://playground.babylonjs.com/textures/palm.png", {
  gridSize: [1, 1],                                  // 画像全体を 1 フレームに
});
const trees = createAxisLockedBillboardSystem(palm, [0, 1, 0], { capacity: 1000 }); // Y 軸ロック

const addTree = (x: number, z: number) =>
  addBillboardSpriteIndex(trees, {
    position: [x, 0, z],
    sizeWorld: [2, 4],        // 512x1024 の比率（幅2 × 高さ4）
    pivot: [0.5, 1],          // ★下端（幹の根元）をアンカー → 地面に立つ
  });

for (let i = 0; i < 200; i++) addTree(Math.random() * -40 - 10, Math.random() * 60 - 30);
for (let i = 0; i < 200; i++) addTree(Math.random() *  40 + 10, Math.random() * 60 - 30);
addAxisLockedBillboardSystem(scene, trees);
```

> **落とし穴**：`pivot` の縦は**テクスチャ座標（0=画像上端, 1=下端）**。`[0.5, 0]` にすると木が地面から下にぶら下がります。地面に立たせるには `[0.5, 1]`（下端＝幹の根元）にします。
> フル機能の `SpriteManager` API（アニメーションフレーム等）は未対応です。

---

← [5-02 頭上の空 (Skies Above)](./5-02-skies-above.md) ・ [6-00 パーティクル噴水 (Build a Particle Fountain)](../06-particles/6-00-particle-fountain.md) →
