# 2-03 メッシュを設置 (Place and Scale) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：メッシュの位置を調整して地面の上に載せる。`createBox` は既定で**中心が原点**に生成されるため、
`position.y` を指定しないと**半分が地面にめり込みます**。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createGround, createStandardMaterial`

> ⚠️ **`createBox` の第 2 引数は数値**（一辺の長さ）です。Babylon.js の `MeshBuilder.CreateBox("box", {})` に
> 引きずられて `createBox(engine, { size: 1 })` のようにオプションオブジェクトを渡すと、
> 頂点が `NaN` 化して**無言で描画されなくなります**。既定値（1）でよければ `createBox(engine)` でも構いません。

## A. 位置を指定しない場合（めり込む）

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

// box の原点は中心。position.y を指定しないと高さ1の半分（0.5）が地面に埋まる
const box = createBox(engine, 1);
box.material = createStandardMaterial();   // ★マテリアル必須（無いと描画されない）
addToScene(scene, box);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/2?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-03 メッシュを設置 / A. 位置を指定しない場合（めり込む）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/2

## B. 地面の上に載せる（`position.y` を指定）

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 3, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

const box = createBox(engine, 1);
box.material = createStandardMaterial();
box.position.y = 0.5;   // 高さの半分だけ上げて地面に載せる
addToScene(scene, box);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-03 メッシュを設置 / B. 地面の上に載せる（position.y を指定）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/3
>
> Babylon.js と同じく **原点はメッシュの中心**。`position` は `ObservableVec3` ですが、
> `box.position.y = 0.5` のように成分へ直接代入できます（[README の落とし穴](../README.md#重要な落とし穴共通)）。

## C. 横に伸ばす（スケール）

```typescript
box.position.x = 2;
box.scaling.x = 2;      // 横に伸ばす
```

---

← [2-02 サウンドを追加](./2-02-add-sound.md) ・ [2-04 基本的な家 (A Basic House)](./2-04-basic-house.md) →
