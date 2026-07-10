# 2-05 テクスチャを貼る (Add Texture) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：家（box＋屋根）にテクスチャを、地面に単色を割り当てる。

追加 import：`loadTexture2D`

```typescript
// テクスチャ2枚を registerScene 前にまとめてロード
// ★ Lite の loadTexture2D は async。registerScene / startEngine 前に await し終える必要がある
//   （起動後の後入れはレンダーループのクラッシュ要因）
const [roofTex, floorTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
  loadTexture2D(engine, "https://www.babylonjs-playground.com/textures/floor.png"),
]);

// 色マテリアル（Color3 → 配列 [r, g, b]）
const groundMat = createStandardMaterial();
groundMat.diffuseColor = [0, 1, 0];
ground.material = groundMat;

// テクスチャマテリアル
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;
roof.material = roofMat;

const houseMat = createStandardMaterial();
houseMat.diffuseTexture = floorTex;
house.material = houseMat;
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/5?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-05 テクスチャを貼る"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/5
>
> Lite は「UV Scaling / Offset」「Bump / Normal」「Emissive」など主要テクスチャに対応しています
> （フィールド名は IntelliSense で確認：要確認）。

---

← [2-04 基本的な家 (A Basic House)](./2-04-basic-house.md) ・ [2-06 マテリアル（面ごと/faceUV）](./2-06-materials-faceuv.md) →
