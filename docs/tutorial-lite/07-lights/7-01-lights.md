# 7-01 ライトを灯す (Light the Night) — ○

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：街灯（ポール＋電球＋スポットライト）を作り、電球にスポットライトを親子付けして夜の街を照らす。

本家の「Light the Night」は、押し出しで街灯のポールを作り、電球（球）とスポットライトを電球に親子付けして、半球光を暗めにします。Lite でも**スポットライト・シェイプの押し出し・親子付け**が揃っているので同じことができます（判定は ○）。

**Lite 対応方針**（v1.9 ソースで確認）：

- **(A) スポットライト** — 本家 `new SpotLight(name, position, direction, angle, exponent)` は `createSpotLight(position, direction, angle, exponent, intensity)`。引数の意味は同じ（`angle`＝広がり、`exponent`＝減衰）。色は `diffuse` に `[r, g, b]` を代入（本家 `Color3.Yellow()` = `[1, 1, 0]`）。`addToScene` で登録します。
- **(B) シェイプの押し出し** — 本家 `MeshBuilder.ExtrudeShape({ cap, shape, path, scale })` は `createExtrudeShape(engine, { shape, path, scale, cap })`。`cap` は `CAP_END` 定数をそのまま使い、`shape` / `path` は `Vec3` の配列です。
- **(C) 親子付け** — メッシュ同士は本家 `child.parent = parent` に相当する `setParent(child, parent)`（ワールド変換を保持したまま親子関係を張る）。
  ただし**ライトは例外**です。`setParent` は内部で `rotationQuaternion` / `scaling` を書き込みますが、ライトはそれらを持たないため `setParent` では落ちます。ライトは **`parent` プロパティを直接代入**すると `worldMatrix` が親を反映して追従します。
- 半球光は暗めに（`createHemisphericLight(direction, 0.5)`）。電球は自己発光させるため `emissiveColor` に黄色を入れます。

## メインコード

```typescript
import {
    addToScene,
    attachControl,
    CAP_END,
    createArcRotateCamera,
    createEngine,
    createExtrudeShape,
    createGround,
    createHemisphericLight,
    createSceneContext,
    createSphere,
    createSpotLight,
    createStandardMaterial,
    registerScene,
    setParent,
    startEngine,
    type EngineContext,
    type SceneContext,
} from "@babylonjs/lite";

type Vec3 = { x: number; y: number; z: number };

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);

    // カメラ（本家と同一: alpha = 3π/2, beta = π/2.2, radius 50）
    const camera = createArcRotateCamera((3 * Math.PI) / 2, Math.PI / 2.2, 50, { x: 0, y: 0, z: 0 });
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // 半球光は暗めに（intensity 0.5）。方向は本家 (0, 50, 0)＝上向き
    addToScene(scene, createHemisphericLight([0, 50, 0], 0.5));

    // スポットライト（本家: position=原点, direction=(0,-1,0), angle=π, exponent=1）
    // 色は黄色。後で電球(bulb)の子にして、電球と一緒に動くようにする。
    const lampLight = createSpotLight([0, 0, 0], [0, -1, 0], Math.PI, 1, 1);
    lampLight.diffuse = [1, 1, 0]; // Color3.Yellow()
    addToScene(scene, lampLight);

    // 押し出す断面形状（半径1の円を20分割、最後に始点を追加して閉じる）
    const lampShape: Vec3[] = [];
    for (let i = 0; i < 20; i++) {
        lampShape.push({ x: Math.cos((i * Math.PI) / 10), y: Math.sin((i * Math.PI) / 10), z: 0 });
    }
    lampShape.push(lampShape[0]!); // 形状を閉じる

    // 押し出しの経路（ポール→上部の曲がり→アーム先端）
    const lampPath: Vec3[] = [];
    lampPath.push({ x: 0, y: 0, z: 0 });
    lampPath.push({ x: 0, y: 10, z: 0 });
    for (let i = 0; i < 20; i++) {
        lampPath.push({
            x: 1 + Math.cos(Math.PI - (i * Math.PI) / 40),
            y: 10 + Math.sin(Math.PI - (i * Math.PI) / 40),
            z: 0,
        });
    }
    lampPath.push({ x: 3, y: 11, z: 0 });

    // 街灯ポールを押し出しで生成（端をキャップ、太さ 0.5 に縮小）
    const lamp = createExtrudeShape(engine, { shape: lampShape, path: lampPath, scale: 0.5, cap: CAP_END });
    lamp.material = createStandardMaterial();
    addToScene(scene, lamp);

    // 電球（非等方球）。自己発光の黄色マテリアル。
    const yellowMat = createStandardMaterial();
    yellowMat.emissiveColor = [1, 1, 0]; // Color3.Yellow()
    const bulb = createSphere(engine, { diameterX: 1.5, diameterZ: 0.8 });
    bulb.material = yellowMat;
    addToScene(scene, bulb);

    // 親子付け: lamp ← bulb ← lampLight。
    // メッシュ同士(bulb→lamp)は setParent（ワールド変換保持＝TRS 分解して復元）。
    bulb.position.set(2, 10.5, 0);
    setParent(bulb, lamp);
    // ライトは rotationQuaternion/scaling を持たず setParent は使えない
    // （setParent は TRS を書き込むためライトで落ちる）。ライトは parent プロパティ
    // を直接代入すれば worldMatrix が親を反映して追従する。position は親ローカル
    // 基準になるので、電球の原点(=ローカル0)から direction(0,-1,0)方向へ照らす。
    lampLight.position.set(0, 0, 0);
    lampLight.parent = bulb;

    // 地面 50×50
    const ground = createGround(engine, { width: 50, height: 50 });
    ground.material = createStandardMaterial();
    addToScene(scene, ground);

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

<iframe src="https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 7-01 ライトを灯す（街灯）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/0
>
> 押し出したポールの先に電球を親子付けし、その電球にスポットライトを親子付けしています。メッシュ同士は `setParent`、ライトは `parent` 直接代入と使い分けるのがポイントです。

---

← [7-00 光と影 (Light the Night)](./7-00-light-intro.md) ・ [7-02 影を追加 (Adding Shadows)](./7-02-shadows.md) →
