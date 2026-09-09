# 8-03 VR の世界へ (Going Virtual) — ✕（API は実装済み・ブラウザ未対応）

> [第8部：世界の見方](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：WebXR で VR 表示にする。

**Lite 対応状況**（v1.28.0 ソースで確認）：

- **v1.25.0 で WebXR サポートが入りました**（[#537](https://github.com/BabylonJS/Babylon-Lite/pull/537)）。
  `enterXr` / `exitXr`、両眼カメラ（`createXrCamera` / `updateXrCameraForView`）、入力（`createXrInputManager` /
  `readXrController`）、コントローラーモデル、ハンドトラッキング（`createXrHandTracking`）、
  ポインタ（`createXrPointer`）、テレポート（`createXrTeleportation`）まで一式が公開されています。
  本家の `scene.createDefaultXRExperienceAsync()` のような「全部入り」ヘルパーではなく、必要な部品を組む流儀です。
- **ただし現時点ではどのブラウザでも動きません**。Lite の XR は **WebXR × WebGPU のドラフト仕様
  `XRGPUBinding` を前提**にしており、これを実装したブラウザがまだ存在しないためです。
  ソース自身が *"Requires the **draft** WebXR/WebGPU binding (`XRGPUBinding`), which no browser implements yet"* と明記し、
  `isWebGpuXrSupported()` は**現状すべての環境で `false`** を返します（`enterXr` はその場合に例外を投げます）。
- したがって本章の判定は **✕ 据え置き**です。実装は「先行実装（forward-looking）」であり、
  **今 VR を届けたい用途では引き続き Babylon.js（WebGL の WebXR 経路）が適しています**。

```typescript
// 参考：Lite での入口。ブラウザが XRGPUBinding を実装するまで isXrSessionSupported() は false のまま
import { enableXrCompatibleAdapter, enterXr, exitXr, isXrSessionSupported } from "@babylonjs/lite";

enableXrCompatibleAdapter();          // ★ createEngine より前に呼ぶ（アダプタを XR 互換で取得させる）
// … createEngine → createSceneContext → registerScene …

if (await isXrSessionSupported("immersive-vr")) {
  const xr = await enterXr(scene, { mode: "immersive-vr", referenceSpaceType: "local-floor" });
  // セッション中は通常のキャンバス描画ループが止まり、終了時に自動で再開する
  await exitXr(xr);
} else {
  console.log("この環境では WebGPU XR (XRGPUBinding) が使えません");
}
```

> この章は**現状のブラウザでは目的を達成できません**（API は用意済みなので、`XRGPUBinding` の実装が進めば判定は変わります）。

---

← [8-02 キャラを追う (Follow That Character)](./8-02-follow-character.md)
