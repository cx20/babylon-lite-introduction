# 7-02 昼から夜へ (Day to Night) — △

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：[前サンプル（複数の街灯）](./7-01-lights.md)に、半球光の `intensity` をスライダーで変えて昼→夜を切り替える UI を足す。

**Lite 対応方針**（v1.9 ソースで確認）。街灯部分は [7-01](./7-01-lights.md) で確立済みなので、ここでは新要素の **2 つの注意点**が要点です。

### GUI は Lite に無い → HTML オーバーレイで代替

本家は `BABYLON.GUI`（`AdvancedDynamicTexture` / `StackPanel` / `Slider`）を使いますが、**Babylon Lite は 3D レンダリング専用の軽量ランタイムで GUI ライブラリを含みません**（`AdvancedDynamicTexture` も `DynamicTexture` も非公開）。本家 GUI の実体は「値を取って `light.intensity` を更新する」だけなので、標準の HTML `<input type="range">` を canvas に重ねて同じことをします。**Lite で 2D UI が要る場合は、HTML/CSS で作って canvas にオーバーレイするのが定石**です（3D シーンと DOM UI は別レイヤーとして共存できます）。

### `intensity` を変えても即座に反映されない → `direction.set` で dirty 化

Lite はライト UBO を「`_lightVersion`（dirty カウンタ）が変化したときだけ」GPU に再アップロードします。ところが `light.intensity` は**単純プロパティ**で、代入しても `_lightVersion` が増えないため、再アップロードされず変化が反映されません。

対策として、`intensity` を変えた後に `direction.set(同値)` を呼びます。`ObservableVec3.set` は**同値でも `onDirty` を発火**して `_lightVersion` を増やすので、再アップロードが走って `intensity` 変更が反映されます。

```typescript
slider.addEventListener("input", () => {
    hemiLight.intensity = Number(slider.value); // 本家 onValueChangedObservable 相当
    // intensity は dirty 通知しないため、direction.set(同値) で再アップロードを強制する
    const d = hemiLight.direction;
    d.set(d.x, d.y, d.z);
});
```

## メインコード

街灯の読み込み・複製・点灯は [7-01「街灯を増やす」](./7-01-lights.md#街灯を増やす-more-street-lights)と同じ手法です（`lamp.babylon` の `geometries` インライン化、`cloneTransformNode`、スポットライトはワールド空間へ直接配置）。ここに `createDayNightSlider` を足したのが差分です。

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
    type HemisphericLight,
    type SceneContext,
    type SceneNode,
} from "@babylonjs/lite";

const LAMP_URL = "https://assets.babylonjs.com/meshes/lamp.babylon";
const VALLEY_VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const SKYBOX_BASE_URL = "https://playground.babylonjs.com/textures/skybox";
const SKYBOX_EXT = ".jpg";

/** lamp.babylon を geometries インライン化 → Blob URL 経由で loadBabylon */
async function loadLampInlined(engine: EngineContext, url: string): Promise<AssetContainer> {
    const res = await fetch(url);
    const data = (await res.json()) as {
        meshes?: { geometryId?: string; positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }[];
        geometries?: { vertexData?: { id: string; positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }[] };
    };
    const geo = new Map<string, { positions?: number[]; normals?: number[]; indices?: number[]; uvs?: number[] }>();
    for (const v of data.geometries?.vertexData ?? []) {
        geo.set(v.id, v);
    }
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
    delete data.geometries;
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

/** bulb のワールド位置に、ワールド空間の下向きスポットライトを置いて点灯 */
function addLampLight(scene: SceneContext, lamp: SceneNode): void {
    const bulb = findDescendant(lamp, "bulb");
    if (!bulb) {
        return;
    }
    const w = bulb.worldMatrix; // 列3 = ワールド位置
    const lampLight = createSpotLight([w[12]!, w[13]!, w[14]!], [0, -1, 0], 0.8 * Math.PI, 0.01, 1);
    lampLight.diffuse = [1, 1, 0]; // Color3.Yellow()
    addToScene(scene, lampLight);
}

/** 昼夜スライダー（HTML <input type="range">）を canvas に重ねて作る */
function createDayNightSlider(canvas: HTMLCanvasElement, hemiLight: HemisphericLight): void {
    // canvas の親を基準に絶対配置でオーバーレイ
    const parent = canvas.parentElement ?? document.body;
    if (getComputedStyle(parent).position === "static") {
        parent.style.position = "relative";
    }

    const container = document.createElement("div");
    container.style.cssText =
        "position:absolute; right:16px; bottom:25px; width:220px; display:flex; flex-direction:column; " +
        "align-items:center; gap:6px; font-family:sans-serif; pointer-events:auto; z-index:10;";

    const header = document.createElement("div");
    header.textContent = "Night to Day";
    header.style.cssText = "color:white; height:30px; line-height:30px; text-shadow:0 0 3px black;";

    const slider = document.createElement("input");
    slider.type = "range";
    slider.min = "0";
    slider.max = "1";
    slider.step = "0.01";
    slider.value = "1";
    slider.style.cssText = "width:200px; height:20px; accent-color:gray;";
    slider.addEventListener("input", () => {
        hemiLight.intensity = Number(slider.value); // 本家 onValueChangedObservable 相当
        // intensity は単純プロパティで dirty 通知しない（＝ライト UBO が再アップロード
        // されず変化が反映されない）。direction.set は同値でも onDirty を発火するので、
        // これで _lightVersion を増やしてライトの再アップロードを強制する。
        const d = hemiLight.direction;
        d.set(d.x, d.y, d.z);
    });

    container.appendChild(header);
    container.appendChild(slider);
    parent.appendChild(container);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);

    const camera = createArcRotateCamera(-Math.PI / 2.2, Math.PI / 2.2, 15, { x: 0, y: 0, z: 0 });
    camera.upperBetaLimit = Math.PI / 2.2;
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // 半球光（初期 intensity 1＝昼）。方向は本家 (1, 1, 0)
    const hemiLight = createHemisphericLight([1, 1, 0], 1);
    addToScene(scene, hemiLight);

    // 昼夜スライダー（HTML オーバーレイ）
    createDayNightSlider(canvas, hemiLight);

    // 街灯・村・スカイボックスを並列ロード
    const [lampContainer, village] = await Promise.all([
        loadLampInlined(engine, LAMP_URL),
        loadGltf(engine, VALLEY_VILLAGE_URL),
        loadSkybox(scene, SKYBOX_BASE_URL, SKYBOX_EXT, 150),
    ]);
    addToScene(scene, village);

    // 元の街灯を配置＋点灯
    const lamp = findInContainer(lampContainer, "lamp");
    lamp.position.set(2, 0, 2);
    lamp.rotation.set(0, -Math.PI / 4, 0);
    addToScene(scene, lampContainer);
    addLampLight(scene, lamp);

    // clone で街灯を増やす
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

## 動作サンプル

<iframe src="https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/2?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 7-02 昼から夜へ（昼夜スライダー）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/QS4ZRZ/v/2
>
> 右下のスライダーで半球光の `intensity` を変えて昼夜を切り替えます。GUI ライブラリを持たない Lite では **HTML の `<input type="range">` を canvas に重ねる**のが定石で、`intensity` 変更を反映させるために **`direction.set(同値)` で dirty 化**するのがハマりどころです。本家の GUI を使わない代替のため、判定は △ です。

---

← [7-01 ライトを灯す (Light the Night)](./7-01-lights.md) ・ [7-03 影を追加 (Adding Shadows)](./7-03-shadows.md) →
