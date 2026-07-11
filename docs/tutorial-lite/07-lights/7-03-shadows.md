# 7-03 影を追加 (Adding Shadows) — ○

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：平行光源で人型モデルの影を地面に落とす。本家 Dude を歩かせて影を確認するサンプルの Lite 移植です。

**Lite 対応方針**（v1.8 ソースで確認）。影の作り方が本家と大きく違うのに加えて、**スキン（ボーンアニメ）モデルを影キャスターにする**特有のハマりどころが複数あります。ここでは要点を 5 つに絞ります。

### モデルは Dude.babylon ではなく Xbot.glb を使う

本家は `Dude.babylon` を読み込んでスキニングアニメで歩かせますが、**Lite の `.babylon` ローダーはスキニング非対応**です。代わりに glTF の `Xbot.glb` を使い、内蔵の `"walk"` アニメーションを再生します。

### glTF の合成ルートは `"__root__"`（＝ `entities[0]`）

`loadGltf` は右手系→左手系変換のため、`scaling=(-1,1,1)` を持つ**合成ルートノードを `"__root__"` という名前で作り**、`container.entities[0]` に置きます（Xbot.glb 自身のノードは `mixamorig:*` などで `"Xbot"` は存在しません）。名前 `"Xbot"` でルートを探すと見つからず例外で `createScene` 全体が失敗するので、**ルートは `entities[0]` で取得**します。拡大はルートの `scaling` へ一様乗算し（x 反転を保つため `set` しない）、向きの補正は `rotation.y = Math.PI` で行います。

### 再生するのは先頭(idle)ではなく `"walk"`

Xbot の `animationGroups` は idle / agree / run / … の順で、**先頭 `[0]` は idle** です。そのまま回すと立ったままなので、全グループを `stopAnimation` してから名前 `"walk"` のグループだけを `playAnimation` します。また**コンテナは「丸ごと」`addToScene`** すること。`animationGroups` が毎フレーム自動 tick されるのはコンテナ渡しのときだけで、`entities` を個別追加すると bind(T)ポーズで固まります。

### 影の作り方が本家と違う（フレームグラフ方式）

本家は `new ShadowGenerator(1024, light)` ＋ `addShadowCaster` ＋ `ground.receiveShadows = true` ですが、Lite は手順が異なります。

- 平行光源 `createDirectionalLight` を作り `position` を設定して `addToScene`。
- `light.shadowGenerator = createEsmDirectionalShadowGenerator(engine, light, {...})`。**StandardMaterial を受け手にする平行光源影は ESM が実績パス**です（PCF 平行光源の受け手は PBR/NME 前提）。`orthoMinZ` / `orthoMaxZ` はライトビュー空間の near/far。
- `setShadowTaskCasterMeshes(gen, [caster メッシュ配列])` で影を落とすメッシュを供給。キャスターは自前走査せず `getContainerMeshes()` で集めます。
- 受ける側は `mesh.receiveShadows = true`。
- 登録は `registerScene` ではなく **`registerSceneWithShadowSupport`**。

### スキンキャスターは「境界のローカル化」と「毎フレーム再描画」が必須

glTF キャスターには 2 つの対策が要ります。

1. **境界の上書き（影切れ・影消失の対策）**：平行光源影のフラスタム自動フィットは境界を「ローカル境界 × worldMatrix」で評価しますが、**glTF ローダーは境界を「ロード時ワールド空間」で保存**するため二重変換となり、影が直線で切れる／キャスター全体がフラスタム外に出て影が消えます。`setLocalBoundsFromWorldAabb()` で実在ワールド体積からローカル境界を再計算して回避します（スケール・回転を確定させた後に呼ぶこと）。
2. **`forceRefreshEveryFrame: true`**：影マップの再描画は `worldMatrixVersion` のダーティ判定で抑制されますが、**スキンアニメはボーンだけが動きノードのワールド行列が変わらない**ため、既定(false)だと影マップが初回の 1 度しか描かれません（初回はアニメ tick 前で骨姿勢が未確定＝影が空になり得る）。毎フレーム強制再描画にすることで、歩行の姿勢に影が毎フレーム追従します。

## メインコード

