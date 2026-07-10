# 2-02 サウンドを追加 — ✕

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**Lite には音声エンジンがありません**（`KHR_audio` も非対応）。どうしても鳴らす場合は、エンジン外で **Web Audio API を直接**使います。

```typescript
// エンジン機能ではない代替（ブラウザ標準）
const audio = new Audio("https://playground.babylonjs.com/sounds/cellolong.wav");
audio.loop = true;
addEventListener("pointerdown", () => audio.play(), { once: true }); // 自動再生制限のためユーザー操作後に
```

> 3D 定位（`BABYLON.Sound` の spatial）に相当する機能はありません。

---

← [2-01 地面 (Grounding)](./2-01-grounding.md) ・ [2-03 メッシュを設置 (Place and Scale)](./2-03-place-and-scale.md) →
