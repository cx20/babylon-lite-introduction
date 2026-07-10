# 3-05 車のアニメーション (Car Animation) — ○

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：車体を `X=-4` から `X=4` へ前進させながら、4 輪も同時に回す。本家の `Animation("position.x", 30fps)` / `beginAnimation` を、Lite の **Property Animation API** で置き換えます。

このステップでは、本家サンプルに近づけるため `car.babylon` を読み込みます。ただし現在の Lite ローダー向けに、読み込み前に `geometryId` 参照を各 mesh の頂点データへ展開し、テクスチャとアニメーションはコード側で設定します。

追加 import：`createAnimationManager, createPropertyAnimationClip, createPropertyAnimationGroup, getContainerMeshes, loadBabylon, onBeforeRender, updateAnimationManager`（＋型 `AnimationManager`）

```typescript
interface CarParts {
  car: Mesh;
  wheels: readonly Mesh[];
}

function findMeshByName(meshes: readonly Mesh[], name: string): Mesh {
  const mesh = meshes.find((candidate) => candidate.name === name);
  if (!mesh) throw new Error(`メッシュ "${name}" が見つかりません。`);
  return mesh;
}

async function loadCar(engine: EngineContext, scene: SceneContext): Promise<CarParts> {
  const sourceUrl = "https://assets.babylonjs.com/meshes/car.babylon";
  const convertedUrl = await createConvertedBabylonUrl(sourceUrl); // geometryId を mesh 頂点データへ展開

  try {
    const container = await loadBabylon(engine, convertedUrl, {
      loadTextures: false,
      loadCamera: false,
    });
    addToScene(scene, container);

    const meshes = getContainerMeshes(container);
    const car = findMeshByName(meshes, "car");
    const wheelRB = findMeshByName(meshes, "wheelRB");
    const wheelRF = findMeshByName(meshes, "wheelRF");
    const wheelLB = findMeshByName(meshes, "wheelLB");
    const wheelLF = findMeshByName(meshes, "wheelLF");

    const carMaterial = createStandardMaterial();
    carMaterial.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/car.png");
    car.material = carMaterial;

    const wheelMaterial = createStandardMaterial();
    wheelMaterial.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/textures/wheelcar.png");
    for (const wheel of [wheelRB, wheelRF, wheelLB, wheelLF]) wheel.material = wheelMaterial;

    return { car, wheels: [wheelRB, wheelRF, wheelLB, wheelLF] };
  } finally {
    URL.revokeObjectURL(convertedUrl);
  }
}

function createCarAnimation(manager: AnimationManager, car: Mesh): void {
  const carClip = createPropertyAnimationClip(
    "carAnimation",
    [{
      path: "position.x",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: -4 },
        { frame: 150, value: 4 },
      ],
    }],
    { frameRate: 30 },
  );

  createPropertyAnimationGroup(manager, car, carClip, {
    fromFrame: 0,
    toFrame: 150,
    loop: true,
  });
}

function createWheelAnimations(manager: AnimationManager, wheels: readonly Mesh[]): void {
  const wheelClip = createPropertyAnimationClip(
    "wheelAnimation",
    [{
      path: "rotation.y",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 0 },
        { frame: 30, value: 2 * Math.PI },
      ],
    }],
    { frameRate: 30 },
  );

  for (const wheel of wheels) {
    createPropertyAnimationGroup(manager, wheel, wheelClip, {
      fromFrame: 0,
      toFrame: 30,
      loop: true,
    });
  }
}

function setupAnimations(engine: EngineContext, scene: SceneContext, car: Mesh, wheels: readonly Mesh[]): void {
  const manager = createAnimationManager({ engine });

  createCarAnimation(manager, car);       // frame 0: x=-4 → frame 150: x=4（5秒）
  createWheelAnimations(manager, wheels); // frame 0: y=0 → frame 30: y=2π（1秒）

  onBeforeRender(scene, (deltaMs: number) => {
    updateAnimationManager(manager, deltaMs);
  });
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 2, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const { car, wheels } = await loadCar(engine, scene);
  setupAnimations(engine, scene, car, wheels);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/6?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-05 車のアニメーション"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/6
>
> 車体は `frame 0 = position.x: -4`、`frame 150 = position.x: 4`、`frameRate = 30` なので 5 秒で移動します。車輪は `frame 0 = rotation.y: 0`、`frame 30 = rotation.y: 2π` なので 1 秒で 1 回転します。`AnimationManager` は自動では進まないため、`onBeforeRender` で `updateAnimationManager(manager, deltaMs)` を呼びます。
>
> **補助関数**：`createConvertedBabylonUrl(sourceUrl)` は、`car.babylon` 内の `geometryId` が参照する `geometries.vertexData` を各 mesh の `positions` / `normals` / `uvs` / `indices` へ展開し、Blob URL として `loadBabylon` に渡します。完全版は Playground 参照。

## 村の中を走らせる

次は `village.glb` と `car.glb` を読み込み、村の道路上で車を走らせます。

> **訂正**：以前この節では「`car.glb` には車輪回転アニメーションが埋め込まれている」と説明していましたが、これは誤りでした。`car.glb` の glTF `animations` は空で、車輪回転は含まれていません。本家チュートリアルも *"car.glb has no built-in wheel animation, so we create one"* としてコード側で作成しています。そのため、**車輪回転もコード側で作成**し、車体の前進とあわせて `AnimationManager` に登録します。

- **車輪**：`rotation.y` を frame 0→30（1 秒）で `-2π` 回すクリップを 4 輪それぞれのノードへ割り当て、ループ再生します。
- **車体**：`position.z` のプロパティアニメーションで前進させます。

> **補足（`rotationQuaternion = null` は不要）**：本家では毎フレームの `rotation.y` 書き込みで劣化しないよう `wheel.rotationQuaternion = null` にしますが、Lite では不要です。Lite の `node.rotation` は quaternion 上の Euler プロキシで、書き込みは直接 quaternion に反映されます。プロキシは Euler 状態をキャッシュし、外部で quaternion が変更されない限り再抽出しないため、毎フレーム書き込んでもジンバル問題は起きません（`car.glb` の車輪ノードはレスト回転なしのため、初期姿勢も本家の null 化後と一致します）。

追加 import：`loadGltf`（＋型 `AssetContainer, PropertyAnimationClip, SceneNode`）

```typescript
function findNodeByName(container: AssetContainer, name: string): SceneNode {
  const visited = new Set<SceneNode>();

  const visit = (node: SceneNode): SceneNode | null => {
    if (visited.has(node)) return null;
    visited.add(node);
    if (node.name === name) return node;

    for (const child of node.children ?? []) {
      const found = visit(child as SceneNode);
      if (found) return found;
    }
    return null;
  };

  for (const entity of container.entities) {
    if (!entity || typeof entity !== "object" || !("name" in entity) || !("children" in entity)) continue;
    const found = visit(entity as SceneNode);
    if (found) return found;
  }

  throw new Error(`ノード "${name}" が見つかりません。`);
}