```typescript
/********************************************************************
 * Getting Started: 影を落とす (Shadows) — Babylon Lite 移植版
 * ------------------------------------------------------------------
 * 平行光源で人型モデルの影を地面に落とす。本家 Dude.babylon は Lite の
 * .babylon ローダーがスキニング非対応のため、代わりに Xbot.glb を使用し、
 * 内蔵の "walk" アニメーションを再生する。
 *
 * ── 今回の移植で押さえた5点（＝元コードが動かなかった原因）──────
 *
 *  (1) glTF の合成ルートは "__root__" であって "Xbot" ではない
 *      loadGltf は RH→LH 変換のため、scaling=(-1,1,1) を持つ合成ルート
 *      ノードを "__root__" という名前で作り、container.entities[0] に置く
 *      （Xbot.glb 自身のノードは mixamorig:* 等で "Xbot" は存在しない）。
 *      → 名前 "Xbot" でルートを探すと見つからず例外落ちし、createScene
 *        全体が失敗して何も描画されない。ルートは entities[0] で取得する。
 *
 *  (2) 再生すべきは先頭クリップ(idle)ではなく "walk"
 *      Xbot の animationGroups は idle / agree / run / sad_pose / walk /
 *      sneak_pose / headShake の順。先頭[0]は idle なので、そのまま回すと
 *      立ったままになる。名前 "walk" のグループを取り出し、他を全停止して
 *      walk だけを再生する（loopAnimation は既定 true）。
 *
 *  (3) コンテナは「丸ごと」addToScene する
 *      addToScene(scene, container) で渡したときだけ、その animationGroups が
 *      毎フレーム自動 tick される。entities を個別に追加すると自動 tick が
 *      働かず bind(T)ポーズで固まる。
 *
 *  (4) 影キャスターは getContainerMeshes() で集める
 *      自前でノード走査せず公式ヘルパーを使う。Lite の影は「マテリアルの
 *      頂点ステージを流用した depth ビュー」で描くので、スキンメッシュも
 *      アニメ姿勢のまま影に反映される（bind ポーズにならない）。
 *
 *  (5) glTF キャスターは境界の上書きが必要（影切れ・影消失の対策）
 *      平行光源影のフラスタム自動フィットは boundMin/boundMax を
 *      「ローカル境界 × worldMatrix」で評価する。手続き生成メッシュは
 *      ローカル境界なので正しいが、glTF ローダーは境界を「ロード時
 *      ワールド空間」で保存するため二重変換となり、フラスタムが実体から
 *      ズレる（軽度: 影が直線で切れる / 重度: キャスター全体がフラスタム外
 *      で影ゼロ）。setLocalBoundsFromWorldAabb() で実在ワールド体積から
 *      ローカル境界を再計算して回避する。あわせてスキンアニメは
 *      forceRefreshEveryFrame: true が必須（影マップの再描画判定が
 *      worldMatrixVersion 基準のため、ボーンだけ動く歩行では発火しない）。
 *
 * ── 影まわりの Lite 対応（本家と仕組みが違う）─────────────
 *   本家: new ShadowGenerator(1024, light); shadowGenerator.addShadowCaster(dude, true);
 *         ground.receiveShadows = true;
 *   Lite: フレームグラフ方式で手順が異なる:
 *     (a) createDirectionalLight(direction, intensity) → light.position 設定 → addToScene。
 *     (b) light.shadowGenerator = createEsmDirectionalShadowGenerator(engine, light, {...})。
 *         平行光源の投影(ortho)の XY はキャスターの AABB に自動フィット、near/far は
 *         orthoMinZ/orthoMaxZ で指定（ライトビュー空間）。StandardMaterial を受け手に
 *         する平行光源影は ESM が実績パス（PCF 平行光源は PBR/NME 受け手前提）。
 *     (c) setShadowTaskCasterMeshes(gen, [caster メッシュ配列]) で影を落とすメッシュを供給。
 *     (d) 受ける側は mesh.receiveShadows = true。
 *     (e) 登録は registerScene ではなく registerSceneWithShadowSupport。
 ********************************************************************/
import {
    addToScene,
    attachControl,
    createArcRotateCamera,
    createDirectionalLight,
    createEngine,
    createEsmDirectionalShadowGenerator,
    createGround,
    createSceneContext,
    createStandardMaterial,
    getContainerMeshes,
    loadGltf,
    playAnimation,
    registerSceneWithShadowSupport,
    setShadowTaskCasterMeshes,
    startEngine,
    stopAnimation,
    type AnimationGroup,
    type EngineContext,
    type Mesh,
    type SceneContext,
    type SceneNode,
} from "@babylonjs/lite";

const XBOT_URL = "https://playground.babylonjs.com/scenes/Xbot.glb";

/**
 * 【glTF キャスターの影切れ対策】メッシュの boundMin/boundMax を
 * 「指定ワールド空間 AABB に対応するローカル境界」で上書きする。
 *
 * 背景: Lite の平行光源影は、キャスターの boundMin/boundMax を
 * 「ローカル境界」と見なし mesh.worldMatrix で変換してフラスタムを
 * 自動フィットする(手続き生成メッシュはローカル境界なので正しい)。
 * ところが glTF ローダーは boundMin/boundMax を「ロード時ワールド空間」で
 * 保存するため(computeAabb(positions, worldMatrix))、フィット時に
 * ワールド変換が二重適用され、フラスタムが実体からズレて影が刈り取られる。
 * → キャスターの実在ワールド体積を指定し、各メッシュの worldMatrix の
 *   逆行列でローカルへ戻した AABB を設定して、フィットの前提に合わせる。
 *
 * 注意: worldMatrix が確定した後(スケール・回転の設定後)に呼ぶこと。
 */
function setLocalBoundsFromWorldAabb(meshes: readonly Mesh[], worldMin: readonly [number, number, number], worldMax: readonly [number, number, number]): void {
    for (const mesh of meshes) {
        const m = mesh.worldMatrix as unknown as ArrayLike<number>;
        // 列優先 4x4 (アフィン)の逆行列: 線形部 A の 3x3 逆 + 平行移動
        const a = m[0]!, b = m[4]!, c = m[8]!;
        const d = m[1]!, e = m[5]!, f = m[9]!;
        const g = m[2]!, h = m[6]!, i = m[10]!;
        const det = a * (e * i - f * h) - b * (d * i - f * g) + c * (d * h - e * g);
        if (!Number.isFinite(det) || Math.abs(det) < 1e-12) {
            continue; // 特異行列はスキップ(既定境界のまま)
        }
        const inv = [
            (e * i - f * h) / det, (c * h - b * i) / det, (b * f - c * e) / det,
            (f * g - d * i) / det, (a * i - c * g) / det, (c * d - a * f) / det,
            (d * h - e * g) / det, (b * g - a * h) / det, (a * e - b * d) / det,
        ];
        const tx = m[12]!, ty = m[13]!, tz = m[14]!;
        let minX = Infinity, minY = Infinity, minZ = Infinity;
        let maxX = -Infinity, maxY = -Infinity, maxZ = -Infinity;
        for (let ci = 0; ci < 8; ci++) {
            const wx = (ci & 1 ? worldMax[0] : worldMin[0]) - tx;
            const wy = (ci & 2 ? worldMax[1] : worldMin[1]) - ty;
            const wz = (ci & 4 ? worldMax[2] : worldMin[2]) - tz;
            const lx = inv[0]! * wx + inv[1]! * wy + inv[2]! * wz;
            const ly = inv[3]! * wx + inv[4]! * wy + inv[5]! * wz;
            const lz = inv[6]! * wx + inv[7]! * wy + inv[8]! * wz;
            minX = Math.min(minX, lx); maxX = Math.max(maxX, lx);
            minY = Math.min(minY, ly); maxY = Math.max(maxY, ly);
            minZ = Math.min(minZ, lz); maxZ = Math.max(maxZ, lz);
        }
        mesh.boundMin = [minX, minY, minZ];
        mesh.boundMax = [maxX, maxY, maxZ];
    }
}

/** 名前でアニメーショングループを取得（見つからなければ例外） */
function requireGroup(groups: readonly AnimationGroup[], name: string): AnimationGroup {
    const group = groups.find((g) => g.name === name);
    if (!group) {
        const available = groups.map((g) => g.name).join(", ");
        throw new Error(`アニメーション "${name}" が見つかりません。存在するのは: ${available}`);
    }
    return group;
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);

    // カメラ（本家と同一: alpha = -3π/4, beta = π/3, radius 50）
    const camera = createArcRotateCamera((-3 * Math.PI) / 4, Math.PI / 3, 50, { x: 0, y: 0, z: 0 });
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // 平行光源（本家 direction=(0,-1,1), position=(0,15,-30)）
    const light = createDirectionalLight([0, -1, 1], 1);
    light.position.set(0, 15, -30);
    addToScene(scene, light);

    // 地面（100×100）。影を受ける。
    const ground = createGround(engine, { width: 100, height: 100 });
    ground.material = createStandardMaterial();
    ground.receiveShadows = true; // 本家 ground.receiveShadows = true
    addToScene(scene, ground);

    // Xbot を読み込み、コンテナ丸ごと追加（→ animationGroups が自動 tick される）
    const xbot = await loadGltf(engine, XBOT_URL);
    addToScene(scene, xbot);

    // スケール調整。
    //   Xbot は等身大(≈1.8ユニット)。本家 Dude は 0.2 倍で使うが、Xbot は native が
    //   小さいので、逆に拡大して 100×100 の地面・radius50 のカメラに見合う大きさにする。
    //   ルートは合成ルート "__root__"（entities[0]）。scaling=(-1,1,1) の x 反転を
    //   保つため、set ではなく現在値へ一様乗算する（回転は恒等なので姿勢に影響なし）。
    const root = xbot.entities[0] as SceneNode;
    const SIZE = 8; // 見た目の大きさ。好みで調整可
    const s = root.scaling;
    s.set(s.x * SIZE, s.y * SIZE, s.z * SIZE);

    // 向きの補正: Xbot のメッシュ正面は本家 Dude と 180° 逆を向くため、
    //   合成ルートを world-Y 周りに 180° 回して正面を反転する。
    //   __root__ の回転は恒等(bakeなし)なので rotation.y の代入で問題ない
    //   （bake クォータニオンを持つ子ノードとは異なり上書き事故は起きない）。
    //   scaling(x反転) とは独立チャンネルなので拡大には影響しない。
    root.rotation.y = Math.PI;

    // 先頭(idle)ではなく "walk" を再生。他クリップは全停止して walk だけを回す。
    const groups = xbot.animationGroups ?? [];
    for (const g of groups) {
        stopAnimation(g);
    }
    const walk = requireGroup(groups, "walk");
    walk.loopAnimation = true; // 既定 true だが明示
    playAnimation(walk);

    // 影生成器: 平行光源には ESM を使う（本家 new ShadowGenerator(1024, light) 相当）。
    //   ※ PCF 平行光源も API 上は存在するが、StandardMaterial を受け手にした
    //     平行光源影の実績パスは ESM（公式 Shadows サンプル= scene4 と同じ）。
    //     PCF 平行光源の受け手は PBR/NME 前提のため、標準マテリアル地面だと
    //     影が出ないことがある。ここでは確実な ESM を採用する。
    //   orthoMinZ/orthoMaxZ はライトビュー空間の near/far。光源(0,15,-30)から
    //     キャスターまでの距離(≈20〜34)を十分に含む範囲にする。
    //   forceRefreshEveryFrame: true が【スキンモデルでは必須】。
    //     影マップの再描画はキャスターの worldMatrixVersion の合計での
    //     ダーティ判定に抑制されるが、スキンアニメはボーン（GPUボーンテクスチャ）
    //     だけが動きノードのワールド行列は変わらないため、既定(false)だと
    //     影マップは初回の1度しか描かれない（初回はアニメ tick 前で骨姿勢が
    //     未確定のため、影が空になり得る）。毎フレーム強制再描画にすることで
    //     影が表示され、かつ walk の姿勢に毎フレーム追従する。
    light.shadowGenerator = createEsmDirectionalShadowGenerator(engine, light, {
        mapSize: 2048, // 1024→2048: 輪郭の解像度を倍に(負荷が気になる場合は1024へ)
        depthScale: 50,
        bias: 0.00005,
        blurKernel: 8, // 32→8: ぼかしを弱めてエッジを立てる(1でほぼ無ぼかし)
        blurScale: 1, // 2→1: ぼかしを縮小バッファでなく等倍で実施(にじみ低減)
        darkness: 0, // 0 = 最も濃い影, 1 = 影なし
        frustumEdgeFalloff: 0,
        orthoMinZ: 0.1,
        orthoMaxZ: 1000,
        forceRefreshEveryFrame: true, // ★スキン(アニメ)キャスターの必須設定
    });

    // 影キャスターに Xbot 配下の全メッシュを供給(本家 addShadowCaster(dude, true) 相当)。
    // スキンメッシュもアニメ姿勢のまま影になる(depth パスが頂点スキニングを流用)。
    //
    // 【重要】glTF メッシュはローダーが境界をワールド空間で持つため、そのままだと
    //   フラスタム自動フィットが二重変換でズレて影が切れる/消える。キャスターの
    //   実在ワールド体積(Xbot: 原点・高さ約14.4・歩行の振り込み)を指定して
    //   境界をローカル化してから登録する(スケール・回転の設定後に行うこと)。
    const casterMeshes = getContainerMeshes(xbot);
    // 体積は実体ぴったりに絞るほど影マップの密度が上がり輪郭が締まる。
    // Xbot(高さ≈14.4, 歩行の腕振り・足のストライドを含め x,z ±5 で十分)
    setLocalBoundsFromWorldAabb(casterMeshes, [-5, 0, -5], [5, 15, 5]);
    setShadowTaskCasterMeshes(light.shadowGenerator, casterMeshes);

    return scene;
}

async function main(): Promise<void> {
    const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement | null;
    if (!canvas) {
        throw new Error('Canvas要素 "#renderCanvas" が見つかりません。');
    }
    const engine = await createEngine(canvas);
    const scene = await createScene(engine, canvas);
    // 影対応の登録（通常の registerScene ではなくこちら）
    await registerSceneWithShadowSupport(scene);
    await startEngine(engine);
}

main().catch((error: unknown) => {
    console.error("シーンの構築に失敗しました:", error);
});
```

