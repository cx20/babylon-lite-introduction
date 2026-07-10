# 第5部：環境改善

> 共通テンプレート・凡例は [README](./README.md) を参照。
> このパートの「空」「木」はゴール完成版の背景要素です。

---

## 5-00 より良い環境に (A Better Environment) — 導入

**—** 遠景の丘、空（スカイボックス）、スプライトの木を足して世界を作り込みます。

---

## 5-01 遠くの丘 (Distant Hills) — ○

**目的**：グレースケール画像（`heightMap.png`）の輝度を高さに変換して、起伏のある地面を作る。

**Lite 移植時の注意点**：

- **`createGroundFromHeightMap` は Lite にもあり**、本家とほぼ同じオプション（`width` / `height` / `subdivisions` / `maxHeight` / `minHeight`）を取ります。
  ただし **第 1 引数に `engine` が要る**のと、**戻り値が `Promise<Mesh>` の async** である点が違います（画像を fetch して輝度から頂点を変位させるため）。**`await` を忘れないでください。**
- **マテリアル必須** — Lite にデフォルトマテリアルは無いので `createStandardMaterial()` を明示的に割り当てます。
- **画像は絶対 URL で** — 本家の `"textures/heightMap.png"` は Playground 相対パスなので Lite では解決できません。公式アセットの絶対 URL に差し替えます。
- **async ロードは `registerScene` / `startEngine` より前に終わらせる** — 起動後にリソースを後入れするとレンダーループのクラッシュ要因になります。

追加 import：`createGroundFromHeightMap, createStandardMaterial`

```typescript
const HEIGHTMAP_URL = "https://playground.babylonjs.com/textures/heightMap.png";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 4, 10, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  // 高さマップから地形を生成（async。registerScene の前に await 完了させる）
  const ground = await createGroundFromHeightMap(engine, HEIGHTMAP_URL, {
    width: 5,
    height: 5,
    subdivisions: 10,
    maxHeight: 1,
  });
  ground.material = createStandardMaterial();   // ★マテリアル必須（無いと描画されない）
  addToScene(scene, ground);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/0
>
> `subdivisions` を上げるほど輝度を細かく拾えます。`maxHeight` / `minHeight` は変位の上下限で、
> 村の遠景として使うなら `width` / `height` / `subdivisions` を大きく（例：`100 / 100 / 100`）取ります。

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
