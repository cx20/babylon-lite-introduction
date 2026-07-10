# 3-04 車輪のアニメーション (Wheel Animation) — ○

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：まず 1 つの車輪だけを作り、画像付きタイヤが回転することを確認する。本家の `Animation("rotation.y", 30fps)` と `scene.beginAnimation(..., true)` を、Lite では `requestAnimationFrame` と `performance.now()` で置き換えます。

本家サンプルの要点は、`frame 0 = rotation.y: 0`、`frame 30 = rotation.y: 2π`、`fps = 30` なので **1 秒で 1 回転**することです。

前章の `FaceUV` / `remapFaceUV` / `createCylinderY` をそのまま使い、まず `wheelRB` 1 本だけを生成します。

```typescript
function animateWheelRotationY(wheel: Mesh, durationSeconds = 1): void {
  const startTime = performance.now();

  const update = (currentTime: number): void => {
    const elapsedSeconds = (currentTime - startTime) / 1000;
    const progress = (elapsedSeconds / durationSeconds) % 1;

    // frame 0 の 0 から frame 30 の 2π までを線形補間
    wheel.rotation.y = progress * 2 * Math.PI;

    requestAnimationFrame(update);
  };

  requestAnimationFrame(update);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  scene.clearColor = [1, 1, 1, 1]; // 元サンプルの白背景に合わせる

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 1.5, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));

  const wheelUV: FaceUV[] = [
    [0, 0, 1, 1],     // 下面キャップ：wheel.png 全体
    [0, 0.5, 0, 0.5], // 円柱側面：同一点を参照し、タイヤ外周を単色化
    [0, 0, 1, 1],     // 上面キャップ：wheel.png 全体
  ];

  const wheelMat = createStandardMaterial();
  wheelMat.diffuseTexture = await loadTexture2D(engine, "https://assets.babylonjs.com/environments/wheel.png");

  const wheelRB = createCylinderY(engine, "wheelRB", {
    diameter: 0.125,
    height: 0.05,
    tessellation: 32,
    faceUV: wheelUV,
  });

  wheelRB.material = wheelMat;
  addToScene(scene, wheelRB);

  // frame 0 = 0、frame 30 = 2π、30fps と同じく 1 秒で 1 回転
  animateWheelRotationY(wheelRB, 1);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/4?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-04 車輪のアニメーション"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/4
>
> `requestAnimationFrame` の時刻から角度を計算するので、30fps / 60fps / 120fps のどれでも 1 秒あたり 1 回転になります。本家の `rotation.y` アニメーションと同じく、この段階では車体へ取り付けず、単体の車輪で動きを確認します。

## 他の車輪も回す

次は車体と 4 輪を生成し、`wheelRB` / `wheelRF` / `wheelLB` / `wheelLF` を同じ回転角で同期させます。本家は `scene.beginAnimation(...)` を 4 回呼びますが、Lite では 1 つの `requestAnimationFrame` 更新で `wheels` 配列をまとめて回します。

```typescript
interface CarParts {
  car: Mesh;
  wheels: Mesh[];
}

async function buildCar(engine: EngineContext, scene: SceneContext): Promise<CarParts> {
  // 3-03 と同じ car.png 付き車体を生成
  const car = extrudePolygonXoZ(engine, "car", {
    shape: outline,
    depth: 0.2,
    faceUV: carFaceUV,
    wrap: true,
  });
  car.material = carMat;
  addToScene(scene, car);

  const wheels: Mesh[] = [];

  const createWheel = (name: string, x: number, y: number, z: number): Mesh => {
    const wheel = createCylinderY(engine, name, {
      diameter: 0.125,
      height: 0.05,
      tessellation: 32,
      faceUV: wheelFaceUV,
    });

    wheel.material = wheelMat;
    wheel.parent = car;
    wheel.position.x = x;
    wheel.position.y = y;
    wheel.position.z = z;
    addToScene(scene, wheel);
    wheels.push(wheel);

    return wheel;
  };

  createWheel("wheelRB", -0.2, 0.035, -0.1);
  createWheel("wheelRF", 0.1, 0.035, -0.1);
  createWheel("wheelLB", -0.2, -0.2 - 0.035, -0.1);
  createWheel("wheelLF", 0.1, -0.2 - 0.035, -0.1);

  car.rotation.x = -Math.PI / 2;

  return { car, wheels };
}

function animateWheelsRotationY(wheels: readonly Mesh[], durationSeconds = 1): void {
  const startTime = performance.now();

  const update = (currentTime: number): void => {
    const elapsedSeconds = (currentTime - startTime) / 1000;
    const progress = (elapsedSeconds / durationSeconds) % 1;
    const rotationY = progress * 2 * Math.PI;

    for (const wheel of wheels) {
      wheel.rotation.y = rotationY;
    }

    requestAnimationFrame(update);
  };

  requestAnimationFrame(update);
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);
  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 2, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);

  addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

  const { wheels } = await buildCar(engine, scene);
  animateWheelsRotationY(wheels, 1);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/BY4KF1/v/5?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-04 車輪のアニメーション / 他の車輪も回す"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/BY4KF1/v/5
>
> 4 輪に別々の `requestAnimationFrame` を設定せず、1 つの更新処理で同じ `rotation.y` を設定します。これで本家の `beginAnimation(wheelRB/RF/LB/LF, 0, 30, true)` と同じく、全タイヤが 1 秒 1 回転で同期します。

---

← [3-03 車のマテリアル (Car Material)](./3-03-car-material.md) ・ [3-05 車のアニメーション (Car Animation)](./3-05-car-animation.md) →
