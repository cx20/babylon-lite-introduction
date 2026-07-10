# ゴール完成版：村の道を歩く Xbot（三人称追従カメラ）

> 各章（[第1部](./01-basics/README.md)〜[第5部](./05-environment/README.md)）の集大成。
> [Babylon Lite Playground](https://liteplayground.babylonjs.com) にそのまま貼り付けて Run できます。

このシーンが含む章の要素：

| 要素 | 対応章 |
|---|---|
| エンジン/シーン/カメラ/ライト | 1-01 |
| 外部モデル読込（glTF） | 1-03 |
| スケルタル歩行アニメ（Xbot.glb） | 3-06 |
| ルート歩行＋三人称追従カメラ | 3-07 |
| 村（valleyvillage.glb、両面描画修正） | 2-04〜2-06 の発展 |
| スカイボックス相当の環境 | 5-02（任意） |
| スプライトの木（軸ロック billboard） | 5-03 |

## 既知の落とし穴と対処（このコードで対策済み）

| 症状 | 原因 | 対処 |
|---|---|---|
| 家が裏返って見える | 単面ポリゴンの壁が片面描画 | ロード後に `material.doubleSided = true` |
| 木が地面から下にぶら下がる | `pivot` の縦が上端基準 | `pivot: [0.5, 1]`（下端＝根元） |
| カメラの前後が逆 | glTF の RH→LH 変換で向きが 180° ズレ | `alpha = -Math.PI/2`（背後） |
| キャラが巨大 | Xbot は等身大、村スケールと差 | `root.scaling = 0.25` |
| `upperBetaLimit` が無い | Lite にフィールドなし | 毎フレーム `camera.beta` を clamp |
| `.babylon` の歩行アニメが出ない | `.babylon` はスキン/アニメ非対応 | glTF（`Xbot.glb`）に置換 |

## 完成コード

```typescript
import {
  createEngine, createSceneContext,
  createArcRotateCamera, attachControl,
  createHemisphericLight, createDirectionalLight,
  addToScene, onBeforeRender, registerScene, startEngine,
  loadGltf, playAnimation, stopAnimation,
  loadSpriteAtlas,
  createAxisLockedBillboardSystem, addBillboardSpriteIndex, addAxisLockedBillboardSystem,
} from "@babylonjs/lite";

async function main(): Promise<void> {
  const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement;
  const engine = await createEngine(canvas);
  const scene  = createSceneContext(engine);

  // --- ライト ---
  addToScene(scene, createHemisphericLight([1, 1, 0], 2.0));
  const sun = createDirectionalLight([-1, -2, -1]);
  sun.intensity = 1.5;
  addToScene(scene, sun);

  // --- キャラクター：Xbot.glb ---
  const xbot = await loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb");
  addToScene(scene, xbot);
  const root = xbot.entities[0];

  // 身長を 1/4 に
  const SCALE = 0.25;
  root.scaling.x = SCALE; root.scaling.y = SCALE; root.scaling.z = SCALE;

  // walk クリップだけ再生
  for (const g of xbot.animationGroups ?? []) stopAnimation(g);
  const walk = (xbot.animationGroups ?? []).find((g) => g.name === "walk")
            ?? xbot.animationGroups?.[0];
  if (walk) { walk.loopAnimation = true; walk.speedRatio = 1.0; playAnimation(walk); }

  // --- カメラ（キャラに parent して背後から追従）---
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 1, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  camera.parent = root;

  // --- 村（壁の裏抜け対策：両面描画へ）---
  const village = await loadGltf(engine, "https://assets.babylonjs.com/meshes/valleyvillage.glb");
  const forceDoubleSided = (node: any) => {
    if (node?.material && "doubleSided" in node.material) node.material.doubleSided = true;
    for (const child of node?.children ?? []) forceDoubleSided(child);
  };
  for (const e of village.entities) forceDoubleSided(e);
  addToScene(scene, village);

  // --- 木（軸ロック billboard、pivot 下端で接地）---
  const palm = await loadSpriteAtlas(engine, "https://playground.babylonjs.com/textures/palm.png", {
    gridSize: [1, 1],
  });
  const trees = createAxisLockedBillboardSystem(palm, [0, 1, 0], { capacity: 1000 });
  const addTree = (x: number, z: number) =>
    addBillboardSpriteIndex(trees, { position: [x, 0, z], sizeWorld: [2, 4], pivot: [0.5, 1] });
  for (let i = 0; i < 200; i++) addTree(Math.random() * -40 - 10, Math.random() * 60 - 30);
  for (let i = 0; i < 200; i++) addTree(Math.random() *  40 + 10, Math.random() * 60 - 30);
  addAxisLockedBillboardSystem(scene, trees);

  // ============================================================
  // 歩行ルート：道路に沿ったウェイポイント（world 座標 x, z）
  // ★ valleyvillage の道路上の点に書き換えると、その道を歩きます。
  // ============================================================
  const waypoints = [
    { x: -25, z: -2 },
    { x:  25, z: -2 },
  ];
  let wp = 1;
  const speed = 0.03;
  const MODEL_YAW_OFFSET = 0;                  // 後ろ向きに歩くなら Math.PI
  let heading = 0;

  const applyPose = () => {                    // Y 軸ヨーのクォータニオン（成分で書き込む）
    const h = (heading + MODEL_YAW_OFFSET) * 0.5;
    root.rotationQuaternion.x = 0;
    root.rotationQuaternion.y = Math.sin(h);
    root.rotationQuaternion.z = 0;
    root.rotationQuaternion.w = Math.cos(h);
  };

  root.position.x = waypoints[0].x;
  root.position.y = 0;
  root.position.z = waypoints[0].z;

  onBeforeRender(scene, () => {
    // 現在のウェイポイントへ前進（movePOV 相当）
    const t = waypoints[wp];
    let dx = t.x - root.position.x;
    let dz = t.z - root.position.z;
    const d = Math.hypot(dx, dz);
    if (d < 0.3) {
      wp = (wp + 1) % waypoints.length;
    } else {
      dx /= d; dz /= d;
      heading = Math.atan2(dx, dz);            // forward = (sin h, 0, cos h)
      applyPose();                             // キャラの向き変化に parent したカメラも追従
      root.position.x += dx * speed;
      root.position.z += dz * speed;
    }

    // upperBetaLimit = π/2.2 の代替
    if (camera.beta > Math.PI / 2.2) camera.beta = Math.PI / 2.2;
  });

  await registerScene(scene);
  await startEngine(engine);
}

main().catch((err) => console.error(err));
```

## 調整ポイント

- **道路合わせ**：`waypoints` の `(x, z)` を実際の道路上の点に置き換える。曲がり道は点を増やす。
- **明るさ**：`createHemisphericLight([1,1,0], 2.0)` の `2.0`、`sun.intensity`。
- **身長**：`SCALE`。カメラの寄りは半径 `3` と注視 `y: 1`。
- **向き**：後ろ向きに歩くなら `MODEL_YAW_OFFSET = Math.PI`。
- **任意の空**：[5-02](./05-environment/5-02-skies-above.md) の `loadEnvironment`（`brdfUrl` 必須）を `registerScene` の前に追加。
