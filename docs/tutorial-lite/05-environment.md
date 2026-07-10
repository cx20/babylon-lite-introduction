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
> `subdivisions` を上げるほど輝度を細かく拾えます。`maxHeight` / `minHeight` は変位の上下限です。

### 村の場合

村の谷を作るときは、専用のハイトマップ `villageheightmap.png` を使い、`150 × 150` の大地面にします。
カメラを引く（`radius = 200`）と全景が見えます。注意点は上と同じで、**`engine` 引数・`await`・マテリアル明示**の 3 点です。

```typescript
const HEIGHTMAP_URL = "https://assets.babylonjs.com/environments/villageheightmap.png";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  // 谷の環境用の大地面（async。registerScene の前に await 完了させる）
  const largeGround = await createGroundFromHeightMap(engine, HEIGHTMAP_URL, {
    width: 150,
    height: 150,
    subdivisions: 20,
    minHeight: 0,
    maxHeight: 10,
  });
  largeGround.material = createStandardMaterial();
  addToScene(scene, largeGround);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 200, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([4, 1, 0], 1));

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘 / 村の場合"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/1
>
> `villageheightmap.png` は `assets.babylonjs.com` の絶対 URL をそのまま使えます。
> `150 × 150` に対して `subdivisions` は `20` と粗めですが、遠景の丘としてはこれで十分です。

### テクスチャ

大地面に `valleygrass.png` の草テクスチャを貼ります。

ここでは **async が 2 つ**（`createGroundFromHeightMap` と `loadTexture2D`）出てきます。互いに独立なので
`Promise.all` で並列に待つのが素直です。本家の `new BABYLON.Texture(url)` は同期的な書き味ですが、
Lite では 2-05 と同じく `await loadTexture2D(engine, url)` に置き換えます。

追加 import：`loadTexture2D`

```typescript
const HEIGHTMAP_URL = "https://assets.babylonjs.com/environments/villageheightmap.png";
const GRASS_URL = "https://assets.babylonjs.com/environments/valleygrass.png";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  // 地形メッシュと草テクスチャを並列に用意（registerScene の前に await 完了）
  const [largeGround, grassTex] = await Promise.all([
    createGroundFromHeightMap(engine, HEIGHTMAP_URL, {
      width: 150,
      height: 150,
      subdivisions: 20,
      minHeight: 0,
      maxHeight: 10,
    }),
    loadTexture2D(engine, GRASS_URL),
  ]);

  const largeGroundMat = createStandardMaterial();
  largeGroundMat.diffuseTexture = grassTex;
  largeGround.material = largeGroundMat;
  addToScene(scene, largeGround);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 200, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([2, 1, 0], 1));

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/2?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘 / テクスチャ"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/2
>
> `createStandardMaterial()` に名前引数は要りません（本家の `new StandardMaterial("largeGroundMat", scene)` 相当）。
> ハイトマップ・テクスチャとも `assets.babylonjs.com` の絶対 URL をそのまま使えます。

### 地面のテクスチャ仕上げ

透過付きの村地面（`villagegreen.png`）を、起伏のある谷の大地面の上へわずかに浮かせて重ねます。
村地面の透明部分から、下の谷の草地が透けて見えます。

> ⚠️ **Lite の透過モデルは本家と違う** — 本家は `groundMat.diffuseTexture.hasAlpha = true` だけで
> diffuse テクスチャのアルファが滑らかにアルファブレンドされます。
> **Lite の diffuse テクスチャのアルファは discard 専用**（`alphaCutOff` によるアルファテスト＝カットアウト）で、
> 出力アルファには混ざりません（シェーダのコメントいわく *"matching BJS ALPHATEST without ALPHAFROMDIFFUSE"*）。
>
> 滑らかなアルファブレンドを得るには、**同じテクスチャを `opacityTexture` にも割り当て**ます。
> opacity のフラグメントが `alpha *= texAlpha` を行い、これで `_alphaBlend` が有効になって透明部分が実際に透過します。
> つまり **`diffuseTexture` と `opacityTexture` の両方に同じテクスチャを割り当てるのが `hasAlpha = true` 相当**です。

追加 import：`createGround`

```typescript
const VILLAGE_GREEN_URL = "https://assets.babylonjs.com/environments/villagegreen.png";
const VALLEY_GRASS_URL = "https://assets.babylonjs.com/environments/valleygrass.png";
const HEIGHTMAP_URL = "https://assets.babylonjs.com/environments/villageheightmap.png";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [largeGround, villageGreenTex, valleyGrassTex] = await Promise.all([
    createGroundFromHeightMap(engine, HEIGHTMAP_URL, {
      width: 150,
      height: 150,
      subdivisions: 20,
      minHeight: 0,
      maxHeight: 10,
    }),
    loadTexture2D(engine, VILLAGE_GREEN_URL),
    loadTexture2D(engine, VALLEY_GRASS_URL),
  ]);

  // 村の地面（24×24）。hasAlpha = true 相当：diffuse と opacity の両方に同じテクスチャを割り当てる
  const groundMat = createStandardMaterial();
  groundMat.diffuseTexture = villageGreenTex;
  groundMat.opacityTexture = villageGreenTex;   // ★ これが無いと透明部分が抜けない
  const ground = createGround(engine, { width: 24, height: 24 });
  ground.material = groundMat;
  addToScene(scene, ground);

  // 谷の大地面（起伏＋草）。村地面のわずかに下へ置いて Z-fighting を避ける
  const largeGroundMat = createStandardMaterial();
  largeGroundMat.diffuseTexture = valleyGrassTex;
  largeGround.material = largeGroundMat;
  largeGround.position.y = -0.01;               // 本家どおり
  addToScene(scene, largeGround);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘 / 地面のテクスチャ仕上げ"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/3
>
> `largeGround.position.y = -0.01` は本家どおりで、2 枚の地面が同一平面で重なる Z-fighting を避けるためのものです。
> `opacityTexture` を割り当て忘れると、村地面の透明部分がアルファテストで抜けるだけになり、
> 境界がギザギザになったり（`alphaCutOff` 次第では）まったく抜けなかったりします。

### 家を追加

ここまで手作業で組み立ててきた地形・地面・家 16 棟・ライトが 1 つにまとまった `valleyvillage.glb` を読み込みます。
テクスチャも透過も起伏も glTF 側に入っているので、**読み込んで `addToScene` するだけ**です。

追加 import：`loadGltf`

```typescript
const VALLEY_VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  // 完成済みの村を読み込み、コンテナ丸ごとシーンへ追加
  const village = await loadGltf(engine, VALLEY_VILLAGE_URL);
  addToScene(scene, village);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/4?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘 / 家を追加"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/4
>
> 本家 `SceneLoader.ImportMeshAsync` は `loadGltf`（async）に置き換えます。
> glTF は 3-07 と同じく **コンテナ丸ごと** `addToScene` してください。配下のノード階層が再帰的に追加されます
> （アニメがあれば自動 tick もされますが、このモデルにアニメはありません）。
>
> 4-01 で問題になった **glTF ルートの x 軸反転**は、ここでは表面化しません。座標を自分で与える箇所が無く、
> 自作メッシュと glTF を混在させないためです。
>
> なお、カメラを村の内側へ入れると家の壁が裏返って見えることがあります（単面ポリゴンの片面描画）。
> その対処は [ゴール完成版](./99-goal-final.md) の「既知の落とし穴」を参照してください。

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
