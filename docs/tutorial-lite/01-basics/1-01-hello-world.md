# 1-01 ハローワールド！最初のシーンとモデル — ○

> [第1部：基礎](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：シーン・カメラ・ライト・球・地面を作る。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createSphere, createGround, createStandardMaterial`

```typescript
// カメラ（キャラ後方視点に合わせて alpha = -π/2）
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 4, { x: 0, y: 1, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);

// ライト
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));

// 球 —— ★マテリアルの明示割り当てが必須（無いと描画されない）
const sphere = createSphere(engine, { diameter: 2, segments: 16 });
sphere.position.y = 1;
const sphereMat = createStandardMaterial();
sphereMat.diffuseColor = [0.8, 0.8, 0.8];   // 色は [r, g, b] 配列
sphere.material = sphereMat;
addToScene(scene, sphere);

// 地面 —— 同様にマテリアルを割り当て
const ground = createGround(engine, { width: 6, height: 6 });
const groundMat = createStandardMaterial();
groundMat.diffuseColor = [0.5, 0.55, 0.5];
ground.material = groundMat;
addToScene(scene, ground);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 1-01 ハローワールド！最初のシーンとモデル"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> ⚠️ **マテリアル必須**：Lite にはデフォルトマテリアルが無く、`material` 未設定のメッシュは
> `addToScene` されても**描画されません（エラーも出ません）**。プリミティブには必ず
> `createStandardMaterial()` / `createPbrMaterial()` を割り当ててください（→ [README の落とし穴](../README.md#重要な落とし穴共通)）。
>
> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/0
>
> `position` などは `ObservableVec3`。**成分（`.x/.y/.z`）で代入**します（丸ごと別オブジェクトを代入すると dirty 追跡が切れます）。

---

← [1-00 最初 (Firsts)](./1-00-firsts.md) ・ [1-02 あなたの web サイトをかっこよくしよう](./1-02-website.md) →