## 動作サンプル

<iframe src="https://liteplayground.babylonjs.com/snippet/O42L06/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 7-03 影を追加（歩く人型モデルの影）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/O42L06/v/0
>
> 平行光源で歩く人型モデル（Xbot）の影を地面に落とします。Lite はスキニング非対応の `.babylon` ローダーを避けて `Xbot.glb` を使い、影は本家の `ShadowGenerator` ではなく **`createEsmDirectionalShadowGenerator` ＋ `setShadowTaskCasterMeshes` ＋ `registerSceneWithShadowSupport`** のフレームグラフ方式で作ります。スキンキャスターでは **境界のローカル化**と **`forceRefreshEveryFrame: true`** がハマりどころです。

## 応用：村を歩き回るキャラクター + 影

上のサンプルは人型モデル単体の影でした。ここでは `valleyvillage.glb` の村の中を Xbot が巡回コースに沿って歩き、平行光源の影を地面と家の壁に落とします。移動の基本は [7-01](./7-01-lights.md) / [7-02](./7-02-day-to-night.md) の村サンプルと共通ですが、**歩行するスキンキャスターに影を組み合わせる**ための追加ポイントがあります。

### 影生成器は ESM ではなく PCF を使う

上の単体サンプルは `createEsmDirectionalShadowGenerator`（ESM）でしたが、**村のように立体物がある場面では `createPcfDirectionalShadowGenerator`（PCF）**を使います。ESM は「深度差の指数」で影を計算するため、遷移幅（深度レンジ）の範囲では**キャスターより手前（光源側）の面にも影がにじみ**、屋根などにキャラの影が写り込みます。PCF はハードウェア深度比較で手前／奥を厳密判定するのでこの問題が起きません。受け手が PBR（glTF）の平行光源 PCF は公式検証済みの組み合わせです。スキンキャスター対応の要点（`forceRefreshEveryFrame` ＋ 境界のローカル化）は ESM と共通です。

