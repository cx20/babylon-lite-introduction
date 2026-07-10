# 2-07 リファクタリング（関数化） — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：[2-06 A の一軒家](#2-06-マテリアル面ごとfaceuv--) を、本家チュートリアルと同じく
`buildGround` / `buildBox` / `buildRoof` の「部品工場」関数へ分割する。`createScene` が「配置図」に徹する構成も本家と同じ。
純粋な TypeScript の関数化なのでそのまま可能ですが、Lite ならではの差分がいくつかあります。

**Lite 移植時の差分**：

- Lite の `create` 系 API は **engine を明示的に要求**するため、`build` 関数はすべて `engine` を引数に取る
  （本家のようなグローバル `scene` への暗黙登録は無い）。
- `loadTexture2D` は **await 必須**なので、テクスチャを使う `buildBox` / `buildRoof` は `async` 関数になる。
  `createScene` 側で `Promise.all` にまとめ、[2-06](#2-06-マテリアル面ごとfaceuv--) と同じ並列ロードを維持する。
- 本家はメッシュ生成時に自動でシーンへ登録されるが、Lite では `addToScene` が必要。
  `build` 関数は「**作って返す**」だけに徹し、シーンへの登録は `createScene` が一括で行う。
- `createBox` に `faceUV` / `wrap` が無い問題は 2-06 と同じく `createWrappedBox`（本家 `boxBuilder` の
  `wrap: true` パスの逐語移植）で対応。ヘルパー本体は [2-06 A](#2-06-マテリアル面ごとfaceuv--) を参照。

追加 import：`createCylinder, createGround, createStandardMaterial, createMeshFromData, loadTexture2D`

```typescript
// createWrappedBox は 2-06 A のものをそのまま流用（ここでは省略）

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement) {
  const scene = createSceneContext(engine);

  // カメラとライト（配置図側でセット）
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const ground = buildGround(engine);
  // buildBox / buildRoof はテクスチャロードを含む async 関数。Promise.all で並列ロードを維持
  const [box, roof] = await Promise.all([buildBox(engine), buildRoof(engine)]);

  // Lite はメッシュの自動登録が無いため、明示的にシーンへ追加
  addToScene(scene, ground);
  addToScene(scene, box);
  addToScene(scene, roof);

  return scene;
}

// 地面組み立て
function buildGround(engine: EngineContext) {
  const groundMat = createStandardMaterial();
  groundMat.diffuseColor = [0, 1, 0];
  const ground = createGround(engine, { width: 10, height: 10 });
  ground.material = groundMat;
  return ground;   // シーンへの登録は createScene が行う
}

// 家体組み立て（テクスチャロードを含むので async）
async function buildBox(engine: EngineContext) {
  const boxMat = createStandardMaterial();
  boxMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/cubehouse.png");

  // アトラスの4分割を各側面へ（上面4・底面5は見えないので未指定）
  const faceUV: (FaceUV | undefined)[] = [];
  faceUV[0] = [0.5, 0.0, 0.75, 1.0]; // 背面
  faceUV[1] = [0.0, 0.0, 0.25, 1.0]; // 正面
  faceUV[2] = [0.25, 0.0, 0.5, 1.0]; // 右側面
  faceUV[3] = [0.75, 0.0, 1.0, 1.0]; // 左側面

  const box = createWrappedBox(engine, { faceUV });   // 本家 CreateBox({ faceUV, wrap: true }) 相当
  box.material = boxMat;
  box.position.y = 0.5;
  return box;
}

// 屋根組み立て（テクスチャロードを含むので async）
async function buildRoof(engine: EngineContext) {
  const roofMat = createStandardMaterial();
  roofMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg");

  const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
  roof.material = roofMat;
  roof.scaling.x = 0.75;
  roof.rotation.z = Math.PI / 2;
  roof.position.y = 1.22;
  return roof;
}
```

> 見た目は [2-06 A の一軒家](#2-06-マテリアル面ごとfaceuv--)（`X79RM0/v/6`）と同一。本節はロジックを変えずに
> 関数へ分割しただけです。本家との唯一の構造的な違いは「**生成と登録の分離**」（`build` 関数は返すだけ、
> `addToScene` は `createScene` が一括）で、Lite にデフォルトマテリアル・自動シーン登録が無いことに由来します。

---

← [2-06 マテリアル（面ごと/faceUV）](./2-06-materials-faceuv.md) ・ [2-08 メッシュを結合 (Combining Meshes)](./2-08-combining-meshes.md) →
