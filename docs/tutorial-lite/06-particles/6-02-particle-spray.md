# 6-02 パーティクルのスプレー (Particle Spray) — △

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：基本的なパーティクルシステムを作り、噴水の口から水しぶきを噴き出す。

**Lite にパーティクルは実装されています**（v1.9 ソースで確認）。ただし本家とは**作り方が違います**。

- 本家は命令的に `new ParticleSystem(...)` を作り、`emitter` / `direction1,2` / `minEmitBox` / `gravity` / `color1,2` / `minLifeTime` などのプロパティを設定します。
- **Lite は Node Particle Editor (NPE) 前提**です。パーティクルの挙動（放出・寿命・方向・色・重力）は**ノードグラフ (NPE) で組み**、`parseNodeParticleSetFromSnippet` で読み込みます（命令型 `ParticleSystem` API は非公開）。

そのため章の主目的（水しぶき）は達成できますが、チュートリアルのコードをそのまま移植する形にはなりません。判定は △ です。

## 全体構成

移植版は 2 ファイル構成です。

- **メインコード**（下に掲載）— 噴水メッシュ（`createLathe`）を作り、NPE パーティクルグラフを読み込んで器の口 `(0, 10, 0)` から噴かせます。
- **`fountain-npe.js`** — 本家 fountain の値を焼き込んだ NPE グラフ（約 1,500 行）。cx20 氏の `dust-npe.js` と同じく「参照グラフに値を焼き込む」方式です。長いので下では**要点だけ**説明します。

`parseNodeParticleSetFromSnippet` にはスニペット ID の代わりに `json:` でグラフ本体を直接渡しています（`emitter` で放出位置、`textureBaseUrl` で `flare.png` の解決先を指定）。

追加 import：`createLathe` 用の `createRibbon` / `createStandardMaterial`、パーティクル用の `parseNodeParticleSetFromSnippet` / `registerNodeParticleSet`。

## メインコード

