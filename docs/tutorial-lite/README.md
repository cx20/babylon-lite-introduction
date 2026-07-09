# Babylon.js チュートリアル → Babylon Lite 章別サンプル集

[Babylon.js チュートリアル日本語版（chomado）](https://zenn.dev/chomado/books/babylonjs-tutorial-ja) の各章を **Babylon Lite（WebGPU 専用エンジン）** で再現するサンプル集です。
最終的に「**Xbot が村の道を歩く三人称シーン**」（[ゴール完成版](./99-goal-final.md)）へ到達します。

対応可否の全体像は [tutorial-compatibility.md](../tutorial-compatibility.md) を参照してください。

## 目次

| パート | 内容 | ファイル |
|---|---|---|
| 第1部 | 基礎（Hello World / モデル読込 / セットアップ） | [01-basics.md](./01-basics.md) |
| 第2部 | 村の構築（地面 / メッシュ / テクスチャ / コピー） | [02-village.md](./02-village.md) |
| 第3部 | アニメーション（親子 / 車 / キャラ歩行） | [03-animation.md](./03-animation.md) |
| 第4部 | 衝突回避 | [04-collisions.md](./04-collisions.md) |
| 第5部 | 環境（丘 / 空 / スプライトの木） | [05-environment.md](./05-environment.md) |
| 第6部 | パーティクル効果 | [06-particles.md](./06-particles.md) |
| ゴール | 完成版（村を歩く Xbot） | [99-goal-final.md](./99-goal-final.md) |

## 動作環境

- **WebGPU 対応ブラウザ**（Chrome / Edge 113+ など）
- 実行は [Babylon Lite Playground](https://liteplayground.babylonjs.com) が最速（インストール不要）
- アセットは Playground 相対パスでは解決しないため、すべて `https://` 絶対 URL で参照します

> 各章のサンプルには、[Lite Playground の embed 機能](https://doc.babylonjs.com/lite/04-playground)（`?embed=runner`）による
> **実行プレビューを埋め込んでいます**。プレビューが見えるのは
> [GitHub Pages 版](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/README.html) だけです
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

> ⚠️ 注記：一部の API（ハイトマップ生成の引数、faceUV、スプライトのフィールド名など）は
> Lite のバージョンで差異が出る可能性があります。「要確認」と付いた箇所は、
> 型エラーが出たら IntelliSense（Playground）で正しい名称に合わせてください。