### 影フラスタムを移動するキャラに毎フレーム追従させる

単体サンプルでは境界のローカル化は初回一度きりでしたが、**キャラが村中を移動する**ため、`setLocalBoundsFromWorldAabb()` を**毎フレーム現在位置の周囲で更新**します。位置は `dude.worldMatrix` の平行移動成分から直接取得すると、`__root__` の x 反転などの符号仮定に依存せず堅牢です。

### 向きは bake クォータニオンへの合成で与える（`rotation.y` 代入は禁止）

Xbot ノードの `rotationQuaternion` には**直立用の bake が入っている**ため、`rotation.y` を代入すると bake が壊れてキャラが倒れます。毎フレーム `rotationQuaternion = quatY(ψ) ⊗ bake` を合成代入します。移動は本家の「正面 = -Z」規約（`forward(φ) = (-sinφ, 0, -cosφ)`）で行い、`__root__` の `scale(-1,1,1)`（RH→LH 変換）に合わせて **ワールド → ローカルの x 反転**（位置 `x → -x`、yaw `φ → -φ`、見た目 yaw `ψ = π - φ`）で与えます。

### スケールは `__root__` ではなく Xbot ノード自身に掛ける

サイズ調整の乗算先は**合成ルート `__root__` ではなく Xbot ノード自身**です。`__root__` に掛けると配下の `position` もワールド換算で縮み、巡回ルートが村の配置からズレます。Xbot ノード自身のスケール（内部 0.01）に掛ければ、`position`/`rotation` は独立チャンネルなのでルートは不変のままキャラだけが縮みます。

