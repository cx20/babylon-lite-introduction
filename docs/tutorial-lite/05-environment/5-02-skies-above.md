# 5-02 頭上の空 (Skies Above) — ○

> [第5部：環境改善](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：完成した村と走る車に、キューブテクスチャのスカイボックスを追加する。

**Lite 移植時の注意点**：

- **`loadSkybox` ヘルパーがある** — 本家は `CreateBox` ＋ `StandardMaterial` ＋ `CubeTexture`（`SKYBOX_MODE`）＋ `backFaceCulling = false` を手で組みますが、
  Lite は **`loadSkybox(scene, baseUrl, ext, size)` 一発**で箱・キューブテクスチャ・`SKYBOX_MODE`・内面描画・シーン登録まで済みます（async）。
  6 面は `baseUrl` に `_px` / `_nx` / `_py` / `_ny` / `_pz` / `_nz` ＋ `ext` を付けて解決されます。`size` の既定は `100`（本家と同じ）。
- **画像は絶対 URL で** — 本家の `"textures/skybox"` は Playground 相対パスなので Lite では解決できません。
- **`camera.upperBetaLimit` はある** — Lite の `ArcRotateCamera` にもフィールドがあり、そのまま代入できます（地面より下へ回り込まないための制限）。

追加 import：`loadSkybox`

```typescript
const VALLEY_VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const CAR_URL = "https://assets.babylonjs.com/meshes/car.glb";
const SKYBOX_BASE_URL = "https://playground.babylonjs.com/textures/skybox";
const SKYBOX_EXT = ".jpg";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  camera.upperBetaLimit = Math.PI / 2.2;   // 地面より下へ回り込まない
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  // 村・車・スカイボックスを並列に用意（registerScene の前に await 完了）
  const [village, carContainer] = await Promise.all([
    loadGltf(engine, VALLEY_VILLAGE_URL),
    loadGltf(engine, CAR_URL),
    loadSkybox(scene, SKYBOX_BASE_URL, SKYBOX_EXT, 150),   // 箱〜シーン登録まで完結
  ]);
  addToScene(scene, village);
  addToScene(scene, carContainer);

  // 車の設定・車輪回転・Z 移動は 5-01「車を追加」と同じ
  const car = findNodeByName(carContainer, "car");
  car.rotation.set(Math.PI / 2, 0, -Math.PI / 2);
  car.position.set(-3, 0.16, 8);
  const wheels = ["wheelRB", "wheelRF", "wheelLB", "wheelLF"].map((n) => findNodeByName(carContainer, n));

  const CAR_PERIOD = 200 / 30;
  const Z_START = 10;
  const Z_END = -15;
  const carZAt = (t: number): number => Z_START + (Z_END - Z_START) * ((t % CAR_PERIOD) / CAR_PERIOD);

  const WHEEL_RATE = -2 * Math.PI;
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

<iframe src="https://liteplayground.babylonjs.com/snippet/DQXJD5/v/6?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 5-02 頭上の空"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DQXJD5/v/6
>
> `loadSkybox` は `scene` を受け取り、内部でシーンへ登録するところまで行うので、戻り値を `addToScene` する必要はありません。
> `findNodeByName` は 3-05 / 3-07 / 4-01 と同じものです。
>
> **`upperBetaLimit` は `setCameraLimits` 経由が本来の作法**です。直接代入でも `attachControl` の毎フレーム処理で
> 制限は効きますが、`setCameraLimits(camera, { upperBetaLimit })` を使うと現在の姿勢が即座にクランプされ、
> 慣性による行き過ぎ→スナップのブレを避けられます。

## 環境光まで欲しい場合（loadEnvironment）

PBR マテリアルを正しく光らせたいなら、スカイボックスと IBL 環境光をまとめて設定する `loadEnvironment` を使います。

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
- 背景として空を出すだけなら、上の **`loadSkybox`（6 枚 JPG のキューブ）が最も手軽**です。

---

← [5-01 遠くの丘 (Distant Hills)](./5-01-distant-hills.md) ・ [5-03 木のスプライト (Sprite Trees)](./5-03-sprite-trees.md) →
