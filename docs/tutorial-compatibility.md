# Babylon.js チュートリアル × Babylon Lite 対応可否検証

Babylon.js 公式の [Getting Started](https://doc.babylonjs.com/features/introductionToFeatures/)（全 8 章）
を、**Babylon Lite（WebGPU 専用の軽量 3D エンジン）** で再現できるか節単位で検証した結果です
（1〜6 章は [chomado 氏の日本語訳](https://zenn.dev/chomado/books/babylonjs-tutorial-ja)に対応）。

判定は Babylon Lite の一次情報に基づきます。

- [Feature Comparison — Babylon Lite vs Babylon.js](https://doc.babylonjs.com/lite/02-feature-comparison)
- [Architecture docs](https://doc.babylonjs.com/lite/architecture/00-overview)（Animation / Skeleton / Loaders / Mesh Generators / Camera / Background & Skybox など）

---

## 判定基準

| 記号 | 意味 |
|:--:|---|
| ○ | 章の目的を Lite の標準機能で達成できる（API 名は Babylon.js と変わってよい） |
| △ | 一部を代替・変換・自作、または機能が部分対応（⚡） |
| ✕ | 該当機能が Lite に無く（—／🚫）、章の主目的を達成できない |
| — | 導入・概念章（コードなし） |

---

## 全 8 章 対応可否テーブル

> 各章の**タイトルは本家（Babylon.js 公式）Getting Started の該当記事**にリンクしています（英語・比較用）。
> Lite 版の各章記事は [tutorial-lite の目次](./tutorial-lite/README.md) から辿れます。
> ※ 2-07 リファクタリングは公式に対応記事が無く、[ちょまど氏の日本語版](https://zenn.dev/chomado/books/babylonjs-tutorial-ja)のオリジナル章のため、そちらへリンクしています。

### 第1部：基礎

| 章 | タイトル | 判定 | 根拠・代替手段（Lite の API / 機能比較表） |
|---|---|:--:|---|
| 1-00 | [最初 (Firsts)](https://doc.babylonjs.com/features/introductionToFeatures/chap1) | — | 導入 |
| 1-01 | [ハローワールド！最初のシーンとモデル](https://doc.babylonjs.com/features/introductionToFeatures/chap1/first_scene) | ○ | `createEngine`／`createSceneContext`／`createArcRotateCamera`／`createHemisphericLight`／`createSphere`／`createGround` すべて✅ |
| 1-02 | [Web サイトをかっこよくしよう](https://doc.babylonjs.com/features/introductionToFeatures/chap1/first_viewer) | △ | Babylon Viewer の `<babylon-viewer>` Web コンポーネントは無し。canvas＋`loadGltf`＋`createDefaultCamera`（auto-framing✅）で同等の見せ方は可 |
| 1-03 | [モデルを扱う](https://doc.babylonjs.com/features/introductionToFeatures/chap1/first_import) | ○ | `loadGltf`（glTF/GLB✅）／`loadBabylon`（.babylon△）。素直な `.babylon` は可だが、絶対URLテクスチャ・`geometries` 参照型は前処理要、アニメ付きは glTF 変換要 |
| 1-04 | [最初の Web アプリのセットアップ](https://doc.babylonjs.com/features/introductionToFeatures/chap1/first_app) | ○ | TypeScript／Vite ネイティブ✅。`@babylonjs/lite` でアプリ構築可（手順は異なる） |

### 第2部：村の構築

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 2-00 | [村を作る](https://doc.babylonjs.com/features/introductionToFeatures/chap2) | — | 概要 |
| 2-01 | [地面 (Grounding)](https://doc.babylonjs.com/features/introductionToFeatures/chap2/ground) | ○ | `createGround`✅ |
| 2-02 | [サウンドを追加](https://doc.babylonjs.com/features/introductionToFeatures/chap2/sound) | ○ | **AudioV2 ポートのオーディオエンジン搭載**✅。`createAudioEngineAsync`＋`createSoundAsync`（`loop` / `autoplay` オプション、`playSound` / `pauseSound` ほか）。ストリーミング音源・3D 空間定位（`SpatialSoundOptions`）まで対応 |
| 2-03 | [メッシュを設置](https://doc.babylonjs.com/features/introductionToFeatures/chap2/placement) | ○ | `position`／`scaling`（ObservableVec3）✅ |
| 2-04 | [基本的な家](https://doc.babylonjs.com/features/introductionToFeatures/chap2/variation) | ○ | `createBox`✅、単一 `diffuseTexture`✅ |
| 2-05 | [テクスチャを貼る](https://doc.babylonjs.com/features/introductionToFeatures/chap2/material) | ○ | 2D テクスチャ✅。UV タイリングは **`material.uvScale`**（既定 `[1,1]`）。本家の per-texture `uScale/vScale/uOffset/vOffset` とは流儀が違い、**オフセット相当のフィールドは無い** |
| 2-06 | [マテリアル（面ごと/faceUV）](https://doc.babylonjs.com/features/introductionToFeatures/chap2/face_material) | △ | **box の faceUV/wrap/width は `createBox` 非対応**（`size` 数値のみ）。公開 API `createMeshFromData` に本家 boxBuilder の wrap:true ロジックを逐語移植した自作 `createWrappedBox` で再現可能（検証済み：一軒家／Semi Detached House） |
| 2-07 | [リファクタリング（関数化）](https://zenn.dev/chomado/books/babylonjs-tutorial-ja/viewer/2-07) | ○ | 純粋な JS 設計、エンジン非依存 |
| 2-08 | [メッシュを結合](https://doc.babylonjs.com/features/introductionToFeatures/chap2/combine) | △ | **`Mesh.MergeMeshes` 相当は無し**（コード検索 0 件）。`CSG2` union／thin instances／手動頂点結合で代替 |
| 2-09 | [メッシュをコピー](https://doc.babylonjs.com/features/introductionToFeatures/chap2/copies) | ○ | Thin Instances✅／`cloneTransformNode`✅。※BJS `InstancedMesh` API は🚫→thin instance で代替 |
| 2-10 | [Viewer のカメラ変更](https://doc.babylonjs.com/features/introductionToFeatures/chap2/viewer2) | ○ | `ArcRotateCamera`（orbit/zoom/pan/inertia）✅ |
| 2-11 | [Web アプリ レイアウト](https://doc.babylonjs.com/features/introductionToFeatures/chap2/app2) | ○ | HTML/CSS 層。canvas 埋め込みのみでエンジン制約なし |

### 第3部：アニメーション

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 3-00 | [村のアニメーション](https://doc.babylonjs.com/features/introductionToFeatures/chap3) | — | 導入 |
| 3-01 | [メッシュの親子関係](https://doc.babylonjs.com/features/introductionToFeatures/chap3/parent) | ○ | 任意エンティティ間の親子化✅、`setParent`（ワールド変換保持）✅ |
| 3-02 | [車を組み立てる](https://doc.babylonjs.com/features/introductionToFeatures/chap3/polycar) | ○ | `createCylinder`／`createBox`✅。**`ExtrudePolygon`（多角形フィル）は無い**（`createExtrudeShape` はパス押し出しで別物、earcut 相当も無し）→ 凸輪郭を `createMeshFromData` で直接組み立てて代替 |
| 3-03 | [車のマテリアル](https://doc.babylonjs.com/features/introductionToFeatures/chap3/carmat) | ○ | Standard/PBR＋テクスチャ✅。**車輪ロゴの cylinder faceUV は対応**（`create-cylinder.ts` に faceUV あり） |
| 3-04 | [車輪のアニメーション](https://doc.babylonjs.com/features/introductionToFeatures/chap3/animation) | ○ | `createPropertyAnimationClip`（rotation キーフレーム）✅、またはレンダーループで回転 |
| 3-05 | [車のアニメーション](https://doc.babylonjs.com/features/introductionToFeatures/chap3/caranimation) | ○ | プロパティアニメ（position キーフレーム、LINEAR/STEP/CUBICSPLINE）✅ |
| 3-06 | [キャラクターのアニメーション](https://doc.babylonjs.com/features/introductionToFeatures/chap3/import_character) | ○※ | **Skeletal Animation✅＋Animation Groups✅**。ただし **Dude.babylon そのままは不可**（`.babylon` はスキン/アニメ非対応）→ **glTF（例：Xbot.glb）に置換で歩行/走行まで検証済み対応** |
| 3-07 | [村を歩き回る](https://doc.babylonjs.com/features/introductionToFeatures/chap3/walkpath) | △ | **`mesh.movePOV`／`mesh.rotate` は—**（ヨー角スカラーを保持して `position`／`rotationQuaternion` を自前更新）。**`FollowCamera` も—**（`ArcRotateCamera`＋毎フレーム target 追従、または `camera.parent` で自作）。線の軌道は `CreateLines` が無いため細円柱＋無照明マテリアルで代替。歩行キャラは 3-06 と同じく **`Dude.babylon` 不可 → `Xbot.glb` に置換** |

### 第4部：衝突回避

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 4-00 | [コリジョン回避](https://doc.babylonjs.com/features/introductionToFeatures/chap4) | — | 導入 |
| 4-01 | [車の衝突事故を回避する](https://doc.babylonjs.com/features/introductionToFeatures/chap4/mesh_intersect) | △ | **`intersectsMesh` は—**（AABB の重なり判定を自作）。**`material.wireframe` も—**（line-list 非対応 → 12 辺を細円柱で描画）。歩行キャラは `Xbot.glb` に置換。`loadGltf` は glTF ルートに x スケール `-1` を入れるため、判定はローカル座標へ揃える。Ray Casting/Picking（✅）でより厳密な判定も可。物理は Havok V2 サブセット（⚡） |

### 第5部：環境改善

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 5-00 | [より良い環境に](https://doc.babylonjs.com/features/introductionToFeatures/chap5) | — | 導入 |
| 5-01 | [遠くの丘](https://doc.babylonjs.com/features/introductionToFeatures/chap5/hills) | ○ | `createGroundFromHeightMap`✅（GPU テクスチャ→頂点変位）。※第1引数に `engine` が要り、戻り値は `Promise<Mesh>` の async。画像は絶対 URL で指定 |
| 5-02 | [頭上の空](https://doc.babylonjs.com/features/introductionToFeatures/chap5/sky) | ○ | **`loadSkybox(scene, baseUrl, ext, size)`** で本家の `CreateBox`＋`CubeTexture`(SKYBOX_MODE)＋`backFaceCulling=false` を一発置換✅。IBL 環境光まで要るなら `loadEnvironment`（`brdfUrl` 必須）。`camera.upperBetaLimit` も**あり**（`setCameraLimits` 経由が作法） |
| 5-03 | [木のスプライト](https://doc.babylonjs.com/features/introductionToFeatures/chap5/trees) | △ | **Sprites は部分対応（⚡）**：`loadSpriteAtlas`＋`createFacingBillboardSystem`／`createAxisLockedBillboardSystem`（cutout）で代替可。**フレームアニメも `playBillboardSpriteAnimation`＋`SpriteAnimationManager` で対応**。`SpriteManager` / `Sprite` クラスそのものは無し。※`gridSize` は分割数でなく**セルのピクセルサイズ** |

### 第6部：パーティクル効果

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 6-00 | [パーティクル噴水](https://doc.babylonjs.com/features/introductionToFeatures/chap6) | — | 導入（器 6-01 ＋ スプレー 6-02 ＋ スイッチ 6-03） |
| 6-01 | [旋盤で回された噴水](https://doc.babylonjs.com/features/introductionToFeatures/chap6/fountain) | △ | `createLathe` メソッドで器を作る章。Lite に `createLathe` は無く **`createRibbon`（`pathArray`＋`closeArray`）で自作**（本家 Lathe は内部で Ribbon 実装、`DOUBLESIDE` は `backFaceCulling=false`） |
| 6-02 | [パーティクルのスプレー](https://doc.babylonjs.com/features/introductionToFeatures/chap6/particlespray) | △ | 基本のパーティクルシステム。**パーティクルは実装ありだが命令的 `new ParticleSystem` とは流儀が違い、Node Particle Editor (NPE)** でグラフを組む→`parseNodeParticleSetFromSnippet`＋`registerNodeParticleSet`。低レベルは `createParticleSystem`＋`animateParticleSystem`＋`createParticleBillboard` |
| 6-03 | [スイッチ オン イベント](https://doc.babylonjs.com/features/introductionToFeatures/chap6/onoff) | △ | クリックで開始/停止。`startParticleSystem`／`stopParticleSystem`＋GPU ピッキング（`createGpuPicker`／`pickAsync`）。※スプライト向けの `pickBillboardSprite` は v1.10 でシグネチャ変更。土台は 6-02 の NPE パーティクル |

### 第7部：光と影

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 7-00 | [光と影（導入）](https://doc.babylonjs.com/features/introductionToFeatures/chap7) | — | 導入 |
| 7-01 | [ライトを灯す](https://doc.babylonjs.com/features/introductionToFeatures/chap7/lights) | ○ | `createPointLight`／`createDirectionalLight`／`createSpotLight`✅ |
| 7-02 | [昼から夜へ](https://doc.babylonjs.com/features/introductionToFeatures/chap7/light_gui) | △ | ライト強度のアニメ自体は○。**GUI（`AdvancedDynamicTexture`/slider）は無い**→ HTML の `<input type=range>` 等で代替 |
| 7-03 | [影を追加](https://doc.babylonjs.com/features/introductionToFeatures/chap7/shadows) | ○ | `registerSceneWithShadowSupport`＋`createEsm/Pcf/Csm…ShadowGenerator`✅、`setShadowTaskCasterMeshes` |

### 第8部：世界の見方

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 8-00 | [世界の見方（導入）](https://doc.babylonjs.com/features/introductionToFeatures/chap8) | — | 導入 |
| 8-01 | [見回す](https://doc.babylonjs.com/features/introductionToFeatures/chap8/camera) | ○ | `createFreeCamera`＋`attachFreeControl`✅（本家 `UniversalCamera` 相当）。周回は `ArcRotateCamera` |
| 8-02 | [キャラを追う](https://doc.babylonjs.com/features/introductionToFeatures/chap8/follow) | △ | **`FollowCamera` は無い**→ `camera.parent`（3-07 で確立）＋`setCameraLimits` |
| 8-03 | [VR の世界へ](https://doc.babylonjs.com/features/introductionToFeatures/chap8/vr) | ✕ | **WebXR/XR の export が0件**。`createDefaultXRExperienceAsync` 相当なし |

---

## 集計（導入章 8 を除く 35 章）

| 判定 | 章数 | 割合 |
|---|:--:|---|
| ○ 対応可 | 23 | 66% |
| △ 条件付き | 11 | 31% |
| ✕ 未対応 | 1 | 3% |

> 純粋に未対応（✕）は **8-03（WebXR / VR）** の 1 章のみ。パーティクル（第6部）は NPE 経由、音声（2-02）は AudioV2 で対応します。

**総括:** 造形・変換・マテリアル・テクスチャ・ライト・**影**・カメラ・親子関係・**スケルタルアニメ（glTF 経由）**・ハイトマップ地形・スカイボックス/IBL・**オーディオ（AudioV2）**・**パーティクル（NPE）**まで、チュートリアルの中核はほぼ再現可能。純粋に詰まるのは **WebXR (8-03)** のみ。△ の主因は、①Babylon.js の便利ヘルパー欠如（`MergeMeshes`・`FollowCamera`・box `faceUV`・`intersectsMesh`）、②作り方が異なる機能（パーティクルは NPE、GUI は HTML 側）、③部分対応機能（Sprites ⚡）。いずれも自前実装や別 API で回避でき、まさに「小さいコードで同等の見た目」という Lite の性格に沿う。

---

## 主な API 置き換え（Babylon.js → Babylon Lite）

| Babylon.js | Babylon Lite |
|---|---|
| `new ArcRotateCamera(...)` ／ `camera.attachControl` | `createArcRotateCamera(...)` ／ `attachControl(camera, canvas, scene)` |
| `SceneLoader.ImportMeshAsync(".glb")` | `loadGltf(engine, ".glb")` → `addToScene` |
| `SceneLoader.ImportMeshAsync(".babylon")` | `loadBabylon(engine, url)` → `addToScene`（静的モデル）※アニメ付きは glTF 変換 |
| `skeleton.beginAnimation("walk")` | `AnimationGroup`（`loadGltf` が自動生成、既定 loop 再生） |
| `mesh.createInstance()` | Thin Instances（`InstancedMesh` は🚫） |
| `Mesh.MergeMeshes([...])` | 非対応 → `CSG2` union ／ 手動結合 |
| `new BABYLON.Sound(...)` | `createAudioEngineAsync` ＋ `createSoundAsync`（AudioV2 ポート） |
| `new ParticleSystem(...)` | Node Particles（NPE）：`parseNodeParticleSetFromSnippet` ＋ `registerNodeParticleSet` |

---

## 補足：スケルタルアニメーションの代替（3-06）

Babylon.js チュートリアル 3-06 は `Dude.babylon` を歩行アニメ付きモデルとして使うが、Babylon Lite の
`.babylon` ローダーは **メッシュ／標準マテリアル／ライトのみ**を読み込み、スケルトン・アニメーションは対象外
（`animationGroups` は `.babylon` では常に空）。一方でエンジンの **Skeletal Animation は glTF 由来のスキン/アニメに
完全対応**しているため、glTF リグ付きモデルに置換すれば歩行アニメを再生できる。

```typescript
import { loadGltf, addToScene } from "@babylonjs/lite";

// Xbot.glb は Babylon Lite が walk/run のパリティを検証済みのモデル
const xbot = await loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb");
addToScene(scene, xbot); // スキン・アニメ tick が自動登録される

const walk = xbot.animationGroups.find((g) => /walk/i.test(g.name));
if (walk) walk.speedRatio = 1.0; // AnimationGroup は既定で自動再生・loop
```

---

> 本表は Babylon Lite v1.13.0 の[リポジトリ](https://github.com/BabylonJS/Babylon-Lite)ソースおよび
> [Feature Comparison](https://doc.babylonjs.com/lite/02-feature-comparison) に基づく（初版は v1.8 で作成。各章冒頭の
> 「vX.Y ソースで確認」は当時の確認記録）。各章の API 名・シグネチャは実際のソースで確認済み。Lite は機能追加が続いており、
> 最新の対応状況は上記を参照のこと。※印は本家との差異に関する補足で、いずれも代替手段を各章に記載している。
>
> **v1.10 → v1.13.0 の主な変化（本表への影響）**：**本表の判定・各章のサンプルコードに変更は不要**。
> ⓪一時期 **v2.0.0 が公開されていたが、これは誤ってリリースされたもの**で、v1.11.0 の
> [#412](https://github.com/BabylonJS/Babylon-Lite/pull/412)（"cap auto releases at minor and skip the burned 2.0.0 version"）
> により欠番となった。v2.0.0 として公開されていた内容は **v1.11.0** として出し直されており、**最新版は v1.13.0**（npm の
> `latest` タグも `1.13.0`）。①この期間の唯一の破壊的変更は v1.11.0 の managed storage buffers で、`setShaderStorageBuffer` が
> 生の `GPUBuffer` ではなく `createStorageBuffer` で作った `StorageBuffer` を要求するようになった点だが、本チュートリアルは
> `ShaderMaterial`／ストレージバッファ系 API を使っていないため影響なし。
> ②公開 export は追加のみ（削除・リネームなし）。`updateMeshPositions` などの更新系は末尾に省略可能な引数
> （`vertexCount` / `sourceVertexOffset`）が増えただけで、`createMeshFromData`・`createRibbon`・`createBox`・thin instances の
> シグネチャは不変。③追加機能は `updateMeshGeometry`（動的ジオメトリ更新／v1.11）・`getMeshGeometry`（v1.13）、CSM の
> `worldSpaceBias`（任意。従来の `bias` の挙動は保持されるため 7-03 は不変）、thin instance の LOD（`setThinInstanceLodPartner`）・
> 描画数の動的変更、2D テクスチャ配列、マテリアル種別ガード（`isPbrMaterial` など）。④v1.13 で `removeFromScene` が
> ライト／カメラ／`TransformNode`／`AssetContainer` も受け取れるよう拡張されたが、`removeFromScene(scene, mesh)` の呼び方は
> そのまま有効。⑤PBR の両面バックフェース法線・負スケール glTF の法線反転、glTF のブロックインターリーブ配置などの修正は
> 描画が正しくなる方向の変更で、コード側の対応は不要。⑥`MergeMeshes` / `FollowCamera` / `intersectsMesh` / `movePOV` /
> `createLathe` / `ExtrudePolygon` / `CreateLines` / `wireframe` / WebXR / GUI は **v1.13.0 でも未実装のまま**
> （box の `faceUV` も同様）のため、△・✕ の判定はいずれも据え置き。
>
> **v1.8 → v1.10 の主な変化**：破壊的変更でサンプルが動かなくなる箇所はない。
> ①パーティクルは **Node Particle Editor (NPE) が v1.9 で正式化**、v1.10 でグラデーション/指向性・半球エミッター/`isLocal`
> を追加（第6部の対応を補強）。②スプライトは v1.10 でパイプラインキャッシュ＋`disposeSpriteAtlas` を追加、ピッキング API
> （`pickSprite2D` / `pickBillboardSprite`）のシグネチャが変更されたが、本チュートリアルのスプライト章（5-03 / 8-01 / 8-02）は
> ピッキングを使わないため影響なし。③`createBoxData`（v1.9 公開）・`playSound` の per-instance pitch/playbackRate（v1.9）・
> トーンマッピングのプラグイン化と Khronos PBR Neutral（v1.9）は追加/内部改善で、既存記述の API は不変。
