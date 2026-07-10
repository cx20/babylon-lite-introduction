# 4-01 車の衝突事故を回避する (Avoiding a Car Crash) — △

> [第4部：衝突回避](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：歩行者（キャラ）が横断ラインを行き来し、車が横断ゾーン（`hitBox`）を通過中は歩行を止めて待つ。交差判定でぶつからないようにする。

キャラは 3-06 / 3-07 と同じ理由で `Dude.babylon` → **`Xbot.glb`** に置換します。

**Lite 移植時の注意点**：

- **`StandardMaterial.wireframe` が無い** — Lite は line-list トポロジ非対応です。`hitBox` の 12 辺を**極細円柱**で描く枠線（`createWireBox`）で代用し、`disableLighting` のフラット白でワイヤーフレームの見た目に寄せます（3-07 の線分と同じ手口）。
- **`mesh.intersectsMesh` が無い** — ワールド AABB 同士の重なり判定（`overlapAABB`）を自前実装します。本家 `intersectsMesh` の既定（バウンディングボックス判定）に相当します。
- **glTF ルートの x 軸反転に注意** — Lite の `loadGltf` は RH→LH 変換のため glTF ルート（`__root__`）に **x 軸スケール `-1`** を入れます。したがって glTF 配下のノード（車・Xbot）は **`world x = -(local x)`**。一方、枠線の箱はシーン直下に置くのでワールド座標そのままです。**片方だけ反転を揃えないと、箱は右・人と車は左** のようにズレます。

**座標の整列**：本家の値（車 `x = -3`、箱 `x = 3.1`、歩行者 `x = 1.5`）はこの反転のせいでかみ合わず、歩行者が待ちません。そこで 3 者を右の大通り（`world x = 3.1, z = -5`）へ集約します。

| 対象 | 置く座標系 | x |
|---|---|---|
| 箱（枠線の描画） | world（シーン直下） | `3.1` |
| 箱（衝突判定） | local（glTF と比較するため符号反転） | `-3.1` |
| 車 | local | `-3.1`（= world `3.1`） |
| 歩行者 | local | `-2.5` から箱（local `-3.1`）へ横断 |

衝突判定は **actor の local 位置と local の箱**で統一して比較します。

`movePOV` 相当（正面 = `-Z`）、Xbot の bake クォータニオン合成・`FRONT_OFFSET`・身長 1/3、コンテナ丸ごと `addToScene` による自動 tick、車輪回転の自作は、いずれも [第3部](../03-animation/README.md) の 3-05 / 3-07 と同じです。

追加 import：`createCylinder, createStandardMaterial, loadGltf, playAnimation, stopAnimation, onBeforeRender`（＋型 `AssetContainer, EngineContext, SceneContext, SceneNode, StandardMaterialProps`）

```typescript
type Vec3 = readonly [number, number, number];
interface AABB { readonly min: Vec3; readonly max: Vec3 }

function aabbFromCenter(cx: number, cy: number, cz: number, hx: number, hy: number, hz: number): AABB {
  return { min: [cx - hx, cy - hy, cz - hz], max: [cx + hx, cy + hy, cz + hz] };
}

/** AABB 同士の重なり判定（本家 intersectsMesh の既定相当） */
function overlapAABB(a: AABB, b: AABB): boolean {
  return a.min[0] <= b.max[0] && a.max[0] >= b.min[0]
      && a.min[1] <= b.max[1] && a.max[1] >= b.min[1]
      && a.min[2] <= b.max[2] && a.max[2] >= b.min[2];
}

/** 軸平行ボックスの 12 辺を極細円柱で描く（wireframe 代用） */
function createWireBox(scene: SceneContext, engine: EngineContext, center: Vec3, half: Vec3,
                       mat: StandardMaterialProps): void {
  const [cx, cy, cz] = center;
  const [hx, hy, hz] = half;
  const c: Vec3[] = [
    [cx - hx, cy - hy, cz - hz], [cx + hx, cy - hy, cz - hz], [cx + hx, cy - hy, cz + hz], [cx - hx, cy - hy, cz + hz],
    [cx - hx, cy + hy, cz - hz], [cx + hx, cy + hy, cz - hz], [cx + hx, cy + hy, cz + hz], [cx - hx, cy + hy, cz + hz],
  ];
  const edges: [number, number][] = [
    [0, 1], [1, 2], [2, 3], [3, 0],   // 下面
    [4, 5], [5, 6], [6, 7], [7, 4],   // 上面
    [0, 4], [1, 5], [2, 6], [3, 7],   // 垂直
  ];
  for (const [i, j] of edges) {
    const a = c[i]!, b = c[j]!;
    const dir: Vec3 = [b[0] - a[0], b[1] - a[1], b[2] - a[2]];
    const len = Math.hypot(dir[0], dir[1], dir[2]);
    const seg = createCylinder(engine, { diameter: 0.03, height: len, tessellation: 5 });
    seg.material = mat;
    seg.position.set((a[0] + b[0]) / 2, (a[1] + b[1]) / 2, (a[2] + b[2]) / 2);
    const q = quatFromYTo(dir);                 // +Y を dir へ向ける最短弧（3-01 の orientYAxisTo と同じ）
    seg.rotationQuaternion.set(q.x, q.y, q.z, q.w);
    addToScene(scene, seg);
  }
}

// 車の走行ライン（glTF ローカル x）。world x = -(local x) なので、
// 右の大通り world x = 3.1 に出すには local x = -3.1 にする。
const CAR_ROAD_X = -3.1;

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2.2, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  // 横断ゾーン hitBox（本家 width 0.5 / height 0.6 / depth 4.5）
  // 枠線はシーン直下なので world 座標
  const HIT_WORLD_CENTER: Vec3 = [3.1, 0.3, -5];
  const HIT_HALF: Vec3 = [0.25, 0.3, 2.25];
  createWireBox(scene, engine, HIT_WORLD_CENTER, HIT_HALF, createFlatWhite());

  // 衝突判定は glTF 配下の actor（local 座標・x 軸 -1）と比べるので、箱も local で持つ
  const hitAABB = aabbFromCenter(-HIT_WORLD_CENTER[0], HIT_WORLD_CENTER[1], HIT_WORLD_CENTER[2], ...HIT_HALF);

  const [villageContainer, carContainer, xbotContainer] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/village.glb"),
    loadGltf(engine, "https://assets.babylonjs.com/meshes/car.glb"),
    loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb"),
  ]);
  addToScene(scene, villageContainer);
  addToScene(scene, carContainer);
  addToScene(scene, xbotContainer);   // コンテナ丸ごと → animationGroups が自動 tick される

  // ---- 車 ----
  const car = findNodeByName(carContainer, "car");
  car.rotation.set(Math.PI / 2, 0, -Math.PI / 2);
  car.position.set(CAR_ROAD_X, 0.16, 8);
  const wheels = ["wheelRB", "wheelRF", "wheelLB", "wheelLF"].map((n) => findNodeByName(carContainer, n));

  // ---- 歩行者 ----
  playOnly(xbotContainer, "walk");
  const xbot = findNodeByName(xbotContainer, "Xbot");
  const q0 = xbot.rotationQuaternion;
  const bake = { x: q0.x, y: q0.y, z: q0.z, w: q0.w };

  const SIZE = 1 / 3;
  const sc = xbot.scaling;
  sc.set(sc.x * SIZE, sc.y * SIZE, sc.z * SIZE);

  // local x = -2.5（= world 2.5）から、箱（local x = -3.1）へ向かって横断する
  const START: Vec3 = [-2.5, 0, -5];
  const START_YAW = (90 * Math.PI) / 180;
  const FRONT_OFFSET = Math.PI;                 // メッシュ正面のズレ吸収（見た目のみ）
  const applyHeading = (yaw: number): void => {
    const q = qmul(yawQuat(yaw + FRONT_OFFSET), bake);
    xbot.rotationQuaternion.set(q.x, q.y, q.z, q.w);
  };

  const track = [
    { turn: 180, dist: 2.5 },
    { turn: 0, dist: 5 },
  ] as const;

  let distance = 0;
  let yaw = START_YAW;
  let p = 0;
  xbot.position.set(START[0], START[1], START[2]);
  applyHeading(yaw);

  const WALK_SPEED = 0.015 * 60;                // 本家 step = 0.015 @60fps
  const WHEEL_RATE = -2 * Math.PI;              // rad/秒（本家 0 → -2π を 30 frame @30fps）
  const CAR_PERIOD = 200 / 30;                  // 秒（frame 200 @30fps）
  let elapsed = 0;

  // 本家の車キー（0 → z=8、150 → z=-7、200 → z=-7、ループ）を時間から解析的に求める
  const carZAt = (t: number): number => {
    const tt = t % CAR_PERIOD;
    const tMove = 150 / 30;                     // 5 秒で 8 → -7
    return tt <= tMove ? 8 + (-7 - 8) * (tt / tMove) : -7;
  };

  onBeforeRender(scene, (deltaMs: number) => {
    const dt = deltaMs / 1000;
    elapsed += dt;

    car.position.z = carZAt(elapsed);
    for (const w of wheels) w.rotation.y += WHEEL_RATE * dt;   // car.glb に車輪アニメは無い

    // 交差判定はすべて glTF ローカル座標で統一する
    const carAABB = aabbFromCenter(car.position.x, 0.16, car.position.z, 0.4, 0.2, 0.5);
    const walkerAABB = aabbFromCenter(xbot.position.x, 0.3, xbot.position.z, 0.2, 0.3, 0.2);

    // 本家の待機条件：歩行者が箱の外 かつ 車が箱の中 → 歩かない
    const dudeInBox = overlapAABB(walkerAABB, hitAABB);
    const carInBox = overlapAABB(carAABB, hitAABB);
    if (!dudeInBox && carInBox) return;         // 待機（walk アニメは自動 tick で継続＝足踏み）

    // 前進（movePOV(0, 0, step) 相当・正面 = -Z）
    const step = WALK_SPEED * dt;
    xbot.position.x += -Math.sin(yaw) * step;
    xbot.position.z += -Math.cos(yaw) * step;
    distance += step;

    if (distance > track[p]!.dist) {
      yaw += (track[p]!.turn * Math.PI) / 180;  // rotate(Axis.Y, toRad(turn), LOCAL) 相当
      applyHeading(yaw);
      p = (p + 1) % track.length;
      if (p === 0) {
        distance = 0;
        yaw = START_YAW;
        xbot.position.set(START[0], START[1], START[2]);
        applyHeading(yaw);
      }
    }
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/UJX6YO/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 4-01 車の衝突事故を回避する"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/UJX6YO/v/0
>
> **省略した補助関数**（完全版は Playground 参照）：
> - `quatFromYTo(dir)` — `+Y` を `dir` へ向ける最短弧クォータニオン（3-01 の `orientYAxisTo` と同じ）
> - `createFlatWhite()` — `disableLighting` のフラット白マテリアル（3-07 の `createLineMaterial` と同じ）
> - `qmul` / `yawQuat` / `findNodeByName` / `playOnly` — 3-05・3-07 と同じ
>
> はまりどころは **glTF ルートの x 軸反転**です。枠線の箱だけワールド座標、車と歩行者は glTF ローカル座標にいるため、
> 判定と描画で座標系を混ぜると「箱は右、人と車は左」になります。**判定は local に揃える**のが安全です。
>
> 待機中も `walk` アニメは自動 tick で回り続けるため、その場で足踏みします。本家と同じ挙動です。

## より簡単な判定

厳密な箱判定が要らないなら、距離（円）判定で十分なことも多いです。

```typescript
const near = (a: {x: number; z: number}, b: {x: number; z: number}, r: number) =>
  Math.hypot(a.x - b.x, a.z - b.z) < r;

const obstacles = [{ x: 5, z: 0 }, { x: -3, z: 4 }];

onBeforeRender(scene, () => {
  const willHit = obstacles.some((o) => near(root.position, o, 1.5));
  if (!willHit) {
    root.position.x += Math.sin(heading) * speed;
    root.position.z += Math.cos(heading) * speed;
  } else {
    heading += 0.05;   // 迂回：向きを少し変える
    applyPose();
  }
});
```

読み込んだメッシュは `boundMin` / `boundMax`（ワールド AABB）を持つので、上の `overlapAABB` をそれに対して直接適用することもできます。ただし本サンプルでは、glTF ローカル座標のノード位置と比較したいため、AABB を中心と半径から解析的に構築しています。

> より厳密な当たり判定が必要なら Ray Casting / Picking（GPU ID パス＋ CPU レイ/三角形、thin-instance・変形メッシュ対応）が使えます。

---

← [4-00 コリジョン回避 (Avoiding Collisions)](./4-00-avoiding-collisions.md) ・ [5-00 より良い環境に (A Better Environment)](../05-environment/5-00-better-environment.md) →
