# 3-07 村を歩き回る (A Walk Around The Village) — △

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：まず球を三角形の軌道に沿って走らせて「前進＋その場で回頭」の組み合わせを掴み、
そのうえでキャラをルートに沿って歩かせ、カメラで追う。**これがゴールの中核**です。

**Lite 移植時の注意点**：

- **`mesh.movePOV` / `mesh.rotate` が無い** — Lite の `Mesh` には視点基準の移動・ローカル軸回転のヘルパーがありません。
  本章の軌道は XZ 平面・ヨーのみなので、**進行方向のヨー角スカラー `yaw` を自前で保持**し、
  `position += (-sin yaw, 0, -cos yaw) * step` で `movePOV(0, 0, step)` を、`yaw += turn` で `rotate(Axis.Y, turn, LOCAL)` を置き換えます。
  本家 `movePOV` は `definedFacingForward = true` が既定なので **yaw = 0 のときの前方は `-Z`**。Lite の `+Z` 前提から符号を反転して規約を合わせます。
- **`CreateLines`（線分描画）が無い** — 3-01 と同じく、**細い `createCylinder` ＋無照明マテリアル**で線分を代用します。
- **`FollowCamera` が無い** — 後半のキャラ追従は `ArcRotateCamera` を `camera.parent = root` で親子付けして代替します。

## 移動と回転 (Combining Movements)

**例）三角形の辺に沿って移動する球** — 直線で `dist` だけ進んだら `turn` だけ回頭する、を繰り返します。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createSphere, createCylinder, createStandardMaterial, onBeforeRender`（＋型 `EngineContext, Mesh, SceneContext, StandardMaterialProps`）

```typescript
type Vec3Tuple = readonly [number, number, number];

/** CreateLines 相当のフラット白線マテリアル（ライト非依存） */
function createLineMaterial(): StandardMaterialProps {
  const mat = createStandardMaterial();
  mat.diffuseColor = [1, 1, 1];
  mat.emissiveColor = [1, 1, 1];  // 出力 = emissive * diffuse * baseColor = 白
  mat.disableLighting = true;     // 陰影計算をスキップしてフラットに
  return mat;
}

/** 2 点間に極細円柱を 1 本置く（線分の代用・全辺水平前提） */
function addSegment(scene: SceneContext, engine: EngineContext,
                    a: Vec3Tuple, b: Vec3Tuple, mat: StandardMaterialProps): void {
  const dx = b[0] - a[0];
  const dz = b[2] - a[2];
  const len = Math.hypot(dx, b[1] - a[1], dz);

  const seg = createCylinder(engine, { diameter: 0.03, height: len, tessellation: 6 });
  seg.material = mat;
  seg.position.set((a[0] + b[0]) / 2, (a[1] + b[1]) / 2, (a[2] + b[2]) / 2);

  // +Y を水平方向 (dx, 0, dz) へ向ける。Lite の Euler は XYZ 順なので
  // rz = π/2 で +Y を水平化 → ry で方位。rz 適用後の +Y は (-cos ry, 0, sin ry)
  seg.rotation.set(0, Math.atan2(dz / len, -dx / len), Math.PI / 2);
  addToScene(scene, seg);
}

/** points を順に線分でつないだ「線メッシュ」相当 */
function createLinePath(scene: SceneContext, engine: EngineContext, points: readonly Vec3Tuple[]): void {
  const mat = createLineMaterial();
  for (let i = 0; i < points.length - 1; i++) {
    addSegment(scene, engine, points[i]!, points[i + 1]!, mat);
  }
}

/** 本家 track と同じ「dist まで進んだら turn だけ回る」 */
interface Slide {
  readonly turn: number;
  readonly dist: number;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 1.5, Math.PI / 5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const START: Vec3Tuple = [2, 0, 2];
  const sphere: Mesh = createSphere(engine, { diameter: 0.25, segments: 16 });
  sphere.material = createStandardMaterial();     // ★マテリアル必須
  sphere.position.set(START[0], START[1], START[2]);
  addToScene(scene, sphere);

  // 三角形の軌道（本家 CreateLines 相当のフラット白線）
  createLinePath(scene, engine, [
    [2, 0, 2],
    [2, 0, -2],
    [-2, 0, -2],
    [2, 0, 2],   // 三角形を閉じる
  ]);

  const track: readonly Slide[] = [
    { turn: Math.PI / 2, dist: 4 },
    { turn: (3 * Math.PI) / 4, dist: 8 },
    { turn: (3 * Math.PI) / 4, dist: 8 + 4 * Math.sqrt(2) },
  ];

  let distance = 0;
  let yaw = 0;
  let p = 0;
  const SPEED = 0.05 * 30;   // 本家 step = 0.05 @30fps を deltaMs 非依存化