function addContainerEntitiesToScene(scene: SceneContext, container: AssetContainer): void {
  for (const entity of container.entities) {
    addToScene(scene, entity);
  }
}

// car.glb には車輪アニメーションが埋め込まれていないため、
// 本家チュートリアルと同様にコード側で作成する（4 輪でクリップを共有）。
function addWheelAnimations(manager: AnimationManager, container: AssetContainer): void {
  const wheelClip: PropertyAnimationClip = createPropertyAnimationClip(
    "wheelAnimation",
    [{
      path: "rotation.y",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 0 },
        { frame: 30, value: -2 * Math.PI },
      ],
    }],
    { frameRate: 30 },
  );

  const wheelNames = ["wheelRB", "wheelRF", "wheelLB", "wheelLF"] as const;
  for (const name of wheelNames) {
    const wheel = findNodeByName(container, name);
    createPropertyAnimationGroup(manager, wheel, wheelClip, {
      fromFrame: 0,
      toFrame: 30,
      loop: true,
      speedRatio: 1,
    });
  }
}

function addCarMovementAnimation(manager: AnimationManager, car: SceneNode): void {
  const carMoveClip = createPropertyAnimationClip(
    "carMovement",
    [{
      path: "position.z",
      frameRate: 30,
      interpolation: "linear",
      keys: [
        { frame: 0, value: 8 },
        { frame: 150, value: -7 },
        { frame: 200, value: -7 },
      ],
    }],
    { frameRate: 30 },
  );

  createPropertyAnimationGroup(manager, car, carMoveClip, {
    fromFrame: 0,
    toFrame: 200,
    loop: true,
    speedRatio: 1,
  });
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 15, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  const [villageContainer, carContainer] = await Promise.all([
    loadGltf(engine, "https://assets.babylonjs.com/meshes/village.glb"),
    loadGltf(engine, "https://assets.babylonjs.com/meshes/car.glb"),
  ]);

  addContainerEntitiesToScene(scene, villageContainer);
  addContainerEntitiesToScene(scene, carContainer);

  const car = findNodeByName(carContainer, "car");
  car.rotation.set(Math.PI / 2, 0, -Math.PI / 2);
  car.position.set(-3, 0.16, 8);

  const animationManager = createAnimationManager({ engine });
  addWheelAnimations(animationManager, carContainer);  // 4 輪の回転（コード側で作成）
  addCarMovementAnimation(animationManager, car);      // 村の中を Z 方向へ走行

  onBeforeRender(scene, (deltaMs: number) => {
    updateAnimationManager(animationManager, deltaMs);
  });

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/8?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-05 車のアニメーション / 村の中を走らせる"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/8
>
> `car.glb` の描画メッシュ名は `gltf_mesh_0` などになるため、`getContainerMeshes()` ではなく `AssetContainer.entities` のノード階層から glTF ノード名 `"car"` / `"wheelRB"` 等を探します。車輪は `frame 0 = rotation.y: 0`、`frame 30 = -2π` なので 1 秒で 1 回転します。車体移動は `frame 0 = position.z: 8`、`frame 150 = -7`、`frame 200 = -7` で、5 秒走行して約 1.67 秒停止してからループします。

---

← [3-04 車輪のアニメーション (Wheel Animation)](./3-04-wheel-animation.md) ・ [3-06 キャラクターのアニメーション (Character Animation)](./3-06-character-animation.md) →
