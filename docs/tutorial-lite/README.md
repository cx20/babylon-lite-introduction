# Babylon.js チュートリアル → Babylon Lite 章別サンプル集

[Babylon.js チュートリアル日本語版（chomado）](https://zenn.dev/chomado/books/babylonjs-tutorial-ja) の各章を **Babylon Lite（WebGPU 専用エンジン）** で再現するサンプル集です。
村の地面・家・車・歩くキャラクター（Xbot）まで、章を追って組み上げていきます。

対応可否の全体像は [tutorial-compatibility.md](../tutorial-compatibility.md) を参照してください。

## 目次

### [第1部：基礎](./01-basics/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 1-00 | [最初 (Firsts)](./01-basics/1-00-firsts.md) | 導入 |
| 1-01 | [ハローワールド！最初のシーンとモデル](./01-basics/1-01-hello-world.md) | ○ |
| 1-02 | [あなたの web サイトをかっこよくしよう](./01-basics/1-02-website.md) | △ |
| 1-03 | [モデルを扱う](./01-basics/1-03-models.md) | ○ |
| 1-04 | [最初の web アプリのセットアップ](./01-basics/1-04-webapp-setup.md) | ○ |

### [第2部：村の構築](./02-village/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 2-00 | [村を作る](./02-village/2-00-build-village.md) | 導入 |
| 2-01 | [地面 (Grounding)](./02-village/2-01-grounding.md) | ○ |
| 2-02 | [サウンドを追加](./02-village/2-02-add-sound.md) | ○ |
| 2-03 | [メッシュを設置 (Place and Scale)](./02-village/2-03-place-and-scale.md) | ○ |
| 2-04 | [基本的な家 (A Basic House)](./02-village/2-04-basic-house.md) | ○ |
| 2-05 | [テクスチャを貼る (Add Texture)](./02-village/2-05-add-texture.md) | ○ |
| 2-06 | [マテリアル（面ごと/faceUV）](./02-village/2-06-materials-faceuv.md) | △ |
| 2-07 | [リファクタリング（関数化）](./02-village/2-07-refactoring.md) | ○ |
| 2-08 | [メッシュを結合 (Combining Meshes)](./02-village/2-08-combining-meshes.md) | △ |
| 2-09 | [メッシュをコピー (Copying Meshes)](./02-village/2-09-copying-meshes.md) | ○ |
| 2-10 | [Viewer のカメラ変更](./02-village/2-10-viewer-camera.md) | ○ |
| 2-11 | [Web アプリ レイアウト](./02-village/2-11-webapp-layout.md) | ○ |

### [第3部：アニメーション](./03-animation/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 3-00 | [村のアニメーション](./03-animation/3-00-village-animation.md) | 導入 |
| 3-01 | [メッシュの親子関係 (Mesh Parents)](./03-animation/3-01-mesh-parents.md) | ○ |
| 3-02 | [車を組み立てる (Building the Car)](./03-animation/3-02-building-the-car.md) | ○ |
| 3-03 | [車のマテリアル (Car Material)](./03-animation/3-03-car-material.md) | ○ |
| 3-04 | [車輪のアニメーション (Wheel Animation)](./03-animation/3-04-wheel-animation.md) | ○ |
| 3-05 | [車のアニメーション (Car Animation)](./03-animation/3-05-car-animation.md) | ○ |
| 3-06 | [キャラクターのアニメーション (Character Animation)](./03-animation/3-06-character-animation.md) | ○（アセットは glTF） |
| 3-07 | [村を歩き回る (A Walk Around The Village)](./03-animation/3-07-walk-around-village.md) | △ |

### [第4部：衝突回避](./04-collisions/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 4-00 | [コリジョン回避 (Avoiding Collisions)](./04-collisions/4-00-avoiding-collisions.md) | 導入 |
| 4-01 | [車の衝突事故を回避する (Avoiding a Car Crash)](./04-collisions/4-01-car-crash.md) | △ |

### [第5部：環境改善](./05-environment/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 5-00 | [より良い環境に (A Better Environment)](./05-environment/5-00-better-environment.md) | 導入 |
| 5-01 | [遠くの丘 (Distant Hills)](./05-environment/5-01-distant-hills.md) | ○ |
| 5-02 | [頭上の空 (Skies Above)](./05-environment/5-02-skies-above.md) | ○（要アセット） |
| 5-03 | [木のスプライト (Sprite Trees)](./05-environment/5-03-sprite-trees.md) | △（部分対応） |

### [第6部：パーティクル効果](./06-particles/README.md)

| 章 | タイトル | 判定 |
|---|---|:--:|
| 6-00 | [パーティクル噴水 (Build a Particle Fountain)](./06-particles/6-00-particle-fountain.md) | ✕ |
| 6-01 | [旋盤で回された噴水 (A Lathe Turned Fountain)](./06-particles/6-01-lathe-fountain.md) | ✕ |

## 動作環境

- **WebGPU 対応ブラウザ**（Chrome / Edge 113+ など）
- 実行は [Babylon Lite Playground](https://liteplayground.babylonjs.com) が最速（インストール不要）
- アセットは Playground 相対パスでは解決しないため、すべて `https://` 絶対 URL で参照します

> 各章のサンプルには、[Lite Playground の embed 機能](https://doc.babylonjs.com/lite/04-playground)（`?embed=runner`）による
> **実行プレビューを埋め込んでいます**。プレビューが見えるのは
> [GitHub Pages 版](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/) だけです
> — GitHub のリポジトリ画面では `<iframe>` がサニタイズされて消えるため、併記したスニペットのリンクから開いてください。

## 共通テンプレート

各章のサンプルは、次のテンプレートの `★ ここに各章のコード ★` の位置に貼り付ける前提です。
`import` は章ごとに「追加 import」を足してください。

```typescript
import {
  createEngine, createSceneContext,
  addToScene, registerScene, startEngine,
  // ↓ 各章の「追加 import」をここに足す
} from "@babylonjs/lite";

async function main(): Promise<void> {
  const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement;
  const engine = await createEngine(canvas);
  const scene  = createSceneContext(engine);

  // ★ ここに各章のコード ★

  await registerScene(scene);
  await startEngine(engine);
}

main().catch((err) => console.error(err));
```

## 重要な落とし穴（共通）

> ⚠️ **メッシュには必ずマテリアルを割り当てる。**
> Lite には**デフォルトマテリアルが存在しません**。`material` 未設定のメッシュは
> `addToScene` に登録はされても **renderable が生成されず、一切描画されません（エラーも出ません）**。
> プリミティブ（`createBox` / `createSphere` / `createGround` など）を作ったら、
> 必ず `createStandardMaterial()` か `createPbrMaterial()` を割り当ててください。
>
> ```typescript
> const sphere = createSphere(engine, { diameter: 2 });
> const mat = createStandardMaterial();
> mat.diffuseColor = [0.8, 0.8, 0.8]; // 色は [r, g, b] 配列
> sphere.material = mat;              // ← これが無いと表示されない
> addToScene(scene, sphere);
> ```
>
> `loadGltf` で読み込んだモデルは glTF 側のマテリアルが付くため、この対応は不要です。

## 凡例

- **○** … Lite の標準機能で再現可能
- **△** … 代替・変換・自作、または部分対応（⚡）
- **✕** … Lite 未対応（—／🚫）。目的は達成不可
- **—** … 導入・概念章（コードなし）

## Babylon.js → Babylon Lite 主な対応

| Babylon.js | Babylon Lite |
|---|---|
| `new WebGPUEngine` + `initAsync` | `await createEngine(canvas)` |
| `new Scene(engine)` | `createSceneContext(engine)` |
| `engine.runRenderLoop(...)` | `await startEngine(engine)` |
| `new ArcRotateCamera(...)` | `createArcRotateCamera(α, β, r, target)` |
| `camera.attachControl` | `attachControl(camera, canvas, scene)` |
| `new HemisphericLight(...)` | `createHemisphericLight([x,y,z], intensity)` |
| `MeshBuilder.CreateBox/Sphere/Ground` | `createBox/createSphere/createGround(engine, opts)` |
| `new StandardMaterial` / `PBRMaterial` | `createStandardMaterial()` / `createPbrMaterial()` |
| `new Texture(url)` | `await loadTexture2D(engine, url)` |
| `SceneLoader.ImportMeshAsync`（.glb） | `addToScene(scene, await loadGltf(engine, url))` |
| `SceneLoader.ImportMeshAsync`（.babylon） | `addToScene(scene, await loadBabylon(engine, url))`（静的モデル） |
| `mesh.createInstance()` | `setThinInstances(mesh, matrices, count)` |
| `scene.onBeforeRenderObservable.add` | `onBeforeRender(scene, cb)` |
| `AnimationGroup.play()` / `.stop()` | `playAnimation(group)` / `stopAnimation(group)` |
| `mesh.dispose()` | `removeFromScene(scene, mesh)` |

> 注記：各章の API 名・シグネチャは Babylon Lite v1.8 の[リポジトリ](https://github.com/BabylonJS/Babylon-Lite)ソースで確認済みです。
> Lite は機能追加が続いているため、バージョンが上がって差異が出た場合は Playground の IntelliSense で正しい名称に合わせてください。
