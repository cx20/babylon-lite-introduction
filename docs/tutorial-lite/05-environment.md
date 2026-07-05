# 第5部：環境改善

> 共通テンプレート・凡例は [README](./README.md) を参照。
> このパートの「空」「木」はゴール完成版の背景要素です。

---

## 5-00 より良い環境に (A Better Environment) — 導入

**—** 遠景の丘、空（スカイボックス）、スプライトの木を足して世界を作り込みます。

---

## 5-01 遠くの丘 (Distant Hills) — ○

**目的**：ハイトマップから起伏のある地形を作る。

追加 import：`createGroundFromHeightMap`（引数は要確認：`engine` 要否／アップロード手順）

```typescript
// ハイトマップ画像の輝度で頂点を上下させる（GPU テクスチャ → 頂点変位）
const hills = await createGroundFromHeightMap(
  "https://playground.babylonjs.com/textures/heightMap.png",
  { width: 100, height: 100, subdivisions: 100, minHeight: 0, maxHeight: 10 }
);
addToScene(scene, hills);
```

> 機能比較表では「Ground from Heightmap ✅」。ただし関数シグネチャ（`engine` 引数の有無、返り値のアップロード手順）は Lite のバージョンで差異が出ることがあるため、Playground の IntelliSense で確認してください。

---

## 5-02 頭上の空 (Skies Above) — ○（要アセット）

**目的**：スカイボックスを表示する。Babylon.js の `CubeTexture` + `SKYBOX_MODE` は、Lite では **`loadEnvironment`**（スカイボックス＋IBL 環境光）に集約されています。

追加 import：`loadEnvironment`

```typescript
await loadEnvironment(scene, "https://assets.babylonjs.com/environments/environmentSpecular.env", {
  brdfUrl: "https://<到達可能なホスト>/brdf-lut.png",   // ★必須：BRDF LUT の PNG
  skyboxUrl: "https://assets.babylonjs.com/environments/backgroundSkybox.dds",
  skyboxSize: 1000,
});
```

- `.env`（specular IBL）＋ `.dds` スカイボックス＋ BRDF LUT で、PBR マテリアルが正しく光ります。
- `brdfUrl` は**必須**。到達可能な URL を用意できない場合はスカイボックスを省略してください（背景色のみになります）。
- 6 枚 JPG のキューブスカイボックス（`SKYBOX_MODE`）も「Cubemap Skybox ✅」として対応していますが、`loadEnvironment` が最も手軽です。

---

## 5-03 木のスプライト (Sprite Trees) — △（部分対応）

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
