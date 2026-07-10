# 2-10 Viewer のカメラ変更 — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

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

`FollowCamera` は未対応（—）。追従は [3-07](../03-animation/3-07-walk-around-village.md) のように毎フレーム target を更新するか、カメラをキャラに `parent` して実現します。

---

← [2-09 メッシュをコピー (Copying Meshes)](./2-09-copying-meshes.md) ・ [2-11 Web アプリ レイアウト](./2-11-webapp-layout.md) →
