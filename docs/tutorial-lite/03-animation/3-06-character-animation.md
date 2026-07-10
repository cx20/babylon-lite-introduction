# 3-06 キャラクターのアニメーション (Character Animation) — ○（アセットは glTF）

> [第3部：アニメーション](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：リグ付きキャラを読み込み、その場で歩行アニメを再生する（移動も当たり判定も無し）。

Babylon.js は `Dude.babylon` を使いますが、Lite の `.babylon` ローダーは **スキン/アニメ非対応**（`animationGroups` は常に空）です。
一方で **glTF 由来のスケルタルアニメは完全対応**なので、`Xbot.glb`（walk/run 検証済み）に置換します。
Xbot が持つクリップは `idle` / `agree` / `run` / `sad_pose` / `walk` / `sneak_pose` / `headShake` です。

**Lite 移植時の注意点**：

- **glTF 内蔵アニメは「コンテナ丸ごと」を `addToScene` する** — こうすると `animationGroups` が毎フレーム自動 tick されます。
  `container.entities` を個別に追加すると自動 tick が働かず、**T ポーズのまま止まります**。
- **ロード直後は先頭クリップ（Xbot では `idle`）だけが再生状態** — `stopAnimation` で全停止してから `playAnimation(walk)` に切り替えます。
- **正面を向かせるには直立 bake を保ったまま Y 軸回転を合成する** — Xbot ルートの `rotationQuaternion` には直立用の `X+90°` が焼き込まれており、
  既定だと背中を向きます。ここに `rotation.y = …` を代入すると **bake が消えてキャラが倒れます**。
  移動しないこのサンプルでは、`Y180° ⊗（X+90° bake）` を**事前計算した固定クォータニオンを直接 `set`** します
  （Lite に `rotate` ヘルパーは無く、quaternion を直接 `set` するのが定石。移動を伴う 3-07 では毎フレーム合成します）。

追加 import：`loadGltf, playAnimation, stopAnimation`（＋型 `AssetContainer, SceneNode`）

```typescript
const XBOT_URL = "https://playground.babylonjs.com/scenes/Xbot.glb";

async function createScene(engine: EngineContext, canvas: HTMLCanvasElement): Promise<SceneContext> {
  const scene = createSceneContext(engine);

  const camera = createArcRotateCamera(-Math.PI / 2, Math.PI / 2.5, 50, { x: 0, y: 0, z: 0 });
  scene.camera = camera;
  attachControl(camera, canvas, scene);
  addToScene(scene, createHemisphericLight([1, 1, 0], 1));

  // コンテナ丸ごと追加（これで walk の自動 tick が有効になる）
  const xbotContainer = await loadGltf(engine, XBOT_URL);
  addToScene(scene, xbotContainer);

  playOnly(xbotContainer, "walk");   // idle を止めて walk を再生（3-07 と同じヘルパー）

  // 身長調整：Xbot 既定（内部スケール 0.01）だと radius 50 では小さすぎる。
  // 現在値へ乗算するので、内部スケールの実値に依存しない
  const xbot = findNodeByName(xbotContainer, "Xbot");
  const SIZE = 8;
  const s = xbot.scaling;
  s.set(s.x * SIZE, s.y * SIZE, s.z * SIZE);

  // 正面を向かせる。直立 bake(X+90°) に Y180° を合成した固定値を直接 set する。
  //   Y180° ⊗ (X+90° bake) = (0, 0.7071067, -0.7071068, 0)
  xbot.rotationQuaternion.set(0, 0.7071067, -0.7071068, 0);

  return scene;
}
```

<iframe src="https://liteplayground.babylonjs.com/snippet/256NBZ/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 3-06 キャラクターのアニメーション"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/256NBZ/v/0
>
> `playOnly` / `findNodeByName` は [3-07](./3-07-walk-around-village.md) と同じものです（完全版は Playground 参照）。
> `stopAnimation` で全停止してから `walk` だけを再生しないと、`idle` が重なって足が動きません。

| 前提 | 判定 |
|---|---|
| `Dude.babylon` をそのまま | ✕（静止表示のみ） |
| glTF（`Xbot.glb` 等）に置換 | ○（歩行/走行/ブレンドまで対応） |

> **`rotation.y` 直接代入で倒れる理由**：Lite の `node.rotation` は quaternion 上の Euler プロキシで、`rotation.y = h` は
> quaternion を**純 Y 回転で上書き**します。すると bake の `X+90°`（直立させる回転）が消え、モデルが寝てしまいます。
> bake を保ったまま向きだけ変えたいので、`Y ⊗ bake` を合成した値を quaternion へ直接書き込みます。

---

← [3-05 車のアニメーション (Car Animation)](./3-05-car-animation.md) ・ [3-07 村を歩き回る (A Walk Around The Village)](./3-07-walk-around-village.md) →
