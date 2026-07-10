# 2-01 地面 (Grounding) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

追加 import：`createGround, createStandardMaterial, loadTexture2D`

```typescript
const ground = createGround(engine, { width: 24, height: 24 });
const groundMat = createStandardMaterial();
groundMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/grass.png");
ground.material = groundMat;
addToScene(scene, ground);
```

---

← [2-00 村を作る](./2-00-build-village.md) ・ [2-02 サウンドを追加](./2-02-add-sound.md) →