```typescript
import {
    addToScene,
    attachControl,
    createArcRotateCamera,
    createEngine,
    createHemisphericLight,
    createRibbon,
    createSceneContext,
    createStandardMaterial,
    parseNodeParticleSetFromSnippet,
    registerNodeParticleSet,
    registerScene,
    startEngine,
    type EngineContext,
    type Mesh,
    type SceneContext,
} from "@babylonjs/lite";
import { createFountainNpe } from "./fountain-npe.js";

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

    // カメラ（本家と同一: alpha = 3π/2, beta = π/2, radius 70）
    const camera = createArcRotateCamera((3 * Math.PI) / 2, Math.PI / 2, 70, { x: 0, y: 0, z: 0 });
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // ライト（本家と同じ上方向 (0, 1, 0)）
    addToScene(scene, createHemisphericLight([0, 1, 0], 1));

    // 噴水プロファイル（本家と同一の 8 点）
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

    // 回転体で噴水を生成（DOUBLESIDE 相当は backFaceCulling=false）
    const fountain = createLathe(engine, fountainProfile, 24);
    const fountainMat = createStandardMaterial();
    fountainMat.backFaceCulling = false;
    fountainMat.diffuseColor = [0.6, 0.6, 0.7];
    fountain.material = fountainMat;
    fountain.position.y = -5;
    addToScene(scene, fountain);

    // パーティクル: 焼き込み済み NPE グラフを読み込み、噴水の上 (0,10,0) から噴かせる
    const particleSet = await parseNodeParticleSetFromSnippet(engine, scene, "", {
        json: createFountainNpe(),
        emitter: { x: 0, y: 10, z: 0 },                       // 本家 particleSystem.emitter = (0,10,0)
        textureBaseUrl: "https://playground.babylonjs.com/",  // JSON 内 textures/flare.png の解決先
    });
    registerNodeParticleSet(scene, particleSet, { autoStart: true });

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

## `fountain-npe.js` の要点（かいつまんで）

`fountain-npe.js` は約 1,500 行の巨大な JSON（NPE グラフ）ですが、中身は**ブロックの配列**で、全体像さえ掴めば読めます。ここでは全文は載せず、**構造と焼き込んだ値**だけ押さえます。

### 焼き込んだ本家 fountain の値

本家の命令型設定に対応する値を、各 `ParticleInputBlock` の `value` に直接書き込んでいます。

| 本家プロパティ | 焼き込んだ値 |
|---|---|
| `capacity` / `emitRate` | 5000 / 1500 |
| `blendMode` | 0（ONEONE = 加算） |
| `updateSpeed` | 0.025 |
| `emitPower` / `lifeTime` / `size` | 1–3 / 2–3.5 / 0.1–0.5（各 `ParticleRandomBlock` で範囲指定） |
| `direction1` / `direction2` | (-2, 8, 2) / (2, 8, -2) |
| `minEmitBox` / `maxEmitBox` | (-1, 0, 0) / (1, 0, 0) |
| `color1` / `color2` / `colorDead` | (0.7, 0.8, 1, 1) / (0.2, 0.5, 1, 1) / (0, 0, 0.2, 0) |
| `texture` | `textures/flare.png` |

### グラフの流れ

ブロックはおおむね次の順に繋がっています（末尾の `SystemBlock` を根に、入力を辿る形）。

1. **CreateParticleBlock** — 各粒子の初期値（emitPower / lifeTime / color / size …）を `ParticleRandomBlock` で乱数化して生成。
2. **BoxShapeBlock** — `direction1,2` と `minEmitBox` / `maxEmitBox` から放出方向と放出位置を決定。
3. **UpdateColorBlock** — 寿命に応じて色を `colorDead` へ補間。
4. **UpdateDirectionBlock**（重力ブロック、下記）— 速度を毎ステップ更新。
5. **UpdatePositionBlock** — `Scaled Direction`（＝速度 × dt）で位置を進める。

### 重力ブロック（本家 `gravity=(0,-9.81,0)` の再現）

本家の `gravity` は単一プロパティですが、NPE には重力の既製ブロックが無いため、**速度を毎ステップ更新するノード列**で表現しています。

```
Gravity(定数 [0,-9.81,0]) ── Math乗算 ──┐
Delta(systemSource=2 = そのステップの dt) ┘   （= gravity × delta）
                                          │
Direction(contextualValue=2 = 現在の粒子速度) ── Math加算 ── UpdateDirectionBlock.direction
```

つまり毎ステップ `速度 = 速度 + gravity × delta` を適用します。このブロックを **UpdateColor と UpdatePosition の間**に挿入することで、位置が `Scaled Direction` で進む前に速度が重力で更新され、粒子は**上昇 → 減速 → 下降**と弧を描いて落ちます。

```javascript
// グラフの新しいコピーを返す（parseNodeParticleSetFromSnippet はオブジェクトを
// 消費するため、パースのたびに fresh copy を渡す）。
export function createFountainNpe() {
    return structuredClone(FOUNTAIN_NPE);
}
```

## 動作サンプル

<iframe src="https://liteplayground.babylonjs.com/snippet/OBNULB/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 6-02 パーティクルのスプレー（噴水）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/OBNULB/v/0
>
> 本家のように「エミッタ・方向・重力をコードで直接設定する」書き味ではなく、**NPE でグラフを組む**のが Lite の流儀です。この差があるため ○ ではなく △ としています。
> 現在はグラフ本体を `json:` で直接渡していますが、**NPE スニペット ID が用意できたら、`parseNodeParticleSetFromSnippet(engine, scene, "<スニペット ID>")` で読み込む別バージョン**として反映します。

---

← [6-01 旋盤で回された噴水 (A Lathe Turned Fountain)](./6-01-lathe-fountain.md) ・ [6-03 スイッチ オン イベント (The Switch On Event)](./6-03-switch-on-event.md) →
