# 6-03 スイッチ オン イベント (The Switch On Event) — △

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：クリックでパーティクルシステムを開始／停止する。

本家は、水を出す/止めるボタンのメッシュを置き、`scene.onPointerObservable` のピックで `particleSystem.start()` / `.stop()` を呼びます。
Lite でも **開始/停止 API とピッキングが揃っている**ので同じことができます（作り方の土台は [6-02](./6-02-particle-spray.md) の NPE パーティクルなので、判定は △）。

本サンプルでは専用のスイッチメッシュを置く代わりに、[6-02](./6-02-particle-spray.md) の**噴水メッシュそのものをクリック**して、パーティクルの再生/停止をトグルします。

**Lite 対応方針**（v1.9 ソースで確認）：

- **開始/停止** — `startParticleSystem(system)` / `stopParticleSystem(system)`。`stopParticleSystem` は新規放出を止め、残りの粒子を出し切ってから消えます。
  NPE 経由なので `registerNodeParticleSet(scene, set, { autoStart: false })` で止めた状態から始め、クリックで開始します。`set.systems` の各 system に対して呼びます。
- **クリック判定** — 本家の `scene.onPointerObservable` + `pickInfo.hit` / `pickedMesh` に相当する高レベルのポインタ機構は Lite に無いため、次で代替します。
  - canvas の DOM イベント `"pointerdown"` で座標を取り、`canvas.getBoundingClientRect()` でキャンバス内ピクセル座標へ変換。
  - `createGpuPicker(scene)` + `pickAsync(picker, x, y)` で **GPU ピッキング**し、返る `PickingInfo` の `hit` / `pickedMesh` を本家と同様に使います。

パーティクル本体は 6-02 と同じ NPE スニペット（`UPEOT4`）を参照します。スニペット ID は先頭 `#` を付けず `"UPEOT4"` と渡します（内部で `#`→`/` 変換するため、`#` 付きだと `//` になり失敗する）。

## メインコード

```typescript
import {
    addToScene,
    attachControl,
    createArcRotateCamera,
    createEngine,
    createGpuPicker,
    createHemisphericLight,
    createRibbon,
    createSceneContext,
    createStandardMaterial,
    parseNodeParticleSetFromSnippet,
    pickAsync,
    registerNodeParticleSet,
    registerScene,
    startEngine,
    startParticleSystem,
    stopParticleSystem,
    type EngineContext,
    type Mesh,
    type SceneContext,
} from "@babylonjs/lite";

const NPE_SNIPPET_ID = "UPEOT4"; // 先頭 "#" は付けない

type Vec3 = { x: number; y: number; z: number };

/** 本家 CreateLathe 相当: 輪郭を Y 軸まわりに回して createRibbon で張る */
function createLathe(engine: EngineContext, shape: readonly Vec3[], tessellation = 24): Mesh {
    const pathArray: Vec3[][] = [];
    for (let s = 0; s <= tessellation; s++) {
        const ang = (2 * Math.PI * s) / tessellation;
        const c = Math.cos(ang);
        const si = Math.sin(ang);
        pathArray.push(shape.map((p) => ({ x: p.x * c - p.z * si, y: p.y, z: p.x * si + p.z * c })));
    }
    return createRibbon(engine, { pathArray, closeArray: true });
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);

    const camera = createArcRotateCamera((3 * Math.PI) / 2, Math.PI / 2, 70, { x: 0, y: 0, z: 0 });
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    addToScene(scene, createHemisphericLight([0, 1, 0], 1));

    // 噴水プロファイル（本家と同一の8点）
    const fountainProfile: Vec3[] = [
        { x: 0, y: 0, z: 0 },
        { x: 10, y: 0, z: 0 },
        { x: 10, y: 4, z: 0 },
        { x: 8, y: 4, z: 0 },
        { x: 8, y: 1, z: 0 },
        { x: 1, y: 2, z: 0 },
        { x: 1, y: 15, z: 0 },
        { x: 3, y: 17, z: 0 },
    ];

    const fountain = createLathe(engine, fountainProfile, 24);
    const fountainMat = createStandardMaterial();
    fountainMat.backFaceCulling = false;
    fountainMat.diffuseColor = [0.6, 0.6, 0.7];
    fountain.material = fountainMat;
    fountain.position.y = -5;
    addToScene(scene, fountain);

    // パーティクル: NPE スニペットを読み込み、最初は止めておく（autoStart:false）
    const particleSet = await parseNodeParticleSetFromSnippet(engine, scene, NPE_SNIPPET_ID, {
        emitter: { x: 0, y: 10, z: 0 },
        textureBaseUrl: "https://playground.babylonjs.com/",
    });
    registerNodeParticleSet(scene, particleSet, { autoStart: false });

    // クリックで噴水を判定 → パーティクルのオン/オフ切り替え
    const picker = createGpuPicker(scene);
    let switched = false; // on/off フラグ（本家 switched）

    canvas.addEventListener("pointerdown", async (evt: PointerEvent) => {
        // クリック位置をキャンバス内ピクセル座標へ（本家 pickInfo 相当を GPU ピックで取得）
        const rect = canvas.getBoundingClientRect();
        const x = ((evt.clientX - rect.left) / rect.width) * canvas.clientWidth;
        const y = ((evt.clientY - rect.top) / rect.height) * canvas.clientHeight;

        const info = await pickAsync(picker, x, y);
        if (info.hit && info.pickedMesh === fountain) {
            switched = !switched; // トグル
            for (const system of particleSet.systems) {
                if (switched) {
                    startParticleSystem(system);
                } else {
                    stopParticleSystem(system);
                }
            }
        }
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
    await registerScene(scene);
    await startEngine(engine);
}

main().catch((error: unknown) => {
    console.error("シーンの構築に失敗しました:", error);
});
```

## 動作サンプル

<iframe src="https://liteplayground.babylonjs.com/snippet/OBNULB/v/2?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 6-03 スイッチ オン イベント（クリックで噴水をオン/オフ）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/OBNULB/v/2
>
> 噴水メッシュをクリックするとパーティクルが再生/停止します。本家の `scene.onPointerObservable` + `start()` / `stop()` を、Lite では GPU ピッキング（`createGpuPicker` / `pickAsync`）と `startParticleSystem` / `stopParticleSystem` で置き換えています。土台が NPE パーティクルのため判定は △ です。

## 発展サンプル：村の広場に噴水を配置

本家 Getting Started の締めと同じく、これまで組み立ててきた**村のシーン（`valleyvillage.glb`）の広場に噴水を置いた**発展版です。噴水を広場の座標（`FOUNTAIN_X = -3, FOUNTAIN_Z = -4`）へ配置し、スカイボックスを加えたうえで、上と同じ**クリックでのオン/オフ**（GPU ピッキング + `startParticleSystem` / `stopParticleSystem`）を組み込んでいます。

<iframe src="https://liteplayground.babylonjs.com/snippet/OBNULB/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 6-03 発展サンプル（村の広場の噴水）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/OBNULB/v/3
>
> 村シーンとの合成のためコードが長くなるため、ここでは全文は掲載せず**サンプルへのリンクのみ**とします。噴水の作り方（`createLathe`）・クリック判定・パーティクルのトグルは上のメインコードと同じ考え方で、そこへ `valleyvillage.glb` の読み込みとスカイボックスを重ねたものです。

---

← [6-02 パーティクルのスプレー (Particle Spray)](./6-02-particle-spray.md) ・ [7-00 光と影 (Light the Night)](../07-lights/7-00-light-intro.md) →
