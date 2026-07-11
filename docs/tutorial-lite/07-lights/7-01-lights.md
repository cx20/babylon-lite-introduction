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

## 街灯を増やす (More Street Lights)

本家の続きでは、街灯モデル（`lamp.babylon`）を読み込んで `clone` で複数に増やし、村（`valleyvillage`）へ並べて夜道を照らします。Lite でも読み込み・複製・ライト配置は可能ですが、**`.babylon` の読み込みとスポットライトの置き方に固有の注意点**があります。

**新要素と Lite での対応**：

- **(A) `.babylon` ファイルの読み込み【重要な制限あり】** — 本家 `SceneLoader.ImportMeshAsync(..., "lamp.babylon")` は `loadBabylon(engine, url)`（`AssetContainer` を返す）。ただし `lamp.babylon` は頂点データを別セクション `geometries`（`geometryId` 参照）に持ち、Lite の `loadBabylon` は**メッシュ直下の `positions` しか読まない**ため、そのままだとメッシュが空になり街灯が表示されません。
  - **対策** — コード内で `lamp.babylon` を `fetch` し、`geometries` を各メッシュ直下に**インライン化**（`positions` / `normals` / `indices` / `uvs` を移植）してから **Blob URL** 化して `loadBabylon` に渡します（マテリアルは色のみでテクスチャ無しなので `baseUrl` 不問）。
- **(B) メッシュの `clone`** — 本家 `mesh.clone("name")` は `cloneTransformNode(mesh)`。ジオメトリ（`_gpu`）を共有しつつ transform を独立させ、子（`bulb`）も再帰クローンし、`material` も引き継ぎます。
- **(C) `maxSimultaneousLights`** — Lite には**ありません**。本家は1メッシュに当たるライト数の上限を 5 に上げますが、Lite は `MAX_LIGHTS=16` でシーンの全ライトが全メッシュに当たるため、この設定は不要です。

**スポットライトはワールド空間に直接置く（親子付けしない）**：ライトを `bulb` に親子付けすると、`bulb` が継承する `lamp` のスケール 0.1 で「方向ベクトル」が縮み、Lite のコーン判定（正規化しない `dot` vs `cosHalfAngle`）が常に失敗して**消灯**します。街灯は静的なので、`bulb` のワールド位置（`worldMatrix` の `w[12], w[13], w[14]`）を読み、そこにワールド空間の下向きスポットライトを置きます。

