# 2-01 地面 (Grounding) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：宙に浮いた box の下に地面を敷き、box を「地面に建つ建物」として載せる。
`createBox` も `createGround` も**中心が原点**に生成されるため、`position.y` を指定しないと
box は**半分が地面にめり込みます**。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createGround, createStandardMaterial`

> ⚠️ **`createBox` の第 2 引数は数値**（一辺の長さ）です。Babylon.js の `MeshBuilder.CreateBox("box", {})` に
> 引きずられて `createBox(engine, { size: 1 })` のようにオプションオブジェクトを渡すと、
> 頂点が `NaN` 化して**無言で描画されなくなります**。既定値（1）でよければ `createBox(engine)` でも構いません。

## A. 地面を敷く（box がめり込む）

`createGround` の `width` は x 方向、`height` は z 方向の大きさです（y が垂直なので直感的には `depth` と呼びたいところですが、本家に合わせています）。

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
        title="Babylon Lite Playground: 2-01 地面 / A. 地面を敷く（box がめり込む）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/2

## B. 地面の上に載せる（`position.y` を指定）

高さの半分だけ持ち上げれば、box の底面が地面と一致します。

```typescript
box.position.y = 0.5;   // 既定サイズ（1）で作った box なので高さの半分は 0.5
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/3?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-01 地面 / B. 地面の上に載せる（position.y を指定）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/3
>
> Babylon.js と同じく **原点はメッシュの中心**。`position` は `ObservableVec3` ですが、
> `box.position.y = 0.5` のように成分へ直接代入できます（[README の落とし穴](../README.md#重要な落とし穴共通)）。

サイズ・位置・回転をまとめて扱う方法は [2-03 メッシュを設置](./2-03-place-and-scale.md) で、
地面の色やテクスチャは [2-05 テクスチャを貼る](./2-05-add-texture.md) で扱います。

## 補足：地面にテクスチャを貼る（本家 2-01 にはない追加例）

本家は地面を素のまま進め、[2-05](./2-05-add-texture.md) で緑の単色（`diffuseColor = [0, 1, 0]`）にします。
草テクスチャを貼りたい場合は `loadTexture2D`（async。`registerScene` 前に `await` し終えること）を使います。

追加 import：`loadTexture2D`

```typescript
const ground = createGround(engine, { width: 24, height: 24 });
const groundMat = createStandardMaterial();
groundMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/grass.png");
ground.material = groundMat;
addToScene(scene, ground);
```

---

← [2-00 村を作る](./2-00-build-village.md) ・ [2-02 サウンドを追加](./2-02-add-sound.md) →