  onBeforeRender(scene, (deltaMs: number) => {
    const step = SPEED * (deltaMs / 1000);

    // movePOV(0, 0, step) 相当。yaw = 0 のときの前方は -Z
    sphere.position.x += -Math.sin(yaw) * step;
    sphere.position.z += -Math.cos(yaw) * step;
    distance += step;

    if (distance > track[p]!.dist) {
      yaw += track[p]!.turn;        // rotate(Axis.Y, turn, LOCAL) 相当
      sphere.rotation.y = yaw;
      p = (p + 1) % track.length;
      if (p === 0) {                // 一周したら初期状態へ戻す
        distance = 0;
        yaw = 0;
        sphere.position.set(START[0], START[1], START[2]);
        sphere.rotation.set(0, 0, 0);
      }
    }
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-07 村を歩き回る / 移動と回転"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/0
>
> ポイントは 2 つです。
> 1. **`movePOV` / `rotate` の代替**は、姿勢をクォータニオンで持たず **ヨー角スカラー `yaw` を単一の真実**として扱い、そこから位置更新と `rotation.y` の両方を導くこと（XZ 平面の軌道なのでこれで十分）
> 2. **線の質感**は `disableLighting = true` が要。Lite の Standard シェーダは無照明時に `clamp(emissiveColor * diffuseColor, 0, 1) * baseColor` を出力するため、両方を白にすればライト方向に依存しないフラットな白＝`CreateLines` と同じ見た目になる。円柱を十分細く（直径 0.03）すれば極細ハードエッジにも近づく
>
> 本家は毎フレーム固定の `step = 0.05` ですが、`onBeforeRender` の `deltaMs` を使って速度をフレームレート非依存にしています（30fps でも 120fps でも同じ速さ）。

## 村を歩かせる

`village.glb` の中を、キャラが `walk` アニメーションで巡回します。
本家は `Dude.babylon` を使いますが、3-06 と同じ理由（**Lite の `.babylon` ローダーはスキニング/アニメ非対応**）で
**`Xbot.glb` に置換**します。Xbot は等身大＋内部スケール 0.01＋直立用の X+90° 回転がルートノードに
**bake（焼き込み）済み**なので、本家の `scaling = 0.008` のような指定は不要です。

**Lite で押さえる 4 点**：

1. **向きは「bake クォータニオンへの合成」で与える** — Xbot ルートの `rotationQuaternion` には直立用の X+90° が入っています。
   ここに `rotation.y = yaw` を代入すると Euler プロキシが quaternion を**純 Y 回転で上書き**し、直立が消えてキャラが倒れます。
   起動時に bake を保存し、毎回 `rotationQuaternion = yaw(worldY) ⊗ bake` を合成して代入してください。
   world-Y は必ず鉛直なので、直立を保ったまま heading だけが変わります。`scaling` / `position` は `rotation` と独立なので、
   heading を毎フレーム上書きしても身長調整と併用できます。
2. **glTF 内蔵アニメは「コンテナ丸ごと」を `addToScene` する** — Lite は `addToScene` に**コンテナ全体**を渡したときだけ、
   その `animationGroups` を毎フレーム自動 tick します（`_beforeRender` に登録）。`container.entities` を個別に追加すると
   自動 tick が働かず、**スキンが bind（T）ポーズのまま止まります**。
   さらにロード直後は先頭クリップ（Xbot では `idle`）だけが再生状態なので、`stopAnimation` で全停止 → `playAnimation(walk)` へ切り替えます。
3. **移動は前節と同じ自前 `movePOV` / `rotate`** — 正面 = `-Z` 規約で `forward = (-sin yaw, 0, -cos yaw)`。
   本家 `track` の `turn` は**度数**なので `Math.PI / 180` でラジアン化します。
4. **メッシュ正面と移動方向のズレは「見た目専用オフセット」で吸収** — Xbot のメッシュ正面は `-Z` 規約と 180° ずれており、
   そのままだと進行方向と逆を向きます（＝ムーンウォーク）。移動方向自体は正しいので、**見た目の heading にだけ**
   `FRONT_OFFSET = π` を足します。移動に使う `yaw` に足すと軌道ごと反転してしまいます。

追加 import：`loadGltf, playAnimation, stopAnimation, onBeforeRender`（＋型 `AssetContainer, EngineContext, SceneContext, SceneNode`）

```typescript
interface Quat { x: number; y: number; z: number; w: number }

/** クォータニオン積 a⊗b（(a⊗b)·v = a(b(v)) の規約） */
function qmul(a: Quat, b: Quat): Quat {
  return {
    x: a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y,
    y: a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x,
    z: a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w,
    w: a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z,
  };
}

/** world-Y 軸まわり yaw のクォータニオン */
function yawQuat(yaw: number): Quat {
  return { x: 0, y: Math.sin(yaw / 2), z: 0, w: Math.cos(yaw / 2) };
}

/** 指定名のアニメーションだけを再生（他は停止） */
function playOnly(container: AssetContainer, name: string): void {
  const groups = container.animationGroups ?? [];
  let matched = false;
  for (const g of groups) {
    if (g.name === name) {
      playAnimation(g);
      g.loopAnimation = true;
      g.speedRatio = 1;
      matched = true;
    } else {
      stopAnimation(g);   // ロード直後は先頭クリップ（idle）が再生状態
    }
  }
  if (!matched) throw new Error(`アニメーション "${name}" が見つかりません。利用可能: ${groups.map((g) => g.name).join(", ")}`);
}

interface Walk {
  readonly turn: number;   // degrees
  readonly dist: number;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 1.5, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [villageContainer, xbotContainer] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/village.glb"),
    loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb"),
  ]);

  // ★ コンテナ丸ごと追加（これで animationGroups の自動 tick が有効になる）
  addToScene(scene, villageContainer);
  addToScene(scene, xbotContainer);

  playOnly(xbotContainer, "walk");   // idle を止めて walk を再生

  // Xbot ルート（直立 bake ＋ 内部スケール 0.01）。findNodeByName は 3-05 と同じもの
  const xbot = findNodeByName(xbotContainer, "Xbot");

  // 直立 bake を保存 → この後 heading を毎回これに合成する
  const q0 = xbot.rotationQuaternion;
  const bake: Quat = { x: q0.x, y: q0.y, z: q0.z, w: q0.w };

  // 身長調整：焼き込み済みの現在値へ乗算するので、内部スケールの実値に依存せず 1/3 になる
  const SIZE = 1 / 3;
  const s = xbot.scaling;
  s.set(s.x * SIZE, s.y * SIZE, s.z * SIZE);

  // 本家の歩行トラック（turn は度数）
  const track: readonly Walk[] = [
    { turn: 86, dist: 7 },
    { turn: -85, dist: 14.8 },
    { turn: -93, dist: 16.5 },
    { turn: 48, dist: 25.5 },
    { turn: -112, dist: 30.5 },
    { turn: -72, dist: 33.2 },
    { turn: 42, dist: 37.5 },
    { turn: -98, dist: 45.2 },
    { turn: 0, dist: 47 },
  ];

  const START: readonly [number, number, number] = [-6, 0, 0];
  const START_YAW = (-95 * Math.PI) / 180;   // 本家 rotate(Y, -95°) 相当の初期向き
  const FRONT_OFFSET = Math.PI;              // ★見た目専用。yaw 側に足すと軌道が壊れる

  let distance = 0;
  let yaw = START_YAW;
  let p = 0;
  const SPEED = 0.015 * 60;                  // 本家 step = 0.015 @60fps

  const applyHeading = (): void => {
    const q = qmul(yawQuat(yaw + FRONT_OFFSET), bake);   // 直立 bake を保ったまま heading だけ変える
    xbot.rotationQuaternion.set(q.x, q.y, q.z, q.w);
  };

  xbot.position.set(START[0], START[1], START[2]);
  applyHeading();

  onBeforeRender(scene, () => {
    // walk の tick はコンテナ追加時に自動登録済み。ここでは移動のみ駆動する
    const step = SPEED * (1 / 60);

    // movePOV(0, 0, step) 相当（正面 = -Z）
    xbot.position.x += -Math.sin(yaw) * step;
    xbot.position.z += -Math.cos(yaw) * step;
    distance += step;

    if (distance > track[p]!.dist) {
      yaw += (track[p]!.turn * Math.PI) / 180;   // rotate(Axis.Y, toRad(turn), LOCAL) 相当
      applyHeading();
      p = (p + 1) % track.length;
      if (p === 0) {                             // 一周したら初期状態へ戻す
        distance = 0;
        yaw = START_YAW;
        xbot.position.set(START[0], START[1], START[2]);
        applyHeading();
      }
    }
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 村を歩き回る Xbot"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/DZDR5Q/v/1
>
> Xbot が持つクリップは `idle` / `agree` / `run` / `sad_pose` / `walk` / `sneak_pose` / `headShake` です。
> `stopAnimation` で全停止してから `walk` だけを再生しないと、`idle` が重なって足が動きません。
>
> 一番はまりやすいのは **(1) の bake 合成**と **(2) のコンテナ丸ごと追加**です。
> `xbot.rotation.y = yaw` と書くと Euler プロキシが quaternion を純 Y 回転で上書きするため、
> 直立用の X+90° が消えてキャラが床に倒れます。必ず `yaw(worldY) ⊗ bake` を合成してください。

## 三人称カメラで追う

キャラに**カメラを `parent`** すると、向きごと追従する三人称視点になります（Babylon.js の `camera.parent = dude` と同じ見え方）。
`FollowCamera` は Lite に無いため、この親子付けが代替手段です。

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 1, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
camera.parent = root;                                    // FollowCamera 代替

setCameraLimits(camera, { upperBetaLimit: Math.PI / 2.2 });   // 地面より下へ回り込まない
```

> RH→LH 変換の影響で前後が反転するので、カメラ角は `alpha = -Math.PI/2`（背後）にします。
> `upperBetaLimit` は Lite の `ArcRotateCamera` にもあり、`attachControl` の毎フレーム処理で制限が効きます。
> **`setCameraLimits` 経由が本来の作法**で、現在の姿勢が即座にクランプされるため、慣性による行き過ぎ→スナップのブレを避けられます
> （`camera.upperBetaLimit = …` の直接代入でも制限自体は効きます）。

---

← [3-06 キャラクターのアニメーション (Character Animation)](./3-06-character-animation.md) ・ [4-00 コリジョン回避 (Avoiding Collisions)](../04-collisions/4-00-avoiding-collisions.md) →
