# 8-02 キャラを追う (Follow That Character) — △

> [第8部：世界の見方](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：村を巡回する Xbot を、`FollowCamera` が滑らかに追跡する。木はスプライト 1000 本、ライトはヘミスフェリック（本家どおり影なし）です。

**Lite 対応方針**（v1.8 ソースで確認）。本家の **`FollowCamera` は Lite に無い**（該当 export なし）ため、`FreeCamera` を土台に本家の平滑化アルゴリズムを再実装します。

### `FollowCamera` を `FreeCamera` でエミュレートする

Lite に `FollowCamera` はありませんが、**`FreeCamera`（位置＋注視点型、両方とも毎フレーム書き換え可能な `ObservableVec3`）**があります。これを土台に、本家 `FollowCamera._follow()` の平滑化アルゴリズムをそのまま再実装して駆動します。

- **目標位置** ＝ ターゲット位置 ＋ `(sin(yaw+rotationOffset), 0, cos(yaw+rotationOffset)) × radius` ＋ `(0, heightOffset, 0)`
  （`(sin, cos) × radius` は進行方向の「真後ろ」＝ `forward = (-sin, -cos)`）
- **速度** ＝ 差分 × `cameraAcceleration`（x,z は ×2）を毎フレーム加算し、各成分を `±maxCameraSpeed` でクランプ
- **注視点** ＝ ターゲット位置（毎フレーム）

キャラが瞬間的に転回してもカメラは加速度制限で緩やかに回り込みます（この章の主題「滑らかな追跡」）。

### 8-01 の肩越しカメラとの違い

[8-01 見回す](./8-01-look-around.md#メインコード)は `ArcRotateCamera` の `alpha` を転回に即同期させる「剛体的な肩越し追従」でした。こちらは `FreeCamera` の位置を**加速度・最高速度でなまして**目標へ寄せるため、転回時にカメラが遅れて回り込む**慣性のある追跡**になります。本家 `attachControl`（`FollowCamera` ではキーボードで `radius`/`rotationOffset`/`heightOffset` の目標値を変える入力）は省略しています（必要なら `keydown` で定数を書き換えれば同等）。`attachFreeControl`（WASD 自由移動）は毎フレームの追跡更新と衝突するため繋ぎません。

### その他は前章までの確立規約

bake クォータニオン合成／`world yaw φ` と x 反転変換／コンテナ丸ごと `addToScene`＋`walk` 切り替え／スプライト＝`loadSpriteAtlas`＋`FacingBillboardSystem`／`loadSkybox(…, 1000)`＋`farPlane 2000` は、[8-01](./8-01-look-around.md) と共通です。

## メインコード

```typescript
/********************************************************************
 * Getting Started: Follow That Character（FollowCamera） — Babylon Lite 移植版
 * ------------------------------------------------------------------
 * 村を巡回する Xbot を FollowCamera が滑らかに追跡する。
 * 木はスプライト1000本、ライトはヘミスフェリック（本家どおり影なし）。
 *
 * ── FollowCamera の Lite 対応 ────────────────────────────────
 *   Lite に FollowCamera は無いが、FreeCamera（位置＋注視点、毎フレーム
 *   書き換え可）があるので、BJS の FollowCamera._follow() の平滑化
 *   アルゴリズムをそのまま再実装して駆動する:
 *     目標位置 = ターゲット位置
 *              + (sin(yaw+rotationOffset), 0, cos(yaw+rotationOffset)) × radius
 *              + (0, heightOffset, 0)
 *       ※ (sin,cos)×radius は進行方向の「真後ろ」（forward = (-sin,-cos)）
 *     速度     = 差分 × cameraAcceleration（x,z は×2） を毎フレーム加算、
 *                各成分 ±maxCameraSpeed でクランプ
 *     注視点   = ターゲット位置（毎フレーム）
 *   キャラが瞬間的に転回してもカメラは加速度制限で緩やかに回り込む
 *   （この章の主題「滑らかな追跡」）。
 *
 *   本家の camera.attachControl は FollowCamera ではキーボードで
 *   radius/rotationOffset/heightOffset の目標値を変える入力だが、
 *   ここでは省略（必要なら keydown で下の定数を書き換えれば同等）。
 *   ※ attachFreeControl(WASD 自由移動) を繋ぐと毎フレームの追跡更新と
 *     衝突するため使わない。
 *
 * ── その他は前章までの確立規約 ─────────────────────────────
 *   bake クォータニオン合成 / world yaw φ と x 反転変換 /
 *   コンテナ丸ごと addToScene + walk 切り替え /
 *   スプライト = loadSpriteAtlas + FacingBillboardSystem /
 *   loadSkybox("textures/skybox", ".jpg", 1000) + farPlane 2000
 ********************************************************************/
import {
    addBillboardSpriteIndex,
    addFacingBillboardSystem,
    addToScene,
    createEngine,
    createFacingBillboardSystem,
    createFreeCamera,
    createHemisphericLight,
    createSceneContext,
    loadGltf,
    loadSkybox,
    loadSpriteAtlas,
    onBeforeRender,
    playAnimation,
    registerScene,
    startEngine,
    stopAnimation,
    type AnimationGroup,
    type AssetContainer,
    type EngineContext,
    type SceneContext,
    type SceneNode,
} from "@babylonjs/lite";

const VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const XBOT_URL = "https://playground.babylonjs.com/scenes/Xbot.glb";
const PALM_URL = "https://playground.babylonjs.com/textures/palm.png";
const SKYBOX_BASE = "https://playground.babylonjs.com/textures/skybox";

const toRad = (deg: number): number => (deg * Math.PI) / 180;

/** FollowCamera パラメータ（本家と同一） */
const CAM_START = { x: -6, y: 0, z: 0 };
const CAM_RADIUS = 1; // ターゲットからの水平距離
const CAM_HEIGHT_OFFSET = 8; // ターゲット中心からの高さ
const CAM_ROTATION_OFFSET = 0; // ターゲット周りの回転（度）
const CAM_ACCELERATION = 0.005; // 目標位置へ向かう加速度
const CAM_MAX_SPEED = 10; // 速度の上限

/** 巡回コース（本家と同一） */
const TRACK: ReadonlyArray<{ turn: number; dist: number }> = [
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

function requireGroup(groups: readonly AnimationGroup[], name: string): AnimationGroup {
    const group = groups.find((g) => g.name === name);
    if (!group) {
        throw new Error(`アニメーション "${name}" が見つかりません`);
    }
    return group;
}

function findNode(container: AssetContainer, name: string): SceneNode {
    const visit = (node: SceneNode): SceneNode | null => {
        if (node.name === name) {
            return node;
        }
        for (const child of node.children ?? []) {
            const found = visit(child);
            if (found) {
                return found;
            }
        }
        return null;
    };
    for (const entity of container.entities) {
        if ("children" in (entity as object)) {
            const found = visit(entity as SceneNode);
            if (found) {
                return found;
            }
        }
    }
    throw new Error(`ノード "${name}" が見つかりません`);
}

/** BJS FollowCamera の速度クランプ（各成分 ±max） */
function clampSpeed(v: number): number {
    if (v > CAM_MAX_SPEED) {
        return CAM_MAX_SPEED;
    }
    if (v < -CAM_MAX_SPEED) {
        return -CAM_MAX_SPEED;
    }
    return v;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);
    scene.clearColor = { r: 0.53, g: 0.75, b: 0.92, a: 1.0 }; // スカイボックス失敗時の空色

    // FollowCamera 相当（本家: new FollowCamera("FollowCam", (-6,0,0))）
    const camera = createFreeCamera(CAM_START, { x: 0, y: 0, z: 0 });
    camera.nearPlane = 0.1;
    camera.farPlane = 2000; // スカイボックス(1000)より十分遠くまで描画
    scene.camera = camera;

    // ヘミスフェリックライト（本家と同一: direction=(1,1,0)。この章は影なし）
    addToScene(scene, createHemisphericLight([1, 1, 0], 1));

    // 村（コンテナ丸ごと追加）
    const village = await loadGltf(engine, VILLAGE_URL);
    addToScene(scene, village);

    // Xbot（コンテナ丸ごと追加 → walk へ切り替え）
    const xbot = await loadGltf(engine, XBOT_URL);
    addToScene(scene, xbot);
    const groups = xbot.animationGroups ?? [];
    for (const g of groups) {
        stopAnimation(g);
    }
    playAnimation(requireGroup(groups, "walk"));

    // 巡回の駆動ノード（bake クォータニオン保持）。半分サイズ。
    const dude = findNode(xbot, "Xbot");
    const bake = { x: dude.rotationQuaternion.x, y: dude.rotationQuaternion.y, z: dude.rotationQuaternion.z, w: dude.rotationQuaternion.w };
    const CHAR_SCALE = 0.5;
    const ds = dude.scaling;
    ds.set(ds.x * CHAR_SCALE, ds.y * CHAR_SCALE, ds.z * CHAR_SCALE);

    // 初期状態（ワールド意図: 位置(-6,0,0)・yaw -95°）→ ローカルへ（x反転）
    const START_YAW = toRad(-95);
    let yaw = START_YAW;
    dude.position.set(6, 0, 0);

    const applyHeading = (): void => {
        const psi = Math.PI - yaw; // 正面180°オフセット + x反転の符号反転
        const sy = Math.sin(psi / 2);
        const cy = Math.cos(psi / 2);
        dude.rotationQuaternion.set(
            cy * bake.x + sy * bake.z,
            cy * bake.y + sy * bake.w,
            cy * bake.z - sy * bake.x,
            cy * bake.w - sy * bake.y
        );
    };
    applyHeading();

    // スプライトの木（本家 SpriteManager("palm.png", 2000, {512,1024}) 相当）
    const palmAtlas = await loadSpriteAtlas(engine, PALM_URL, {
        gridSize: [512, 1024], // 画像全体を1フレームに
        sampling: "linear",
    });
    const trees = createFacingBillboardSystem(palmAtlas, { capacity: 1000 });
    for (let i = 0; i < 500; i++) {
        addBillboardSpriteIndex(trees, {
            position: [Math.random() * -30, 0.5, Math.random() * 20 + 8],
            sizeWorld: [1, 1],
        });
    }
    for (let i = 0; i < 500; i++) {
        addBillboardSpriteIndex(trees, {
            position: [Math.random() * 25 + 7, 0.5, Math.random() * -35 + 8],
            sizeWorld: [1, 1],
        });
    }
    addFacingBillboardSystem(scene, trees);

    // 巡回＋FollowCamera 追跡（本家 onBeforeRenderObservable 相当）
    let distance = 0;
    const STEP = 0.015; // この章の本家値
    let p = 0;
    onBeforeRender(scene, () => {
        // 移動（world forward=(-sinφ,0,-cosφ) → local へは x 反転）
        dude.position.x += Math.sin(yaw) * STEP;
        dude.position.z += -Math.cos(yaw) * STEP;
        distance += STEP;

        if (distance > TRACK[p]!.dist) {
            yaw += toRad(TRACK[p]!.turn);
            applyHeading();
            p = (p + 1) % TRACK.length;
            if (p === 0) {
                distance = 0;
                dude.position.set(6, 0, 0);
                yaw = START_YAW;
                applyHeading();
            }
        }

        // ── FollowCamera._follow() 忠実再現 ─────────────────────
        // ターゲットのワールド位置（worldMatrix の平行移動成分から直接取得）
        const w = dude.worldMatrix as unknown as ArrayLike<number>;
        const tx = w[12]!;
        const ty = w[13]!;
        const tz = w[14]!;
        // 目標位置: yaw+rotationOffset 方向（＝真後ろ）に radius、上に heightOffset
        const radians = toRad(CAM_ROTATION_OFFSET) + yaw;
        const goalX = tx + Math.sin(radians) * CAM_RADIUS;
        const goalZ = tz + Math.cos(radians) * CAM_RADIUS;
        // 速度 = 差分×加速度（x,z は×2）、各成分 ±maxCameraSpeed
        const vx = clampSpeed((goalX - camera.position.x) * CAM_ACCELERATION * 2);
        const vy = clampSpeed((ty + CAM_HEIGHT_OFFSET - camera.position.y) * CAM_ACCELERATION);
        const vz = clampSpeed((goalZ - camera.position.z) * CAM_ACCELERATION * 2);
        camera.position.set(camera.position.x + vx, camera.position.y + vy, camera.position.z + vz);
        // 注視点は常にターゲット（本家 setTarget(targetPosition)）
        camera.target.set(tx, ty, tz);
    });

    return scene;
}

async function main(): Promise<void> {
    const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement | null;
    if (!canvas) {
        throw new Error('Canvas要素 "#renderCanvas" が見つかりません。');
    }
    const engine = await createEngine(canvas);
    const scene = await createScene(engine, canvas);

    // スカイボックス（本家 CubeTexture("textures/skybox") と同じ6枚jpg）
    try {
        await loadSkybox(scene, SKYBOX_BASE, ".jpg", 1000);
    } catch (e) {
        console.warn("スカイボックスの読み込みに失敗（clearColor で続行）:", e);
    }
    scene.clearColor = { r: 0.53, g: 0.75, b: 0.92, a: 1.0 };

    await registerScene(scene);
    await startEngine(engine);
}

main().catch((error: unknown) => {
    console.error("シーンの構築に失敗しました:", error);
});
```

## 動作サンプル

<iframe src="https://liteplayground.babylonjs.com/snippet/O42L06/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 8-02 キャラを追う（FollowCamera エミュレーション）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/O42L06/v/3
>
> 村を巡回する Xbot を、カメラが慣性を持って滑らかに追跡します。Lite に `FollowCamera` は無いので、**`FreeCamera`（位置＋注視点の `ObservableVec3`）を土台に本家 `FollowCamera._follow()` の平滑化アルゴリズム（目標位置へ加速度・最高速度でなます）を再実装**します。8-01 の剛体的な肩越し追従に対し、転回時にカメラが遅れて回り込む慣性のある追跡になるのが違いです。本家 `FollowCamera` そのものではなく代替実装のため、判定は △ です。

---

← [8-01 見回す (Have a Look Around)](./8-01-look-around.md) ・ [8-03 VR の世界へ (Going Virtual)](./8-03-going-virtual.md) →
