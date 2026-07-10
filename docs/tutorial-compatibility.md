# Babylon.js チュートリアル × Babylon Lite 対応可否検証

Babylon.js の公式日本語チュートリアル本
[「Babylon.js チュートリアル日本語版」（chomado）](https://zenn.dev/chomado/books/babylonjs-tutorial-ja)
の全 33 章を、**Babylon Lite（WebGPU 専用の軽量 3D エンジン）** で再現できるか記事単位で検証した結果です。

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

## 全 33 章 対応可否テーブル

### 第1部：基礎

| 章 | タイトル | 判定 | 根拠・代替手段（Lite の API / 機能比較表） |
|---|---|:--:|---|
| 1-00 | 最初 (Firsts) | — | 導入 |
| 1-01 | ハローワールド！最初のシーンとモデル | ○ | `createEngine`／`createSceneContext`／`createArcRotateCamera`／`createHemisphericLight`／`createSphere`／`createGround` すべて✅ |
| 1-02 | Web サイトをかっこよくしよう | △ | Babylon Viewer の `<babylon-viewer>` Web コンポーネントは無し。canvas＋`loadGltf`＋`createDefaultCamera`（auto-framing✅）で同等の見せ方は可 |
| 1-03 | モデルを扱う | ○ | `loadGltf`（glTF/GLB✅）／`loadBabylon`（.babylon△）。素直な `.babylon` は可だが、絶対URLテクスチャ・`geometries` 参照型は前処理要、アニメ付きは glTF 変換要 |
| 1-04 | 最初の Web アプリのセットアップ | ○ | TypeScript／Vite ネイティブ✅。`@babylonjs/lite` でアプリ構築可（手順は異なる） |

### 第2部：村の構築

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 2-00 | 村を作る | — | 概要 |
| 2-01 | 地面 (Grounding) | ○ | `createGround`✅ |
| 2-02 | サウンドを追加 | ✕ | **音声エンジン非搭載**（`KHR_audio` も—）。Web Audio API を直接叩けば鳴らせるがエンジン機能外 |
| 2-03 | メッシュを設置 | ○ | `position`／`scaling`（ObservableVec3）✅ |
| 2-04 | 基本的な家 | ○ | `createBox`✅、単一 `diffuseTexture`✅ |
| 2-05 | テクスチャを貼る | ○ | 2D テクスチャ✅、UV スケール/オフセット✅ |
| 2-06 | マテリアル（面ごと/faceUV） | △ | **box の faceUV/wrap/width は `createBox` 非対応**（`size` 数値のみ）。公開 API `createMeshFromData` に本家 boxBuilder の wrap:true ロジックを逐語移植した自作 `createWrappedBox` で再現可能（検証済み：一軒家／Semi Detached House） |
| 2-07 | リファクタリング（関数化） | ○ | 純粋な JS 設計、エンジン非依存 |
| 2-08 | メッシュを結合 | △ | **`Mesh.MergeMeshes` 相当は無し**（コード検索 0 件）。`CSG2` union／thin instances／手動頂点結合で代替 |
| 2-09 | メッシュをコピー | ○ | Thin Instances✅／`cloneTransformNode`✅。※BJS `InstancedMesh` API は🚫→thin instance で代替 |
| 2-10 | Viewer のカメラ変更 | ○ | `ArcRotateCamera`（orbit/zoom/pan/inertia）✅ |
| 2-11 | Web アプリ レイアウト | ○ | HTML/CSS 層。canvas 埋め込みのみでエンジン制約なし |

### 第3部：アニメーション

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 3-00 | 村のアニメーション | — | 導入 |
| 3-01 | メッシュの親子関係 | ○ | 任意エンティティ間の親子化✅、`setParent`（ワールド変換保持）✅ |
| 3-02 | 車を組み立てる | ○ | `createCylinder`／`createBox`／extrude✅。※車体の `ExtrudePolygon`（多角形フィル）は要確認、箱/カスタムメッシュで代替可 |
| 3-03 | 車のマテリアル | ○ | Standard/PBR＋テクスチャ✅。**車輪ロゴの cylinder faceUV は対応**（`create-cylinder.ts` に faceUV あり） |
| 3-04 | 車輪のアニメーション | ○ | `createPropertyAnimationClip`（rotation キーフレーム）✅、またはレンダーループで回転 |
| 3-05 | 車のアニメーション | ○ | プロパティアニメ（position キーフレーム、LINEAR/STEP/CUBICSPLINE）✅ |
| 3-06 | キャラクターのアニメーション | ○※ | **Skeletal Animation✅＋Animation Groups✅**。ただし **Dude.babylon そのままは不可**（`.babylon` はスキン/アニメ非対応）→ **glTF（例：Xbot.glb）に置換で歩行/走行まで検証済み対応** |
| 3-07 | 村を歩き回る | △ | **`mesh.movePOV`／`mesh.rotate` は—**（ヨー角スカラーを保持して `position`／`rotationQuaternion` を自前更新）。**`FollowCamera` も—**（`ArcRotateCamera`＋毎フレーム target 追従、または `camera.parent` で自作）。線の軌道は `CreateLines` が無いため細円柱＋無照明マテリアルで代替。歩行キャラは 3-06 と同じく **`Dude.babylon` 不可 → `Xbot.glb` に置換** |

### 第4部：衝突回避

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 4-00 | コリジョン回避 | — | 導入 |
| 4-01 | 車の衝突事故を回避する | △ | **`intersectsMesh` は—**（AABB の重なり判定を自作）。**`material.wireframe` も—**（line-list 非対応 → 12 辺を細円柱で描画）。歩行キャラは `Xbot.glb` に置換。`loadGltf` は glTF ルートに x スケール `-1` を入れるため、判定はローカル座標へ揃える。Ray Casting/Picking（✅）でより厳密な判定も可。物理は Havok V2 サブセット（⚡） |

### 第5部：環境改善

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 5-00 | より良い環境に | — | 導入 |
| 5-01 | 遠くの丘 | ○ | `createGroundFromHeightMap`✅（GPU テクスチャ→頂点変位）。※第1引数に `engine` が要り、戻り値は `Promise<Mesh>` の async。画像は絶対 URL で指定 |
| 5-02 | 頭上の空 | ○ | **`loadSkybox(scene, baseUrl, ext, size)`** で本家の `CreateBox`＋`CubeTexture`(SKYBOX_MODE)＋`backFaceCulling=false` を一発置換✅。IBL 環境光まで要るなら `loadEnvironment`（`brdfUrl` 必須）。`camera.upperBetaLimit` も**あり**（`setCameraLimits` 経由が作法） |
| 5-03 | 木のスプライト | △ | **Sprites は部分対応（⚡）**：2D レイヤー/ビルボード（facing/axis-locked/cutout）は可だが、完全な `SpriteManager` API は無し |

### 第6部：パーティクル効果

| 章 | タイトル | 判定 | 根拠・代替手段 |
|---|---|:--:|---|
| 6-00 | パーティクル噴水 | ✕ | **Particle System は—（未対応）** |
| 6-01 | 旋盤で回された噴水 | ✕ | Particle System 未対応＋**`CreateLathe` も無し**（回転体は ribbon/tube で近似可だが噴水が主目的） |

---

## 集計（導入章 5 を除く 28 章）

| 判定 | 章数 | 割合 |
|---|:--:|---|
| ○ 対応可 | 19 | 68% |
| △ 条件付き | 6 | 21% |
| ✕ 未対応 | 3 | 11% |

**総括:** 造形・変換・マテリアル・テクスチャ・ライト・カメラ・親子関係・**スケルタルアニメ（glTF 経由）**・ハイトマップ地形・スカイボックス/IBL まで、村チュートリアルの中核はほぼ再現可能。詰まる 3 章は **音声 (2-02)・パーティクル (6-00 / 6-01)** に集約される。△ の主因は、①Babylon.js の便利ヘルパー欠如（`MergeMeshes`・`FollowCamera`・box `faceUV`・`intersectsMesh`）と ②部分対応機能（Sprites ⚡）。①は自前実装や別 API で回避でき、まさに「小さいコードで同等の見た目」という Lite の性格に沿う。

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
| `new BABYLON.Sound(...)` | 非対応 → Web Audio API |
| `new ParticleSystem(...)` | 非対応 |

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

> 本表は 2026-07-06 時点の Babylon Lite ドキュメントに基づく。Lite は機能追加が続いているため、
> 最新の対応状況は [Feature Comparison](https://doc.babylonjs.com/lite/02-feature-comparison) を参照のこと。
> ※印および「未確認」は、機能比較表・アーキテクチャ docs・リポジトリのコード検索から推定した項目。
