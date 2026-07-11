# 2-06 マテリアル（面ごと/faceUV） — △

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：1 枚のテクスチャアトラス（横に並んだ正面/右/背面/左の 4 コマ）を、box の各側面に別々に貼る。
Babylon.js の `MeshBuilder.CreateBox({ faceUV, wrap: true })` に相当する機能は **`createBox` には無い**ことが確認できました
（`createBox(engine, size)` は数値の一辺長のみ。`faceUV` / `wrap` / `width` オプションは存在しません）。

## 技法：`createMeshFromData` で本家 box ビルダーを逐語移植する

Lite は公開 API に **`createMeshFromData(engine, name, positions, normals, indices, uvs)`** という、
頂点データを直接渡してメッシュを組み立てる低レベル関数を持っています。これを使い、Babylon.js 本家の
`boxBuilder`（`wrap: true` パス）の頂点配置・UV 計算式・インデックスをそのまま移植した `createWrappedBox` ヘルパーを自作します。

- **面番号**は本家と同じ：`0=+Z`（背面）`1=-Z`（正面）`2=+X`（右）`3=-X`（左）`4=+Y`（上）`5=-Y`（底）
- **`faceUV`** は本家 `Vector4(x, y, z, w)` の代わりに `[uMin, vMin, uMax, vMax]` の配列で表現
- 面ごとの UV は本家と同じ式（頂点順に `(z,w) (x,w) (x,y) (z,y)`）で 4 頂点に適用
- `width` はプリミティブオプションではなく**頂点座標に直接焼き込む**（非一様 scaling に頼らない）

これにより、Lite 組み込み box と同一の UV・巻き順規約を保ったまま、本家と見た目が一致する faceUV box を再現できます。

> 補足（v1.9 以降）：v1.9 で **`createBoxData` が公開エクスポート**されました。標準 box の頂点データを取得して UV だけ差し替える簡素化も可能になっていますが、本章の `createWrappedBox`（`createMeshFromData` に本家 boxBuilder の `wrap:true` ロジックを逐語移植）は面番号・巻き順の対応が明示的で分かりやすいため、そのまま採用します。

追加 import：`createCylinder, createGround, createStandardMaterial, createMeshFromData, loadTexture2D`

## A. 一軒家 (Detached House) — `cubehouse.png`

`cubehouse.png` は横 1 列に「正面/右/背面/左」の 4 コマが並んだアトラス。各面の U 範囲を `faceUV` で割り当てます。

