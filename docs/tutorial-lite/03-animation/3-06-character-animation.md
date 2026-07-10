# 3-06 キャラクターのアニメーション (Character Animation) — ○（アセットは glTF）

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

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

← [3-05 車のアニメーション (Car Animation)](./3-05-car-animation.md) ・ [3-07 村を歩き回る (A Walk Around The Village)](./3-07-walk-around-village.md) →