### 影を濃くするために IBL の寄与を絞る

影が遮るのは平行光源の**直接光だけ**で、`loadEnvironment` の IBL（環境光）は影の中まで一様に照らすため、既定強度（1.0）のままだと影が薄く見えます。村・キャラ両方の PBR マテリアルの `environmentIntensity` を `0.3` 程度に下げると、本家（IBL なし・平行光源のみ）のような濃い影に近づきます。あわせてスカイボックスは村の地形より大きい `1000` を明示（既定 100 だと地形が箱を突き抜けて空色が帯状に見える）、`upperBetaLimit` は Lite に無いので `onBeforeRender` で `beta` をクランプします。

```typescript
/********************************************************************
 * Getting Started: 村を歩き回るキャラクター + 影 — Babylon Lite 移植版
 * ------------------------------------------------------------------
 * valleyvillage.glb の中を Xbot が walk で巡回し、平行光源の影を落とす。
 * 本家 Dude.babylon は Lite の .babylon ローダーがスキニング非対応のため
 * Xbot.glb を使用（等身大・内部スケール0.01・直立 X+90° bake 済み →
 * 本家の scaling 0.008 相当の指定は不要）。
 *
 * ── 移動系（確立済みの規約）────────────────────────────────
 *  (1) 向きは「bake クォータニオンへの合成」で与える。
 *      Xbot ノードの rotationQuaternion には直立用 bake が入っている。
 *      rotation.y 代入は bake を破壊してキャラが倒れるので、
 *      毎フレーム rotationQuaternion = quatY(ψ) ⊗ bake を合成代入する。
 *  (2) glTF 内蔵アニメは「コンテナ丸ごと」addToScene（自動 tick）。
 *      ロード直後は idle が再生中 → 全停止して walk へ切り替え。
 *  (3) movePOV/rotate 相当は自前実装。本家は「正面 = -Z」規約:
 *      world forward(φ) = (-sin φ, 0, -cos φ)、rotate(Y,deg) は φ += toRad(deg)。
 *  (4) glTF の合成ルート __root__ は scale(-1,1,1)（RH→LH 変換）。
 *      Xbot ノードのローカル値はワールドと x 反転の関係になるので、
 *        ワールド位置 (x,y,z) → ローカル (-x, y, z)
 *        ワールド yaw φ      → ローカル yaw -φ
 *      で与える。メッシュ正面は -Z 規約と 180° ずれるため、見た目の
 *      ローカル yaw は ψ = π - φ（= -(φ+π) mod 2π）。移動には足さない。
 *
 * ── 影系（Shadows 移植で確立済みの規約）───────────────────
 *  (5) light.shadowGenerator = createPcfDirectionalShadowGenerator(...)、
 *      setShadowTaskCasterMeshes でキャスター供給、受け手は
 *      receiveShadows = true、登録は registerSceneWithShadowSupport。
 *      ※ ESM は深度差の指数計算の遷移幅により「キャスターより手前の面
 *        (屋根など)」にも影がにじむため、立体物のある場面では PCF を使う。
 *  (6) スキンアニメのキャスターには forceRefreshEveryFrame: true が必須
 *      （影マップ再描画のダーティ判定が worldMatrixVersion 基準のため、
 *      ボーンだけ動く歩行では発火しない）。
 *  (7) glTF キャスターは境界の上書きが必須（Lite の既知問題）。
 *      ローダーは boundMin/boundMax を「ロード時ワールド空間」で保存するが、
 *      フラスタム自動フィットは「ローカル境界 × worldMatrix」を仮定して
 *      おり二重変換になる → 影が切れる/消える。
 *      setLocalBoundsFromWorldAabb() で実在体積からローカル境界を再計算。
 *      さらに本サンプルはキャラが村中を移動するので、毎フレーム現在位置の
 *      周囲でこれを更新し、影フラスタムをキャラに追従させる。
 *
 * ── その他 ─────────────────────────────────────────────
 *  - スカイボックス: 本家 CubeTexture("textures/skybox") 相当の loadSkybox
 *    （同じ6枚jpg）。IBL は loadEnvironment(skipSkybox) で別途ロード。
 *    箱サイズは valleyvillage の地形より大きい 1000 を明示
 *    （既定100だと地形が箱を突き抜けて clearColor が帯状に見える）。
 *  - upperBetaLimit は Lite に無い → onBeforeRender で beta をクランプ。
 ********************************************************************/
import {
    addToScene,
    attachControl,
    createArcRotateCamera,
    createDirectionalLight,
    createEngine,
    createPcfDirectionalShadowGenerator,
    createSceneContext,
    getContainerMeshes,
    loadEnvironment,
    loadGltf,
    loadSkybox,
    onBeforeRender,
    playAnimation,
    registerSceneWithShadowSupport,
    setShadowTaskCasterMeshes,
    startEngine,
    stopAnimation,
    type AnimationGroup,
    type AssetContainer,
    type EngineContext,
    type Mesh,
    type PbrMaterialProps,
    type SceneContext,
    type SceneNode,
} from "@babylonjs/lite";

const VILLAGE_URL = "https://assets.babylonjs.com/meshes/valleyvillage.glb";
const XBOT_URL = "https://playground.babylonjs.com/scenes/Xbot.glb";
const ENV_URL = "https://assets.babylonjs.com/core/environments/environmentSpecular.env";
// 本家 CubeTexture("textures/skybox") と同じ6枚jpg（_px/_nx/_py/_ny/_pz/_nz）
const SKYBOX_BASE = "https://playground.babylonjs.com/textures/skybox";
// Babylon-Lite リポジトリの公開 BRDF LUT（cx20/webgpu-test の実績構成）
const BRDF_URL = "https://raw.githubusercontent.com/BabylonJS/Babylon-Lite/master/lab/public/brdf-lut.png";

const toRad = (deg: number): number => (deg * Math.PI) / 180;

/** 巡回コース（本家と同一）。turn は度、dist は累計距離 */
const TRACK: ReadonlyArray<{ turn: number; dist: number }> = [
    { turn: 86, dist: 7 },
    { turn: -85, dist: 14.8 },
    { turn: -93, dist: 16.5 },
    { turn: 48, dist: 25.5 },
    { turn: -112, dist: 30.5 },
    { turn: -72, dist: 33.2 },
    { turn: 42, dist: 37.5 },
    { turn: -98, dist: 45.2 },
    { turn: 0, dist: 47 },
];

/** 名前でアニメーショングループを取得 */
function requireGroup(groups: readonly AnimationGroup[], name: string): AnimationGroup {
    const group = groups.find((g) => g.name === name);
    if (!group) {
        throw new Error(`アニメーション "${name}" が見つかりません`);
    }
    return group;
}

/** コンテナ配下から名前でノードを検索 */
function findNodeByName(container: AssetContainer, name: string): SceneNode {
    const seen = new Set<SceneNode>();
    const visit = (node: SceneNode): SceneNode | null => {
        if (seen.has(node)) {
            return null;
        }
        seen.add(node);
        if (node.name === name) {
            return node;
        }
        for (const child of node.children ?? []) {
            const found = visit(child);
            if (found) {
                return found;
            }
        }
        return null;
    };
    for (const entity of container.entities) {
        if ("children" in (entity as object)) {
            const found = visit(entity as SceneNode);
            if (found) {
                return found;
            }
        }
    }
    throw new Error(`ノード "${name}" が見つかりません`);
}

/**
 * 【glTF キャスターの影切れ対策】boundMin/boundMax を「指定ワールド AABB に
 * 対応するローカル境界」で上書きする（詳細はヘッダー (7) 参照）。
 * worldMatrix 確定後に呼ぶこと。移動するキャスターには毎フレーム呼ぶ。
 */
function setLocalBoundsFromWorldAabb(meshes: readonly Mesh[], worldMin: readonly [number, number, number], worldMax: readonly [number, number, number]): void {
    for (const mesh of meshes) {
        const m = mesh.worldMatrix as unknown as ArrayLike<number>;
        const a = m[0]!, b = m[4]!, c = m[8]!;
        const d = m[1]!, e = m[5]!, f = m[9]!;
        const g = m[2]!, h = m[6]!, i = m[10]!;
        const det = a * (e * i - f * h) - b * (d * i - f * g) + c * (d * h - e * g);
        if (!Number.isFinite(det) || Math.abs(det) < 1e-12) {
            continue;
        }
        const inv = [
            (e * i - f * h) / det, (c * h - b * i) / det, (b * f - c * e) / det,
            (f * g - d * i) / det, (a * i - c * g) / det, (c * d - a * f) / det,
            (d * h - e * g) / det, (b * g - a * h) / det, (a * e - b * d) / det,
        ];
        const tx = m[12]!, ty = m[13]!, tz = m[14]!;
        let minX = Infinity, minY = Infinity, minZ = Infinity;
        let maxX = -Infinity, maxY = -Infinity, maxZ = -Infinity;
        for (let ci = 0; ci < 8; ci++) {
            const wx = (ci & 1 ? worldMax[0] : worldMin[0]) - tx;
            const wy = (ci & 2 ? worldMax[1] : worldMin[1]) - ty;
            const wz = (ci & 4 ? worldMax[2] : worldMin[2]) - tz;
            const lx = inv[0]! * wx + inv[1]! * wy + inv[2]! * wz;
            const ly = inv[3]! * wx + inv[4]! * wy + inv[5]! * wz;
            const lz = inv[6]! * wx + inv[7]! * wy + inv[8]! * wz;
            minX = Math.min(minX, lx); maxX = Math.max(maxX, lx);
            minY = Math.min(minY, ly); maxY = Math.max(maxY, ly);
            minZ = Math.min(minZ, lz); maxZ = Math.max(maxZ, lz);
        }
        mesh.boundMin = [minX, minY, minZ];
        mesh.boundMax = [maxX, maxY, maxZ];
    }
}

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
    const scene = createSceneContext(engine);
    scene.clearColor = { r: 0.53, g: 0.75, b: 0.92, a: 1.0 }; // スカイボックス失敗時の空色

    // カメラ（本家: alpha=0, beta=π/4, radius=15）
    const camera = createArcRotateCamera(0, Math.PI / 4, 15, { x: 0, y: 0, z: 0 });
    camera.nearPlane = 0.1;
    camera.farPlane = 2000; // スカイボックス(1000)より十分遠くまで描画
    scene.camera = camera;
    attachControl(camera, canvas, scene);

    // 平行光源（本家: direction=(0,-1,1), position=(0,50,-100)）
    const light = createDirectionalLight([0, -1, 1], 1);
    light.position.set(0, 50, -100);
    addToScene(scene, light);

    // 村（コンテナ丸ごと追加）。影の受け手を設定。
    //
    // 【注意】valleyvillage.glb は glTF の mesh.name が全て未定義で、
    //   "ground" は「ノード名」（Lite の mesh.name はメッシュ名由来のため
    //   名前検索ではヒットしない）。本家は ground のみ受け手にしているが、
    //   ここでは名前解決に依存せず村の全メッシュを受け手にする
    //   （地面に加え、家の壁にもキャラの影が落ちるようになる）。
    // 【影を濃くする鍵】IBL(環境光)の寄与を絞る。
    //   影が遮るのは平行光源の「直接光」だけで、loadEnvironment の IBL は
    //   影の中まで一様に照らすため、既定強度(1.0)のままだと影が薄く見える
    //   （本家サンプルは IBL なし・平行光源のみなので影が濃い）。
    //   environmentIntensity は PBR マテリアル単位のスケール(BJS 同名プロパティ相当)。
    //   0 に近づけるほど本家のような濃い影になる。0.2〜0.4 が自然な日なた感。
    const ENV_INTENSITY = 0.3;

    const village = await loadGltf(engine, VILLAGE_URL);
    addToScene(scene, village);
    for (const mesh of getContainerMeshes(village)) {
        mesh.receiveShadows = true;
        const mat = mesh.material as PbrMaterialProps | null;
        if (mat) {
            mat.environmentIntensity = ENV_INTENSITY;
        }
    }

    // Xbot（コンテナ丸ごと追加 → アニメ自動 tick）。walk へ切り替え。
    const xbot = await loadGltf(engine, XBOT_URL);
    addToScene(scene, xbot);
    const groups = xbot.animationGroups ?? [];
    for (const g of groups) {
        stopAnimation(g);
    }
    playAnimation(requireGroup(groups, "walk"));

    // 巡回の駆動ノード（bake クォータニオン保持）
    const dude = findNodeByName(xbot, "Xbot");
    const bake = { x: dude.rotationQuaternion.x, y: dude.rotationQuaternion.y, z: dude.rotationQuaternion.z, w: dude.rotationQuaternion.w };

    // サイズ調整: 等身大(≈1.84)の半分に。
    //   掛ける先は __root__ ではなく Xbot ノード自身であることが重要。
    //   __root__ に掛けると配下の position もワールド換算で縮み、巡回ルートが
    //   村の配置からズレてしまう。Xbot ノード自身のスケール（内部0.01）に
    //   掛ければ、position/rotation は独立チャンネルなのでルートは不変のまま
    //   キャラ（ボーン→スキン）だけが縮む。
    const CHAR_SCALE = 0.5;
    const ds = dude.scaling;
    ds.set(ds.x * CHAR_SCALE, ds.y * CHAR_SCALE, ds.z * CHAR_SCALE);

    // 初期状態（ワールド意図: 位置(-6,0,0)・yaw -95°）→ ローカルへ変換（ヘッダー(4)）
    const START_YAW = toRad(-95); // world yaw φ
    let yaw = START_YAW;
    dude.position.set(6, 0, 0); // world x=-6 → local x=+6（__root__ の x 反転）

    /** world yaw φ から見た目のローカル quaternion を合成代入 */
    const applyHeading = (): void => {
        const psi = Math.PI - yaw; // ψ = π - φ（正面180°オフセット + x反転の符号反転）
        const sy = Math.sin(psi / 2);
        const cy = Math.cos(psi / 2);
        // quatY(ψ) ⊗ bake（Hamilton積）
        dude.rotationQuaternion.set(
            cy * bake.x + sy * bake.z,
            cy * bake.y + sy * bake.w,
            cy * bake.z - sy * bake.x,
            cy * bake.w - sy * bake.y
        );
    };
    applyHeading();

    // 影生成器: この場面は【PCF】を使う（ESM から変更）。
    //   ESM は「深度差の指数」で影を計算するため、遷移幅(深度レンジ/depthScale
    //   ≈20ユニット)の範囲では【キャスターより手前(光源側)の面まで影がにじむ】。
    //   村では屋根がキャラの頭上斜め手前でこの帯域に入り、キャラの影が屋根に
    //   写り込んでしまう。PCF はハードウェア深度比較(手前=点灯を厳密判定)なので
    //   この問題が起きない。受け手が PBR(glTF)の平行光源 PCF は公式検証済みの
    //   組み合わせ(scene66/72/140)。スキンキャスター対応の要点は ESM と共通:
    //   forceRefreshEveryFrame + 境界ローカル化(毎フレーム追従)。
    light.shadowGenerator = createPcfDirectionalShadowGenerator(engine, light, {
        mapSize: 1024,
        bias: 0.00005,
        darkness: 0, // 0 = 最も濃い影, 1 = 影なし
        orthoMinZ: 0.1,
        orthoMaxZ: 1000,
        forceRefreshEveryFrame: true, // ★スキン(アニメ)キャスターの必須設定
    });
    const casterMeshes = getContainerMeshes(xbot);
    for (const mesh of casterMeshes) {
        const mat = mesh.material as PbrMaterialProps | null;
        if (mat) {
            mat.environmentIntensity = ENV_INTENSITY; // キャラも同強度(白飛び防止・陰影の統一)
        }
    }
    setShadowTaskCasterMeshes(light.shadowGenerator, casterMeshes);

    /** キャラ現在位置(ワールド)の周囲に影フラスタム用の境界を張り直す。
     *  位置は dude.worldMatrix の平行移動成分から直接取得（__root__ の
     *  x 反転などの符号仮定に依存しない、最も堅牢な方法）。 */
    const updateCasterBounds = (): void => {
        const w = dude.worldMatrix as unknown as ArrayLike<number>;
        const wx = w[12]!;
        const wy = w[13]!;
        const wz = w[14]!;
        // 体積はキャラサイズ(半分化後 ≈0.92)に合わせて絞る(密度=影の精細度が上がる)
        setLocalBoundsFromWorldAabb(casterMeshes, [wx - 0.8, wy, wz - 0.8], [wx + 0.8, wy + 1.2, wz + 0.8]);
    };
    updateCasterBounds();

    // 巡回（本家 onBeforeRenderObservable 相当）
    let distance = 0;
    const STEP = 0.01; // 歩く速さ。足滑りが気になる場合は 0.015〜0.02 に
    let p = 0;
    onBeforeRender(scene, () => {
        // 本家 upperBetaLimit = π/2.2 の再現（Lite に beta 上限は無い）
        if (camera.beta > Math.PI / 2.2) {
            camera.beta = Math.PI / 2.2;
        }

        // world forward(φ) = (-sinφ, 0, -cosφ) → local へは x 反転
        dude.position.x += Math.sin(yaw) * STEP;
        dude.position.z += -Math.cos(yaw) * STEP;
        distance += STEP;

        if (distance > TRACK[p]!.dist) {
            yaw += toRad(TRACK[p]!.turn); // 本家 dude.rotate(Y, turn, LOCAL)
            applyHeading();
            p = (p + 1) % TRACK.length;
            if (p === 0) {
                // 一周 → 初期状態へ（本家と同じ）
                distance = 0;
                dude.position.set(6, 0, 0);
                yaw = START_YAW;
                applyHeading();
            }
        }

        // 影フラスタムをキャラに追従させる（ヘッダー(7)）
        updateCasterBounds();
    });

    return scene;
}

async function main(): Promise<void> {
    const canvas = document.getElementById("renderCanvas") as HTMLCanvasElement | null;
    if (!canvas) {
        throw new Error('Canvas要素 "#renderCanvas" が見つかりません。');
    }
    const engine = await createEngine(canvas);
    const scene = await createScene(engine, canvas);

    // IBL(環境光)。スカイボックスは loadSkybox で別途張るので skipSkybox。
    // brdf-lut は Babylon-Lite リポジトリの公開 raw URL を使用。
    try {
        await loadEnvironment(scene, ENV_URL, {
            skipSkybox: true,
            skipGround: true,
            brdfUrl: BRDF_URL,
        });
    } catch (e) {
        console.warn("環境(IBL)の読み込みに失敗（続行）:", e);
    }

    // スカイボックス: 本家と同じ textures/skybox（6枚jpg）を loadSkybox で張る。
    //   サイズ既定値は 100 で、valleyvillage の地形(数百ユニット)より小さく
    //   「青い帯」問題が再発するため 1000 を明示指定（camera.farPlane=2000 とセット）。
    try {
        await loadSkybox(scene, SKYBOX_BASE, ".jpg", 1000);
    } catch (e) {
        console.warn("スカイボックスの読み込みに失敗（clearColor で続行）:", e);
    }
    // ロード処理が clearColor を書き換える場合があるため空色を再設定
    // （スカイボックスが視界を覆っていれば見えないが、失敗時の保険）
    scene.clearColor = { r: 0.53, g: 0.75, b: 0.92, a: 1.0 };

    await registerSceneWithShadowSupport(scene);
    await startEngine(engine);
}

main().catch((error: unknown) => {
    console.error("シーンの構築に失敗しました:", error);
});
```

## 応用サンプルの動作

<iframe src="https://liteplayground.babylonjs.com/snippet/O42L06/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 7-03 応用（村を歩き回るキャラクター + 影）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/O42L06/v/1
>
> `valleyvillage.glb` の村を Xbot が巡回コースに沿って歩き、影を地面と家の壁に落とします。立体物のある場面なので影生成器は **PCF**（ESM は手前の屋根に影がにじむため）、移動するキャラに合わせて**影フラスタムの境界を毎フレーム更新**、向きは **bake クォータニオンへの合成**で与え、IBL の `environmentIntensity` を下げて影を濃く見せるのがポイントです。

---

← [7-02 昼から夜へ (Day to Night)](./7-02-day-to-night.md) ・ [8-00 世界の見方 (Ways to See The World)](../08-cameras/8-00-cameras-intro.md) →
