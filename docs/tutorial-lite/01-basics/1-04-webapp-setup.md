# 1-04 最初の web アプリのセットアップ — ○

> [第1部：基礎](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：Playground でなく実プロジェクトとして構築する。

Lite は **TypeScript / Vite ネイティブ**。エンジン側の制約はなく、パッケージ名が変わるだけです。

```bash
npm create vite@latest my-app -- --template vanilla-ts
cd my-app
npm install @babylonjs/lite
```

```typescript
// src/main.ts … 共通テンプレートと同じ（import 元は "@babylonjs/lite"）
```

`index.html` に `<canvas id="renderCanvas"></canvas>` を置けば、Playground と同じコードがそのまま動きます。

---

← [1-03 モデルを扱う](./1-03-models.md) ・ [2-00 村を作る](../02-village/2-00-build-village.md) →
