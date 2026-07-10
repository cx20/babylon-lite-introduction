# 2-11 Web アプリ レイアウト — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：3D canvas を Web ページ内にレイアウトする。HTML/CSS の話でエンジン制約なし。

```html
<div class="app">
  <header>My Village</header>
  <canvas id="renderCanvas"></canvas>
</div>
<style>
  .app { display: grid; grid-template-rows: auto 1fr; height: 100vh; }
  #renderCanvas { width: 100%; height: 100%; display: block; }
</style>
```

---

← [2-10 Viewer のカメラ変更](./2-10-viewer-camera.md) ・ [3-00 村のアニメーション](../03-animation/3-00-village-animation.md) →
