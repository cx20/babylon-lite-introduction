# 1-03 モデルを扱う — ○

> [第1部：基礎](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：外部 3D モデルを読み込んで操作する。Lite は **glTF / GLB** と **.babylon** の両方に対応します。

追加 import：`createDefaultCamera, attachControl, createHemisphericLight, loadGltf, loadBabylon`

## A. glTF / GLB を読み込む（`loadGltf`）

```typescript
addToScene(scene, await loadGltf(engine, "https://playground.babylonjs.com/scenes/BoomBox.glb"));

const camera = createDefaultCamera(scene);   // 読み込んだモデルを自動フレーミング
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));
```

`loadGltf` は glTF/GLB を読み込み、メッシュ・マテリアルに加え **スキン・モーフ・アニメーション**まで対応します。

## B. .babylon を読み込む（`loadBabylon`）

`.babylon` 形式は **`loadBabylon`** で読み込めます。BJS の `SceneLoader.ImportMeshAsync` に相当（`rootUrl` + `filename` を **1 つの URL** にまとめて渡す）。
**頂点データが inline で、テクスチャ名が相対パス**の素直な `.babylon` なら、これだけで表示できます。

```typescript
addToScene(scene, await loadBabylon(engine, "https://example.com/simple_model.babylon"));

const camera = createDefaultCamera(scene);
attachControl(camera, canvas, scene);
addToScene(scene, createHemisphericLight([0, 1, 0], 1.0));
```

`loadBabylon` は **メッシュ・標準マテリアル・ライト・カメラ・シーン設定**（submesh / multiMaterial 含む）を読み込みます。

> ⚠️ **ファイルによっては現状のローダーで表示できません**（下記 D）。特にチュートリアルの
> `both_houses_scene.babylon` は 2 つの未対応を踏み、そのままでは**紫背景（clearColor）だけ**になります。

## C. 複数モデル・特定メッシュだけ読み込む

BJS はメッシュ名の配列で読み込むメッシュを選べます。Lite の `loadBabylon` はファイル全体を読み込むので、
**読み込み後に `entities` を名前で絞って** `addToScene` します。

```typescript
// BJS: ImportMeshAsync(["ground", "semi_house"], ".../", "model.babylon")
const container = await loadBabylon(engine, "https://example.com/model.babylon");

const wanted = ["ground", "semi_house"];
for (const e of container.entities) {
  if (wanted.includes((e as any).name)) addToScene(scene, e);   // 必要なメッシュだけ登録
}
// ・ファイル全体を登録するなら addToScene(scene, container) でOK
// ・読み込むメッシュ数を制限したいだけなら loadBabylon(engine, url, { maxMeshes: 2 })
```

## D. 既知の制約と回避策（.babylon ローダー）

現状の `.babylon` ローダーには 2 つの未対応があり、これを踏むファイル（例：`both_houses_scene.babylon`）は**そのままでは表示できません**。

| # | 未対応 | 症状 | 回避策 |
|---|---|---|---|
| (1) | テクスチャ `name` が**絶対URL** | `baseUrl + name` の無条件連結で壊れた URL になり `createImageBitmap` が `InvalidStateError` | `loadTextures: false` ＋ `loadTexture2D` で手動割り当て |
| (2) | 頂点データが inline でなく **`geometries` 参照型**（`mesh.geometryId` → `geometries.vertexData[]`） | ローダーが `geometryId` を処理せず全メッシュが無言でスキップ → 紫背景だけ | JSON を fetch して `vertexData` をメッシュに inline 化し、Blob URL で渡す |

`both_houses_scene.babylon` は **両方**を踏むため、次の完全版で表示できます。

動作確認済み（Lite Playground）: https://liteplayground.babylonjs.com/snippet/X79RM0/v/1

```typescript
// 追加 import: createStandardMaterial, loadTexture2D, getContainerMeshes
const SCENE_URL = "https://assets.babylonjs.com/meshes/both_houses_scene.babylon";
const ENV = "https://assets.babylonjs.com/environments/";

// ── (2) geometries 参照を inline 頂点データに変換 ──
const raw = await (await fetch(SCENE_URL)).json();
const vdMap = new Map<string, any>((raw.geometries?.vertexData ?? []).map((vd: any) => [vd.id, vd]));
for (const mesh of raw.meshes ?? []) {
  if (!mesh.positions && mesh.geometryId) {
    const vd = vdMap.get(mesh.geometryId);
    if (vd) { mesh.positions = vd.positions; mesh.normals = vd.normals; mesh.indices = vd.indices; mesh.uvs = vd.uvs; }
  }
}
// パッチ済み JSON を Blob URL で渡す
const blobUrl = URL.createObjectURL(new Blob([JSON.stringify(raw)], { type: "application/json" }));
const container = await loadBabylon(engine, blobUrl, { loadTextures: false, loadCamera: false });
URL.revokeObjectURL(blobUrl);

// ── (1) テクスチャを絶対URLのまま手動ロード ──
const [cubehouse, semihouse, roof] = await Promise.all([
  loadTexture2D(engine, ENV + "cubehouse.png"),
  loadTexture2D(engine, ENV + "semihouse.png"),
  loadTexture2D(engine, ENV + "roof.jpg"),
]);

// multiMaterial は "{メッシュ名}_sub{materialIndex}" に分割される（_sub0=壁 / _sub1=屋根）
const texMap = new Map([
  ["detached_house_sub0", cubehouse], ["detached_house_sub1", roof],
  ["semi_house_sub0", semihouse],     ["semi_house_sub1", roof],
]);
for (const mesh of getContainerMeshes(container)) {
  const tex = texMap.get(mesh.name);
  if (!tex) continue;                       // ground はファイル定義の色マテリアルのまま
  const mat = createStandardMaterial();
  mat.diffuseTexture = tex;
  mesh.material = mat;
}
addToScene(scene, container);
```

<iframe src="https://liteplayground.babylonjs.com/snippet/X79RM0/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 1-03 モデルを扱う / D. 既知の制約と回避策（.babylon ローダー）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> `getContainerMeshes(container)` は AssetContainer からメッシュを取り出すヘルパーです。
> `loadBabylon` のオプションには `loadTextures` / `loadCamera` / `maxMeshes` があります。

## 使い分けと制約

| 形式 | 関数 | メッシュ/マテリアル | スキン/アニメ | 備考 |
|---|---|:--:|:--:|---|
| glTF / GLB | `loadGltf` | ○ | **○** | 最も安定。まず glTF を推奨 |
| .babylon | `loadBabylon` | △ | **✕** | 素直なファイルは可。絶対URLテクスチャ／`geometries` 参照型は前処理が必要（D） |

- **素直な `.babylon`**（inline 頂点・相対テクスチャ名）は `loadBabylon` でそのまま表示できます。
- **一部の `.babylon`**（絶対URLテクスチャ／`geometries` 参照型）は現状のローダーで表示できず、D の前処理が必要です。
- **アニメーション付き `.babylon`**（例：`Dude.babylon`）は再生できないため、**glTF に変換**するか glTF モデル（例：`Xbot.glb`）を使います（→ [3-06](../03-animation/3-06-character-animation.md)）。

---

← [1-02 あなたの web サイトをかっこよくしよう](./1-02-website.md) ・ [1-04 最初の web アプリのセットアップ](./1-04-webapp-setup.md) →
