# 2-03 メッシュを設置 (Place and Scale) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：メッシュの**サイズ・位置・向き**を変え、建物としてのバリエーションを出す。
地面に載せる（`position.y` で半分持ち上げる）話は [2-01 地面](./2-01-grounding.md) で扱っています。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createGround, createStandardMaterial`

## A. サイズ（`scaling`）

本家は「**同じ大きさ (2, 1.5, 3) の箱を 3 通りの書き方で作る**」というサンプルです。

```javascript
// Babylon.js
const box1 = BABYLON.MeshBuilder.CreateBox("box", { width: 2, height: 1.5, depth: 3 });  // 生成時にサイズ指定

const box2 = BABYLON.MeshBuilder.CreateBox("box", {});   // 単位キューブ
box2.scaling.x = 2; box2.scaling.y = 1.5; box2.scaling.z = 3;   // 成分ごとに代入

const box3 = BABYLON.MeshBuilder.CreateBox("box", {});   // 単位キューブ
box3.scaling = new BABYLON.Vector3(2, 1.5, 3);           // ベクトルを一括代入
```

> ⚠️ **Lite の `createBox` は第 2 引数が一辺の長さ（数値）**で、`createBox(engine, size = 1)` というシグネチャです。
> `{ width, height, depth }` のような**非等方サイズは生成時に指定できません**
> （オプションオブジェクトを渡すと頂点が `NaN` 化し、**無言で描画されなくなります**）。
> つまり本家 `box1` の書き方は Lite では使えず、**3 通りとも「単位キューブ + スケーリング」に集約**されます。
> 結果の形状は本家と同じ (2, 1.5, 3) になります。
>
> `createGround` / `createSphere` など他のファクトリは**オプションオブジェクトを取る**ので、
> **`createBox` だけが非対称**である点に注意してください。どうしても生成時にサイズを焼き込みたい場合は、
> `createMeshFromData` で頂点を自作します（[2-06](./2-06-materials-faceuv.md) の `faceUV` で使った手法）。

Lite では `scaling` / `position` は `ObservableVec3` で、**`.set(x, y, z)` の一括指定と、`.x` / `.y` / `.z` の個別代入**のどちらも使えます。
本家のように**新しいベクトルオブジェクトを代入する形ではありません**。

```typescript
const box = createBox(engine, 1);   // 単位キューブ
box.scaling.set(2, 1.5, 3);         // 一括指定
// box.scaling.x = 2; box.scaling.y = 1.5; box.scaling.z = 3;   // 個別代入でも同じ
```

## B. 位置（`position`）

`position` はメッシュの**中心**を置く座標です。高さ 1.5 の箱を地面に載せるには `position.y = 0.75`（高さの半分）にします。
本家と同じく、3 通りの書き方で作った同サイズの箱を並べます。

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1));

// box1: 本家は生成時サイズ指定だが、Lite では単位キューブ + スケーリングで同じ (2, 1.5, 3) を作る
const box1 = createBox(engine, 1);
box1.material = createStandardMaterial();   // ★マテリアル必須（無いと描画されない）
box1.scaling.set(2, 1.5, 3);
box1.position.y = 0.75;                     // 高さ 1.5 の箱を地面に載せる
addToScene(scene, box1);

// box2: scaling を成分ごとに代入
const box2 = createBox(engine, 1);
box2.material = createStandardMaterial();
box2.scaling.x = 2;
box2.scaling.y = 1.5;
box2.scaling.z = 3;
box2.position.set(-3, 0.75, 0);
addToScene(scene, box2);

// box3: scaling を一括設定（本家のベクトル代入に相当）
const box3 = createBox(engine, 1);
box3.material = createStandardMaterial();
box3.scaling.set(2, 1.5, 3);
box3.position.x = 3;
box3.position.y = 0.75;
box3.position.z = 0;
addToScene(scene, box3);

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/062TZ6/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-03 メッシュを設置（サイズ・位置）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/062TZ6/v/0
>
> 3 つとも同じ (2, 1.5, 3) の箱になります（本家と同じ結果）。

## C. 向き（`rotation`）

回転は**ラジアン**で指定します。本家は全 3 軸の同時回転を避け、1 軸だけ扱う方針なので、ここでも y 軸まわりだけにします。

```typescript
box.rotation.y = Math.PI / 4;   // 45 度
```

> Lite の内部姿勢は `rotationQuaternion` ですが、`rotation`（オイラー角）は
> そこへの代入プロキシとして使えます（[2-04](./2-04-basic-house.md) の屋根も `rotation.z` で倒しています）。
> 本家の `BABYLON.Tools.ToRadians(45)` に相当するヘルパーは無いので、`度 * Math.PI / 180` で変換してください。

---

← [2-02 サウンドを追加](./2-02-add-sound.md) ・ [2-04 基本的な家 (A Basic House)](./2-04-basic-house.md) →
