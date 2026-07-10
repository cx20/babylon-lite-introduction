# 6-01 旋盤で回された噴水 (A Lathe Turned Fountain) — △

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

本章は、水を噴き上げる**器の回転体**（`CreateLathe` 相当）と、**噴水の水（パーティクル）**の組み合わせです。
Lite では、器は `createRibbon` で忠実に再現でき、水は [6-00](./6-00-particle-fountain.md) の **NPE パーティクル**で表現します。
ここではまず器を作って村に置くところまでを示し、水は 6-00 の NPE パーティクルを器の口の位置に重ねます。判定は △ です。

**Lite 移植時の注意点**：

- **`CreateLathe` は無い** — ただし本家の `CreateLathe` は内部で「輪郭を回転角ごとに複製した複数パスを `CreateRibbon` で張る」実装です。
  Lite には **`createRibbon`（`pathArray` を面で張る）があるので忠実に再現**できます。輪郭を Y 軸まわりに `tessellation` 分割して各断面パスを作り、
  `createRibbon(engine, { pathArray, closeArray: true })` で張ります（`closeArray: true` で一周を閉じて筒状に。法線は内部で自動計算されます）。
- **`sideOrientation: DOUBLESIDE` の相当は `material.backFaceCulling = false`** — Lite の `createRibbon` は `sideOrientation` を取りません。
  両面描画（回転体の内側も見える）にはマテリアル側で `backFaceCulling = false` にします。

追加 import：`createRibbon, createStandardMaterial`（＋型 `Mesh`）

```typescript
type Vec3 = { x: number; y: number; z: number };

// 本家 MeshBuilder.CreateLathe 相当。輪郭を Y 軸まわりに tessellation 分割で回し、createRibbon で張る
function createLathe(engine: EngineContext, shape: readonly Vec3[], tessellation = 24): Mesh {
  const pathArray: Vec3[][] = [];
  for (let s = 0; s <= tessellation; s++) {
    const ang = (2 * Math.PI * s) / tessellation;
    const c = Math.cos(ang);
    const si = Math.sin(ang);
    const path = shape.map((p) => ({ x: p.x * c - p.z * si, y: p.y, z: p.x * si + p.z * c }));
    pathArray.push(path);
  }
  return createRibbon(engine, { pathArray, closeArray: true });   // closeArray: true で筒状に閉じる
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 1.5, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
  camera.upperBetaLimit = Math.PI / 2.2;
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  await loadSkybox(scene, "https://playground.babylonjs.com/textures/skybox", ".jpg", 150);

  // 噴水の輪郭（本家と同一の 8 点）
  const fountainOutline: Vec3[] = [
    { x: 0, y: 0, z: 0 },
    { x: 0.5, y: 0, z: 0 },
    { x: 0.5, y: 0.2, z: 0 },
    { x: 0.4, y: 0.2, z: 0 },
    { x: 0.4, y: 0.05, z: 0 },
    { x: 0.05, y: 0.1, z: 0 },
    { x: 0.05, y: 0.8, z: 0 },
    { x: 0.15, y: 0.9, z: 0 },
  ];

  const fountain = createLathe(engine, fountainOutline, 24);
  const fountainMat = createStandardMaterial();
  fountainMat.backFaceCulling = false;   // DOUBLESIDE 相当。回転体の内側も見える
  fountain.material = fountainMat;
  fountain.position.x = -4;
  fountain.position.z = -6;
  addToScene(scene, fountain);

  const village = await loadGltf(engine, "https://assets.babylonjs.com/meshes/valleyvillage.glb");
  addToScene(scene, village);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/9?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 6-01 旋盤で回された噴水（器の形状）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/9
>
> ここで作れるのは**器まで**です。水は [6-00](./6-00-particle-fountain.md) の NPE パーティクルを、器の口（この輪郭では最上部あたり）に `emitter` 位置を合わせて重ねます。
> `createRibbon` による回転体は、噴水の器に限らず柱・壺・レンズ形状などにもそのまま応用できます。

---

← [6-00 パーティクル噴水 (Build a Particle Fountain)](./6-00-particle-fountain.md) ・ [7-00 光と影 (Light the Night)](../07-lights/7-00-light-intro.md) →
