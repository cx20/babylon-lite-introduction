# 5-01 遠くの丘 (Distant Hills) — ○

> [第5部：環境改善](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

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

## 村の場合

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

## テクスチャ

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

## 地面のテクスチャ仕上げ

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

## 家を追加

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
> その対処は [ゴール完成版](../99-goal-final.md) の「既知の落とし穴」を参照してください。

## 車を追加

完成した村を背景に、`car.glb` を Z 方向へ走らせます。

> 💡 **glTF 同士なら x 軸反転の補正は要らない** — 4-01 では「`loadGltf` が glTF ルートに x スケール `-1` を入れる」ことに悩まされましたが、
> ここでは **車も村も同じ x 軸 `-1` のルート配下**にいるので、両者は同じだけ反転します。
> つまり道路（村ローカル `x = -3`）に車を合わせるには、**本家の値 `-3` をそのまま `car` のローカル x に入れれば一致**します。
>
> 負号の補正が要るのは、4-01 の当たり判定箱のように **反転しない world 空間の自作メッシュ**と突き合わせるときだけです。
> なお z は元々反転しません。

車輪アニメは `car.glb` に入っていないので、3-05 と同じくコード側で回します。車体の前進も、本家の
`Animation("position.z")`（frame 0 → `z = 10`、frame 200 → `z = -15`、30fps、ループ）を時間から解析的に求めます。

追加 import：`onBeforeRender`（＋型 `AssetContainer, SceneNode`）

```typescript
const VALLEY_VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const CAR_URL = "https://assets.babylonjs.com/meshes/car.glb";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [village, carContainer] = await Promise.all([
    loadGltf(engine, VALLEY_VILLAGE_URL),
    loadGltf(engine, CAR_URL),
  ]);
  addToScene(scene, village);
  addToScene(scene, carContainer);

  const car = findNodeByName(carContainer, "car");   // 3-05 と同じヘルパー
  car.rotation.set(Math.PI / 2, 0, -Math.PI / 2);
  car.position.set(-3, 0.16, 8);                     // 村と同じ反転を受けるので本家の値のまま

  // car.glb に車輪アニメは無いのでコード側で回す
  const wheels = ["wheelRB", "wheelRF", "wheelLB", "wheelLF"].map((n) => findNodeByName(carContainer, n));

  // 本家キー（frame 0 → z=10、frame 200 → z=-15、30fps、ループ）を時間から解析的に
  const CAR_PERIOD = 200 / 30;                       // 秒
  const Z_START = 10;
  const Z_END = -15;
  const carZAt = (t: number): number => {
    const tt = t % CAR_PERIOD;
    return Z_START + (Z_END - Z_START) * (tt / CAR_PERIOD);
  };

  const WHEEL_RATE = -2 * Math.PI;                   // rad/秒（1 回転/秒）
  let elapsed = 0;

  onBeforeRender(scene, (deltaMs: number) => {
    const dt = deltaMs / 1000;
    elapsed += dt;
    car.position.z = carZAt(elapsed);
    for (const w of wheels) w.rotation.y += WHEEL_RATE * dt;
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/5?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-01 遠くの丘 / 車を追加"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/5
>
> `findNodeByName` は 3-05 / 3-07 / 4-01 と同じものです（完全版は Playground 参照）。
> このモデル群に glTF 内蔵アニメは無いため、車の動きは `onBeforeRender` で駆動します。
> 内蔵アニメがある場合は、コンテナ丸ごと `addToScene` するだけで自動 tick されます（→ 3-07）。

---

← [5-00 より良い環境に (A Better Environment)](./5-00-better-environment.md) ・ [5-02 頭上の空 (Skies Above)](./5-02-skies-above.md) →