```typescript
type FaceUV = readonly [number, number, number, number];   // [uMin, vMin, uMax, vMax]

function createWrappedBox(
  engine: EngineContext,
  options: { width?: number; height?: number; depth?: number; faceUV?: (FaceUV | undefined)[] } = {},
) {
  const w = (options.width ?? 1) / 2, h = (options.height ?? 1) / 2, d = (options.depth ?? 1) / 2;

  // prettier-ignore
  const base = [
    -1,  1,  1,   1,  1,  1,   1, -1,  1,  -1, -1,  1,   // 0: +Z（背面）
     1,  1, -1,  -1,  1, -1,  -1, -1, -1,   1, -1, -1,   // 1: -Z（正面）
     1,  1,  1,   1,  1, -1,   1, -1, -1,   1, -1,  1,   // 2: +X（右側面）
    -1,  1, -1,  -1,  1,  1,  -1, -1,  1,  -1, -1, -1,   // 3: -X（左側面）
     1,  1,  1,  -1,  1,  1,  -1,  1, -1,   1,  1, -1,   // 4: +Y（上面）
     1, -1, -1,  -1, -1, -1,  -1, -1,  1,   1, -1,  1,   // 5: -Y（底面）
  ];
  const positions = new Float32Array(72);
  for (let i = 0; i < 24; i++) {
    positions[i * 3 + 0] = base[i * 3 + 0]! * w;
    positions[i * 3 + 1] = base[i * 3 + 1]! * h;
    positions[i * 3 + 2] = base[i * 3 + 2]! * d;
  }

  const AXES = [[0,0,1],[0,0,-1],[1,0,0],[-1,0,0],[0,1,0],[0,-1,0]];
  const normals = new Float32Array(72);
  for (let f = 0; f < 6; f++) for (let v = 0; v < 4; v++) normals.set(AXES[f]!, (f * 4 + v) * 3);

  // faceUV 適用（本家と同じ式: 頂点順に (z,w) (x,w) (x,y) (z,y)）
  const uvs = new Float32Array(48);
  for (let f = 0; f < 6; f++) {
    const [x, y, z, wv] = options.faceUV?.[f] ?? [0, 0, 1, 1];
    uvs.set([z, wv, x, wv, x, y, z, y], f * 8);
  }

  // prettier-ignore
  const indices = new Uint32Array([
     2,  3,  0,   2,  0,  1,   // +Z
     4,  5,  6,   4,  6,  7,   // -Z
     9, 10, 11,   9, 11,  8,   // +X
    12, 14, 15,  12, 13, 14,   // -X
    17, 19, 16,  17, 18, 19,   // +Y
    20, 22, 23,  20, 21, 22,   // -Y
  ]);

  return createMeshFromData(engine, "box", positions, normals, indices, uvs);
}

const [houseTex, roofTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/cubehouse.png"),
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
]);
const boxMat = createStandardMaterial();
boxMat.diffuseTexture = houseTex;
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;

// アトラスの4分割を各側面へ（上面4・底面5は見えないので未指定）
const faceUV: (FaceUV | undefined)[] = [];
faceUV[0] = [0.5, 0.0, 0.75, 1.0]; // 背面
faceUV[1] = [0.0, 0.0, 0.25, 1.0]; // 正面
faceUV[2] = [0.25, 0.0, 0.5, 1.0]; // 右側面
faceUV[3] = [0.75, 0.0, 1.0, 1.0]; // 左側面

const box = createWrappedBox(engine, { faceUV });
box.material = boxMat;
box.position.y = 0.5;
addToScene(scene, box);

const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
roof.material = roofMat;
roof.scaling.x = 0.75;
roof.rotation.z = Math.PI / 2;
roof.position.y = 1.22;
addToScene(scene, roof);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/6?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-06 マテリアル（面ごと/faceUV） / A. 一軒家"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/6

## B. Semi Detached House（2 戸 1 棟）— `semihouse.png`

`semihouse.png` は「正面(0〜0.4)・窓なし壁(0.4〜0.6)・背面(0.6〜1.0)」の 3 区画。側面 2 面（右/左）は同じ「窓なし壁」区画を共有します。
家体は `width: 2`（幅を頂点に直接焼き込み）、屋根は本家同様 `roof.scaling.y = 2` で**長さ**を 2 倍にします。

```typescript
const [houseTex, roofTex] = await Promise.all([
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/semihouse.png"),
  loadTexture2D(engine, "https://assets.babylonjs.com/environments/roof.jpg"),
]);
const boxMat = createStandardMaterial();
boxMat.diffuseTexture = houseTex;
const roofMat = createStandardMaterial();
roofMat.diffuseTexture = roofTex;

const faceUV: (FaceUV | undefined)[] = [];
faceUV[0] = [0.6, 0.0, 1.0, 1.0]; // 背面
faceUV[1] = [0.0, 0.0, 0.4, 1.0]; // 正面
faceUV[2] = [0.4, 0.0, 0.6, 1.0]; // 右側面（窓なし壁を共有）
faceUV[3] = [0.4, 0.0, 0.6, 1.0]; // 左側面（窓なし壁を共有）

const box = createWrappedBox(engine, { width: 2, faceUV });
box.material = boxMat;
box.position.y = 0.5;
addToScene(scene, box);

const roof = createCylinder(engine, { diameter: 1.3, height: 1.2, tessellation: 3 });
roof.material = roofMat;
roof.scaling.x = 0.75;
roof.scaling.y = 2;   // TRS はスケール→回転の順で合成されるため、rotation.z 適用後も「長さ」が2倍になる
roof.rotation.z = Math.PI / 2;
roof.position.y = 1.22;
addToScene(scene, roof);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/7?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-06 マテリアル（面ごと/faceUV） / B. Semi Detached House（2 戸 1 棟）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/7

> **まとめ**：box の `faceUV`/`wrap`/非一様 `width` は `createBox` 単体では非対応ですが、公開 API の
> `createMeshFromData` に本家 boxBuilder のロジックを逐語移植すれば、見た目を完全に一致させて再現できます。
> より単純に済ませたい場合は、代わりに以下も選択肢です：
> 1. **面ごとに別プレーンを貼る**（`createPlane` を複数枚、各面に別マテリアル）
> 2. **マルチマテリアル**（`.babylon` ローダー経由なら submesh + multiMaterial に対応）

---

← [2-05 テクスチャを貼る (Add Texture)](./2-05-add-texture.md) ・ [2-07 リファクタリング（関数化）](./2-07-refactoring.md) →
