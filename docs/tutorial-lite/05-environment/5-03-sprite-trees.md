# 5-03 木のスプライト (Sprite Trees) — △（部分対応）

> [第5部：環境改善](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：村の周囲に `palm.png` のスプライトの木を大量に配置する。

本家の `SpriteManager` / `Sprite` は、Lite では **ビルボードシステム**に置き換えます（Sprites は部分対応 ⚡）。
手順は 4 段階です。

1. **`loadSpriteAtlas(engine, url, { gridSize })`** で画像からアトラスを作る
2. **`createFacingBillboardSystem(atlas, opts)`** でシステムを作る（`facing` = 常時カメラ正対＝本家 `SpriteManager` 相当）
3. **`addBillboardSprite(system, { position, sizeWorld })`** で 1 本ずつ追加する
4. **`addFacingBillboardSystem(scene, system)`** でシーンへ登録する

**Lite 移植時の注意点**：

- ⚠️ **`gridSize` は「セルのピクセルサイズ」**（グリッドの分割数ではない）。`palm.png` は 512×1024 なので、
  画像全体を 1 フレームにするには **`gridSize: [512, 1024]`** と書きます。本家 `SpriteManager` の
  `{ width: 512, height: 1024 }`（セルサイズ）に対応します。`[1, 1]` と書くと 1×1 ピクセルのセルに分割されてしまいます。
- **`blendMode` は文字列ではなくブレンド記述子オブジェクト** — 透過 PNG を抜くには `billboardBlendCutout` を渡します
  （`billboardBlendAlpha` / `billboardBlendPremultiplied` / `billboardBlendAdditive` もあります）。
- **`pivot` の既定は中心 `[0.5, 0.5]`** — 底を地面（`y = 0`）へ接地させるには、`position.y` を高さの半分にするか、
  `pivot: [0.5, 1]`（下端＝幹の根元）を指定します。`pivot` の縦はテクスチャ座標（`0` = 画像上端、`1` = 下端）です。
- **スプライトはシーン直下**なので、glTF ルートの x 軸 `-1` 反転（→ [4-01](../04-collisions/4-01-car-crash.md)）は掛かりません。木はワールド座標そのままです。

追加 import：`loadSpriteAtlas, createFacingBillboardSystem, addBillboardSprite, addFacingBillboardSystem, billboardBlendCutout`

```typescript
const PALM_URL = "https://playground.babylonjs.com/textures/palm.png";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  camera.upperBetaLimit = Math.PI / 2.2;
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [village, palmAtlas] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/valleyvillage.glb"),
    loadSpriteAtlas(engine, PALM_URL, { gridSize: [512, 1024] }),   // ★セルのピクセルサイズ
    loadSkybox(scene, "https://playground.babylonjs.com/textures/skybox", ".jpg", 150),
  ]);
  addToScene(scene, village);

  // 木のビルボードシステム（常時カメラ正対・透過はカットアウトで抜く）
  const treeSystem = createFacingBillboardSystem(palmAtlas, {
    capacity: 1000,                    // 木 500 本 × 2
    blendMode: billboardBlendCutout,   // ★文字列ではなくブレンド記述子
    alphaCutoff: 0.5,
  });

  // palm.png は縦長（512:1024 = 1:2）。sizeWorld も 1:2 で木らしい縦横比にする
  const TREE_SIZE: [number, number] = [1, 2];
  const TREE_Y = TREE_SIZE[1] / 2;     // pivot が中心なので、底を y=0 に接地させる

  // 本家の乱数配置をそのまま（村の左手前と右奥に散らす）
  for (let i = 0; i < 500; i++) {
    addBillboardSprite(treeSystem, {
      position: [Math.random() * -30, TREE_Y, Math.random() * 20 + 8],
      sizeWorld: TREE_SIZE,
    });
  }
  for (let i = 0; i < 500; i++) {
    addBillboardSprite(treeSystem, {
      position: [Math.random() * 25 + 7, TREE_Y, Math.random() * -35 + 8],
      sizeWorld: TREE_SIZE,
    });
  }

  addFacingBillboardSystem(scene, treeSystem);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/7?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-03 木のスプライト"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/7
>
> 本家の `Sprite` は既定サイズ 1 相当ですが、そのままだと村に対して大きいので `sizeWorld = [1, 2]` に縮めています。
> フル機能の `SpriteManager` API（アニメーションフレーム等）は未対応です。

## Y 軸ロックにする

`createFacingBillboardSystem` は常にカメラへ正対します（ピッチも追従）。木のように **上下方向は立たせたまま、
水平方向だけカメラを向かせたい**場合は、Y 軸ロック版を使います。

追加 import：`createAxisLockedBillboardSystem, addBillboardSpriteIndex, addAxisLockedBillboardSystem`

```typescript
const trees = createAxisLockedBillboardSystem(palmAtlas, [0, 1, 0], { capacity: 1000 });   // Y 軸ロック

const addTree = (x: number, z: number) =>
  addBillboardSpriteIndex(trees, {
    position: [x, 0, z],
    sizeWorld: [2, 4],
    pivot: [0.5, 1],        // 下端（幹の根元）をアンカー → そのまま地面に立つ
  });

for (let i = 0; i < 200; i++) addTree(Math.random() * -40 - 10, Math.random() * 60 - 30);
for (let i = 0; i < 200; i++) addTree(Math.random() *  40 + 10, Math.random() * 60 - 30);
addAxisLockedBillboardSystem(scene, trees);
```

> `addBillboardSprite` はハンドルを返し、`addBillboardSpriteIndex` はインデックスを返します。あとから
> 個別に動かす／消すなら前者、追加しっぱなしなら後者で十分です。
> この Y 軸ロック版は [ゴール完成版](../99-goal-final.md) で使っています。

---

← [5-02 頭上の空 (Skies Above)](./5-02-skies-above.md) ・ [6-00 パーティクル噴水 (Build a Particle Fountain)](../06-particles/6-00-particle-fountain.md) →
