# 2-03 メッシュを設置 (Place and Scale) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：メッシュの**サイズ・位置・向き**を変え、建物としてのバリエーションを出す。
地面に載せる（`position.y` で半分持ち上げる）話は [2-01 地面](./2-01-grounding.md) で扱っています。

追加 import：`createArcRotateCamera, attachControl, createHemisphericLight, createBox, createGround, createStandardMaterial`

## A. サイズ（`scaling`）

本家は生成時にオプションで寸法を指定できます。

```javascript
// Babylon.js
const box = BABYLON.MeshBuilder.CreateBox("box", { width: 2, height: 1.5, depth: 3 });
```

> ⚠️ **Lite の `createBox` は第 2 引数が数値**（一辺の長さ）で、`width` / `height` / `depth` を受け付けません。
> オプションオブジェクトを渡すと頂点が `NaN` 化して**無言で描画されなくなります**。
> 寸法違いの box は、単位立方体を作って **`scaling` で伸縮**させて作ります（本家も同じ結果になる書き方を併記しています）。

```typescript
const box = createBox(engine, 1);        // 単位立方体
box.scaling.x = 2;
box.scaling.y = 1.5;
box.scaling.z = 3;
```

`scaling` は `position` と同じく x / y / z を持つベクトル（`ObservableVec3`）で、成分に直接代入できます。

## B. 位置（`position`）

`position` はメッシュの**中心**を置く座標です。高さ 1.5 の box を地面に載せるには `position.y = 0.75`（高さの半分）にします。

```typescript
const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 10, { x: 0, y: 0, z: 0 });
scene.camera = camera;
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([1, 1, 0], 1.0));

const ground = createGround(engine, { width: 10, height: 10 });
ground.material = createStandardMaterial();
addToScene(scene, ground);

// 高さ 1.5 の box を 3 つ、間隔をあけて地面に並べる
for (const x of [-3, 0, 3]) {
  const box = createBox(engine, 1);
  box.material = createStandardMaterial();   // ★マテリアル必須（無いと描画されない）
  box.scaling.x = 2;
  box.scaling.y = 1.5;
  box.scaling.z = 3;
  box.position.x = x;
  box.position.y = 0.75;                     // 高さ 1.5 の半分だけ持ち上げて接地
  addToScene(scene, box);
}
```

## C. 向き（`rotation`）

回転は**ラジアン**で指定します。本家は全 3 軸の同時回転を避け、1 軸だけ扱う方針なので、ここでも y 軸まわりだけにします。

```typescript
box.rotation.y = Math.PI / 4;   // 45 度
```

> Lite の内部姿勢は `rotationQuaternion` ですが、`rotation`（オイラー角）は
> そこへの代入プロキシとして使えます（[2-04](./2-04-basic-house.md) の屋根も `rotation.z` で倒しています）。
> 本家の `BABYLON.Tools.ToRadians(45)` に相当するヘルパーは無いので、`度 * Math.PI / 180` で変換してください。

> ℹ️ この章は Playground の実行プレビュー未添付です（サイズ・位置・回転はいずれも
> [2-04 基本的な家](./2-04-basic-house.md) 以降のサンプルで実際に使っています）。

---

← [2-02 サウンドを追加](./2-02-add-sound.md) ・ [2-04 基本的な家 (A Basic House)](./2-04-basic-house.md) →
