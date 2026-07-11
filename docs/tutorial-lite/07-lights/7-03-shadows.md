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

---

← [7-02 昼から夜へ (Day to Night)](./7-02-day-to-night.md) ・ [8-00 世界の見方 (Ways to See The World)](../08-cameras/8-00-cameras-intro.md) →
