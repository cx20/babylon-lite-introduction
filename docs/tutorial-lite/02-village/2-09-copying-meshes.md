# 2-09 メッシュをコピー (Copying Meshes) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：[2-07 の関数化](#2-07-リファクタリング関数化--)を一歩進め、`buildHouse(width)` で
「**家体 + 屋根**」を1ユニットにまとめ、コピー可能にする。`width == 2` なら `semihouse.png`（長屋）、
それ以外は `cubehouse.png`（一軒家）を貼ります。

**Lite 移植時の注意点**（v1.8.0 の d.ts / lib ソースで確認）：

- 本家は `buildHouse` の最後で
  `BABYLON.Mesh.MergeMeshes([box, roof], true, false, null, false, true)`（最後の引数 `multiMultiMaterials`）で
  **2 テクスチャを保持したまま単一メッシュへ結合**します。Lite には **`MergeMeshes` が存在せず**、`Mesh` は
  `material` を **1 つだけ**持ち **SubMesh / MultiMaterial の概念が無い**ため、この結合は表現できません。
- 代わりに `createTransformNode("house")` で親ノードを作り、`setParent(box, house)` / `setParent(roof, house)` で
  1ユニット化します。`setParent` は本家 `TransformNode.setParent()` 同様、**ワールド変換を保ったまま**親子付けし、
  scene-graph の children 配列も維持します。
- `addToScene(scene, house)` は **children を再帰的に追加**する（`scene-core.js` の実装で確認）ので、
  `box` / `roof` を個別に `addToScene` する必要はありません。