```typescript
import {
    addToScene,
    attachControl,
    cloneTransformNode,
    createArcRotateCamera,
    createEngine,
    createHemisphericLight,
    createSceneContext,
    createSpotLight,
    loadBabylon,
    loadGltf,
    loadSkybox,
    registerScene,
    startEngine,
    type AssetContainer,
    type EngineContext,
    type SceneContext,
    type SceneNode,
} from "@babylonjs/lite";

const LAMP_URL = "https://assets.babylonjs.com/meshes/lamp.babylon";
const VALLEY_VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const SKYBOX_BASE_URL = "https://playground.babylonjs.com/textures/skybox";
const SKYBOX_EXT = ".jpg";

/**
 * lamp.babylon を読み込む。Lite の loadBabylon は geometries(geometryId 参照)を
 * 読まないため、fetch して各メッシュ直下に頂点データをインライン化してから
 * Blob URL 経由で loadBabylon に渡す。
 */
async function loadLampInlined(engine: EngineContext, url: string): Promise<AssetContainer> {
    const res = await fetch(url);
    const data = (await res.json()) as {
        meshes?: { geometryId?: string; positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }[];
        geometries?: { vertexData?: { id: string; positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }[] };
    };
    // geometryId → 頂点データ
    const geo = new Map<string, { positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }>();
    for (const v of data.geometries?.vertexData ?? []) {
        geo.set(v.id, v);
    }
    // 各メッシュ直下に positions/normals/indices/uvs を移植
    for (const m of data.meshes ?? []) {
        const g = m.geometryId ? geo.get(m.geometryId) : undefined;
        if (g) {
            m.positions = g.positions;
            m.normals = g.normals;
            m.indices = g.indices;
            if (g.uvs) {
                m.uvs = g.uvs;
            }
        }
    }
    delete data.geometries; // インライン化済み

    // Blob URL 化して loadBabylon に渡す（fetch は blob: URL を扱える）
    const blob = new Blob([JSON.stringify(data)], { type: "application/json" });
    const blobUrl = URL.createObjectURL(blob);
    try {
        return await loadBabylon(engine, blobUrl);
    } finally {
        URL.revokeObjectURL(blobUrl);
    }
}

/** ノードの子孫から名前の一部が一致するものを探す（clone で名前に _clone が付くため部分一致） */
function findDescendant(root: SceneNode, namePart: string): SceneNode | null {
    if (root.name && root.name.includes(namePart)) {
        return root;
    }
    for (const child of root.children ?? []) {
        if (child && typeof child === "object" && "name" in child && "children" in child) {
            const found = findDescendant(child as SceneNode, namePart);
            if (found) {
                return found;
            }
        }
    }
    return null;
}

/** コンテナ直下のエンティティから名前一致のノードを探す */
function findInContainer(container: AssetContainer, namePart: string): SceneNode {
    for (const entity of container.entities) {
        if (entity && typeof entity === "object" && "name" in entity && "children" in entity) {
            const found = findDescendant(entity as SceneNode, namePart);
            if (found) {
                return found;
            }
        }
    }
    throw new Error(`ノード "${namePart}" が見つかりません。`);
}

/**
 * 指定した街灯(lamp)の bulb のワールド位置を読み、そこに【ワールド空間の】
 * 下向きスポットライトを置いて点灯する。
 *
 * ※ライトを bulb に親子付けしない理由: bulb は lamp のスケール0.1を継承する。
 *   Lite のスポット方向はワールド行列の列2をそのまま使い（正規化しない）、
 *   親のスケール0.1で方向長が0.1に縮む→コーン判定 dot が最大0.1にしかならず
 *   cosHalfAngle(≈0.309)を超えられず常に消灯になる。ワールド空間に直接置けば
 *   方向(0,-1,0)が正しい長さを保ち、地面が照らされる。街灯は静的なので
 *   親子付け（追従）は不要。
 */
function addLampLight(scene: SceneContext, lamp: SceneNode): void {
    const bulb = findDescendant(lamp, "bulb");
    if (!bulb) {
        return;
    }
    // bulb のワールド位置（worldMatrix 列3 = w[12],w[13],w[14]）
    const w = bulb.worldMatrix;
    const pos: [number, number, number] = [w[12]!, w[13]!, w[14]!];
    // 本家: direction=(0,-1,0), angle=0.8π, exponent=0.01
    const lampLight = createSpotLight(pos, [0, -1, 0], 0.8 * Math.PI, 0.01, 1);
    lampLight.diffuse = [1, 1, 0]; // Color3.Yellow()
    addToScene(scene, lampLight);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);

    // カメラ（本家と同一: alpha = -π/2.2, beta = π/2.2, radius 15）
    const camera = createArcRotateCamera(-Math.PI / 2.2, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
    camera.upperBetaLimit = Math.PI / 2.2;
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // 半球光は夜なので暗めに（intensity 0.1）。方向は本家 (1, 1, 0)
    addToScene(scene, createHemisphericLight([1, 1, 0], 0.1));

    // 街灯・村・スカイボックスを並列ロード（registerScene の前に await 完了）
    const [lampContainer, village] = await Promise.all([
        loadLampInlined(engine, LAMP_URL),
        loadGltf(engine, VALLEY_VILLAGE_URL),
        loadSkybox(scene, SKYBOX_BASE_URL, SKYBOX_EXT, 150),
    ]);
    addToScene(scene, village);

    // 元の街灯を取得・配置
    const lamp = findInContainer(lampContainer, "lamp");
    lamp.position.set(2, 0, 2);
    lamp.rotation.set(0, -Math.PI / 4, 0);
    addToScene(scene, lampContainer); // 元の lamp（と bulb）を追加
    addLampLight(scene, lamp);

    // clone で街灯を増やす（本家 lamp.clone(...) 相当 = cloneTransformNode）
    const lamp3 = cloneTransformNode(lamp);
    lamp3.position.set(2, 0, -8);
    lamp3.rotation.set(0, -Math.PI / 4, 0);
    addToScene(scene, lamp3);
    addLampLight(scene, lamp3);

    const lamp1 = cloneTransformNode(lamp);
    lamp1.position.set(-8, 0, 1.2);
    lamp1.rotation.set(0, Math.PI / 2, 0);
    addToScene(scene, lamp1);
    addLampLight(scene, lamp1);

    const lamp2 = cloneTransformNode(lamp1);
    lamp2.position.set(-2.7, 0, 0.8);
    lamp2.rotation.set(0, -Math.PI / 2, 0);
    addToScene(scene, lamp2);
    addLampLight(scene, lamp2);

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

<iframe src="https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 7-01 街灯を増やす（More Street Lights）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/1
>
> `lamp.babylon` を読み込み、`cloneTransformNode` で 4 本に増やして村へ配置し、各街灯にスポットライトを点灯させています。Lite 固有のハマりどころは、**`.babylon` の `geometries` インライン化**と、**スポットライトを親子付けせずワールド空間に置く**ことの 2 点です。

---

← [7-00 光と影 (Light the Night)](./7-00-light-intro.md) ・ [7-02 昼から夜へ (Day to Night)](./7-02-day-to-night.md) →