- 「コピー」自体は **`cloneTransformNode(house)`** で行います。ノードツリーを deep-clone し、メッシュは
  **GPU バッファ共有の shallow-clone**（本家 `clone` よりむしろ instance に近いメモリ特性）になります。
  同一メッシュの**大量配置**なら thin instances（[2-08](#2-08-メッシュを結合-combining-meshes--) / `setThinInstances`）も選択肢です。
- `createBox(engine, size?)` は v1.8.0 でも数値 1 つのみ（`width` / `faceUV` / `wrap` は無い）なので、家体は引き続き
  `createWrappedBox`（本家 `boxBuilder` の `wrap: true` パスの逐語移植、[2-06 A](#2-06-マテリアル面ごとfaceuv--) 参照）で作ります。`width` はこの自作関数側でサポート済みです。
- `SceneContext` 型は v1.8.0 で公開エクスポートされたため、2-07 の `ReturnType` 導出から直接 import に変更できます。

追加 import：`createCylinder, createGround, createStandardMaterial, createMeshFromData, createTransformNode, setParent, loadTexture2D`（＋型 `SceneContext, Mesh, TransformNode`）／コピーには `cloneTransformNode`

```typescript
// createWrappedBox は 2-06 A のものをそのまま流用（width オプション対応済み。ここでは省略）

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const ground = buildGround(engine);
  const house = await buildHouse(engine, 2);   // 家の幅 1（一軒家）または 2（長屋）

  addToScene(scene, ground);
  // house は TransformNode。addToScene が children（box / roof）を再帰追加するので、この1回でユニット全体が入る
  addToScene(scene, house);

  return scene;
}

// 家組み立て（家体 + 屋根 の1ユニット化）
async function buildHouse(engine: EngineContext, width: number): Promise<TransformNode> {
  // buildBox / buildRoof はテクスチャロードを含む async なので並列に
  const [box, roof] = await Promise.all([buildBox(engine, width), buildRoof(engine, width)]);

  // 本家: return BABYLON.Mesh.MergeMeshes([box, roof], true, false, null, false, true);
  // Lite: SubMesh / MultiMaterial が無く単一メッシュへの結合は不可のため、TransformNode の
  //       親子付けで「1ユニットとして動かせる家」を表現する。コピーは cloneTransformNode(house) で行える。
  const house = createTransformNode("house");
  setParent(box, house);
  setParent(roof, house);
  return house;
}

// 家体組み立て（width で貼るテクスチャと faceUV を切り替え）
async function buildBox(engine: EngineContext, width: number): Promise<Mesh> {
  const boxMat = createStandardMaterial();
  boxMat.diffuseTexture = await loadTexture2D(
    engine,
    width == 2
      ? "https://assets.babylonjs.com/environments/semihouse.png"
      : "https://assets.babylonjs.com/environments/cubehouse.png",
  );

  // 面ごとの UV（上面4・底面5は見えないので未指定）
  const faceUV: (FaceUV | undefined)[] = [];
  if (width == 2) {
    faceUV[0] = [0.6, 0.0, 1.0, 1.0]; // 背面
    faceUV[1] = [0.0, 0.0, 0.4, 1.0]; // 正面
    faceUV[2] = [0.4, 0.0, 0.6, 1.0]; // 右側面（窓なし壁を共有）
    faceUV[3] = [0.4, 0.0, 0.6, 1.0]; // 左側面（窓なし壁を共有）
  } else {
    faceUV[0] = [0.5, 0.0, 0.75, 1.0]; // 背面
    faceUV[1] = [0.0, 0.0, 0.25, 1.0]; // 正面
    faceUV[2] = [0.25, 0.0, 0.5, 1.0]; // 右側面
    faceUV[3] = [0.75, 0.0, 1.0, 1.0]; // 左側面
  }

  const box = createWrappedBox(engine, { width, faceUV });   // 本家 CreateBox({ width, faceUV, wrap: true }) 相当
  box.material = boxMat;
  box.position.y = 0.5;
  return box;
}

// 屋根組み立て（width で長さを伸ばす）
async function buildRoof(engine: EngineContext, width: number): Promise<Mesh> {
  const roofMat = createStandardMaterial();
  roofMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg");

  const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
  roof.material = roofMat;
  roof.scaling.x = 0.75;
  roof.scaling.y = width;          // 長屋（width:2）は屋根の長さも2倍に
  roof.rotation.z = Math.PI / 2;
  roof.position.y = 1.22;
  return roof;
}

// 地面組み立て（2-07 と同じ）
function buildGround(engine: EngineContext): Mesh {
  const groundMat = createStandardMaterial();
  groundMat.diffuseColor = [0, 1, 0];
  const ground = createGround(engine, { width: 10, height: 10 });
  ground.material = groundMat;
  return ground;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/9?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-09 メッシュをコピー"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/9
>
> **コピーするには** `cloneTransformNode(house)` を使います（ノードツリーは deep-clone、メッシュは GPU バッファ共有の
> shallow-clone）。複製ごとに `position` などをずらせば村になります：
>
> ```typescript
> const house = await buildHouse(engine, 1);
> addToScene(scene, house);
> for (let i = 1; i < 4; i++) {
>   const copy = cloneTransformNode(house);   // 家体+屋根ごと複製（バッファは共有）
>   copy.position.x = i * 2;
>   addToScene(scene, copy);
> }
> ```
>
> Babylon.js の `InstancedMesh`（`createInstance()`）は非対応。**同一メッシュを数百〜数千**並べて
> ドローコールを抑えたい場合は thin instances（[2-08](#2-08-メッシュを結合-combining-meshes--) / `setThinInstances`）を使います。

## 村を配置する — 家を複製して並べる

**目的**：`buildHouse(1)`（一軒家）と `buildHouse(2)`（長屋）の**2軒をテンプレート**に、
`places`（`[家のタイプ, Y回転, x, z]` の 17 件）を複製して村を作る。本家は
`detached_house.createInstance("house" + i)` で GPU インスタンスを生成し、
`rotation.y` / `position.x` / `position.z` を各戸に設定します。

**Lite 移植時の注意点**（v1.8.0 の d.ts / lib ソースで確認）：

- Lite に `InstancedMesh`（`createInstance`）は **無い**。GPU インスタンシングの等価物は thin instance で、
  **階層ごと複製**するには **`createHierarchyInstancePool(root, capacity)`** ＋ **`addHierarchyInstance(pool, matrix)`** を使います。
  ルート行列が各子メッシュ（box / roof）の thin instance バッファへ展開されるため、2-09 前半の
  `TransformNode` 家ユニットをそのまま「1 戸」として複製できます。
- **プール化するとテンプレートのメッシュは `activeCount = 0` の雛形に変わり描画されなくなります**（実装確認済み）。
  本家はテンプレート 2 軒を `places[0]`・`[1]` と同座標に置いていますが、Lite 版では**テンプレートの配置は不要**。
  `places` の 17 件を全戸インスタンス化すれば、描画結果は本家と同じ 17 軒です。
- **呼び出し順の制約**：`createHierarchyInstancePool` は「**`setParent` 済み 〜 `registerScene()` より前**」に呼ぶこと。
  テンプレート各メッシュの `worldMatrix` スナップショットからルート相対行列を計算するためです。
- インスタンスの姿勢は node の `rotation` / `position` ではなく**ワールド行列**で渡します。`mat4Compose`（T·R·S）を使い、
  `rotation.y = θ` は四元数 `(0, sin(θ/2), 0, cos(θ/2))` に変換します。
- 数戸程度なら前半の `cloneTransformNode(house)` でも複製できます（node として `position` / `rotation` を本家同様に扱える）。
  本章のテーマは `createInstance` の描画効率なので、ここでは thin instance プールを採用しています。
  実行時に動かすなら `setHierarchyInstanceMatrix` / `removeHierarchyInstance` も使えます。

追加 import：`createHierarchyInstancePool, addHierarchyInstance, mat4Compose`（＋型 `Mat4`）

```typescript
// buildHouse / buildBox / buildRoof / buildGround / createWrappedBox は 2-09 前半のものを流用
// （ここでは省略。buildGround の地面だけ { width: 15, height: 16 } に広げる）

/** places 1要素 = [家のタイプ(1|2), Y回転, x, z] */
type Place = readonly [number, number, number, number];

/**
 * 本家 houses[i] の rotation.y / position.x / position.z 相当の
 * インスタンス用ワールド行列。rotation.y = θ の四元数は (0, sin(θ/2), 0, cos(θ/2))。
 * mat4Compose(tx, ty, tz, qx, qy, qz, qw, sx, sy, sz)
 */
function placeMatrix(rotY: number, x: number, z: number): Mat4 {
  const half = rotY / 2;
  return mat4Compose(x, 0, z, 0, Math.sin(half), 0, Math.cos(half), 1, 1, 1);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const ground = buildGround(engine);

  // テンプレート 2 軒。プール化で描画数0の雛形になるため、テンプレート自体の配置は不要
  const [detachedHouse, semiHouse] = await Promise.all([buildHouse(engine, 1), buildHouse(engine, 2)]);

  // 各エントリは [家のタイプ, Y回転, x, z]
  const places: Place[] = [
    [1, -Math.PI / 16, -6.8, 2.5],
    [2, -Math.PI / 16, -4.5, 3],
    [2, -Math.PI / 16, -1.5, 4],
    [2, -Math.PI / 3, 1.5, 6],
    [2, (15 * Math.PI) / 16, -6.4, -1.5],
    [1, (15 * Math.PI) / 16, -4.1, -1],
    [2, (15 * Math.PI) / 16, -2.1, -0.5],
    [1, (5 * Math.PI) / 4, 0, -1],
    [1, Math.PI + Math.PI / 2.5, 0.5, -3],
    [2, Math.PI + Math.PI / 2.1, 0.75, -5],
    [1, Math.PI + Math.PI / 2.25, 0.75, -7],
    [2, Math.PI / 1.9, 4.75, -1],
    [1, Math.PI / 1.95, 4.5, -3],
    [2, Math.PI / 1.9, 4.75, -5],
    [1, Math.PI / 1.9, 4.75, -7],
    [2, -Math.PI / 3, 5.25, 2],
    [1, -Math.PI / 3, 6, 4],
  ];

  // 本家 createInstance 相当: 家タイプ別に thin instance プールを作成
  // ※ setParent 済み・registerScene 前、というタイミング制約に注意
  const detachedCount = places.filter((p) => p[0] === 1).length;
  const detachedPool = createHierarchyInstancePool(detachedHouse, detachedCount);
  const semiPool = createHierarchyInstancePool(semiHouse, places.length - detachedCount);

  for (const [type, rotY, x, z] of places) {
    addHierarchyInstance(type === 1 ? detachedPool : semiPool, placeMatrix(rotY, x, z));
  }

  addToScene(scene, ground);
  // thin instance の「器」としてテンプレート階層をシーンへ（children の box / roof も再帰追加される）
  addToScene(scene, detachedHouse);
  addToScene(scene, semiHouse);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/10?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-09 メッシュをコピー / 村を配置する"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/10
>
> 本家の `createInstance` は 17 個の `InstancedMesh` を作りますが、Lite ではタイプごとに 1 プール
> （＝box / roof それぞれ 1 本の thin instance バッファ）へ集約されるため、ドローコールはさらに少なくなります。

---

← [2-08 メッシュを結合 (Combining Meshes)](./2-08-combining-meshes.md) ・ [2-10 Viewer のカメラ変更](./2-10-viewer-camera.md) →
