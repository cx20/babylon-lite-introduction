# 第3部：アニメーション

> 共通テンプレート・凡例は [README](./README.md) を参照。
> このパートの終着点が [ゴール完成版](./99-goal-final.md) です。

---

## 3-00 村のアニメーション — 導入

**—** 親子関係 → 車の組み立て → 車輪/車の動き → キャラの歩行、と進みます。
Lite のアニメは 2 系統：**glTF 由来のスケルタル/クリップ**（`AnimationGroup`）と、**プロパティアニメ**（`createPropertyAnimationClip`）や `onBeforeRender` での自前更新です。

---

## 3-01 メッシュの親子関係 (Mesh Parents) — ○

追加 import：`createBox, createSphere, createStandardMaterial, onBeforeRender`

```typescript
const parent = createBox(engine, 1);   // createBox の第2引数は数値（一辺の長さ）
parent.position.y = 0.5;
parent.material = createStandardMaterial();   // ★マテリアル必須
addToScene(scene, parent);

const child = createSphere(engine, { diameter: 0.5 });
child.position.x = 1.5;        // 親からの相対位置
child.parent = parent;         // 親子付け（親の移動/回転に追従）
child.material = createStandardMaterial();     // ★マテリアル必須
addToScene(scene, child);

// 親を回すと子も一緒に回る
let a = 0;
onBeforeRender(scene, () => {
  a += 0.02;
  parent.rotationQuaternion.y = Math.sin(a / 2);
  parent.rotationQuaternion.w = Math.cos(a / 2);
});
```

> ワールド変換を保持したまま付け替えたい場合は `setParent(child, parent)` を使います。

---

## 3-02 車を組み立てる (Building the Car) — ○

**目的**：ボディ＋4 輪を組み立てる。Babylon.js の `ExtrudePolygon`（多角形フィル）はボックスで代用。

追加 import：`createBox, createCylinder, createStandardMaterial`

```typescript
// ボディ（★マテリアル必須。色/テクスチャは 3-03 で付け直す）
// createBox は一辺の長さ（数値）のみ指定可。非一様なサイズは scaling で伸縮する
const body = createBox(engine, 1);
body.scaling.x = 2; body.scaling.y = 0.5; body.scaling.z = 1;
body.position.y = 0.5;
body.material = createStandardMaterial();
addToScene(scene, body);

// 車輪（シリンダーを寝かせる：Z 軸まわり 90°）
const makeWheel = (x: number, z: number) => {
  const w = createCylinder(engine, { diameter: 0.6, height: 0.3, tessellation: 24 });
  w.rotationQuaternion.z = Math.sin(Math.PI / 4);   // 90° 傾けて車軸を横向きに
  w.rotationQuaternion.w = Math.cos(Math.PI / 4);
  w.position.x = x; w.position.y = 0.3; w.position.z = z;
  w.material = createStandardMaterial();            // ★マテリアル必須
  w.parent = body;                                  // ボディの子にする
  addToScene(scene, w);
  return w;
};
const wheels = [makeWheel(-0.7, 0.55), makeWheel(0.7, 0.55), makeWheel(-0.7, -0.55), makeWheel(0.7, -0.55)];
```

> `createCylinder` は `diameterTop/diameterBottom` 指定で円錐/角柱にもできます。

---

## 3-03 車のマテリアル (Car Material) — ○

追加 import：`createStandardMaterial, loadTexture2D`

```typescript
const bodyMat = createStandardMaterial();
bodyMat.diffuseColor = [0.8, 0.1, 0.1];       // 赤いボディ
body.material = bodyMat;

// 車輪ロゴ：cylinder は faceUV に対応しているので端面にテクスチャを貼れる
const wheelMat = createStandardMaterial();
wheelMat.diffuseTexture = await loadTexture2D(engine, "https://playground.babylonjs.com/textures/logo.png");
for (const w of wheels) w.material = wheelMat;
```

---

## 3-04 車輪のアニメーション (Wheel Animation) — ○

**目的**：車輪を回す。`onBeforeRender` で車軸（X 軸）まわりに回転。

```typescript
let spin = 0;
onBeforeRender(scene, () => {
  spin += 0.15;
  for (const w of wheels) {
    // 車軸(Z で寝かせた後の回転)まわりに回す簡易版：X 軸ヨー成分を更新
    w.rotationQuaternion.x = Math.sin(spin / 2);
    // 寝かせ(Z)成分は維持したいので、厳密には合成が必要。回転見せが目的なら下記の別法が簡単
  }
});
```

> 厳密な合成が面倒なら、**キーフレームのプロパティアニメ**が使えます：
> ```typescript
> // 追加 import: createAnimationManager, createPropertyAnimationClip, createPropertyAnimationGroup
> const mgr = createAnimationManager({ engine });
> const clip = createPropertyAnimationClip("spin", [
>   { path: "rotationQuaternion", keys: [ /* 0→360° のクォータニオン列 */ ], quaternion: true }
> ]);
> // createPropertyAnimationGroup(mgr, wheel, clip, { loop: true });
> ```
> 回転を見せるだけなら `onBeforeRender` 版で十分です。

---

## 3-05 車のアニメーション (Car Animation) — ○

**目的**：車を経路に沿って動かす。`onBeforeRender` で位置を更新（キーフレームでも可）。

```typescript
let t = 0;
onBeforeRender(scene, () => {
  t += 0.02;
  body.position.z = Math.sin(t) * 5;   // 前後に往復
});
```

---

## 3-06 キャラクターのアニメーション (Character Animation) — ○（アセットは glTF）

**目的**：リグ付きキャラを読み込み歩行アニメを再生する。

Babylon.js は `Dude.babylon` を使いますが、Lite の `.babylon` ローダーは **スキン/アニメ非対応**（`animationGroups` は常に空）。
一方で **glTF 由来のスケルタルアニメは完全対応**なので、`Xbot.glb`（walk/run 検証済み）に置換します。

追加 import：`loadGltf, playAnimation, stopAnimation`

```typescript
const xbot = await loadGltf(engine, "https://playground.babylonjs.com/scenes/Xbot.glb");
addToScene(scene, xbot);

// 全クリップを止めて walk だけ再生（idle/run が同時再生されるのを防ぐ）
for (const g of xbot.animationGroups ?? []) stopAnimation(g);
const walk = (xbot.animationGroups ?? []).find((g) => g.name === "walk") ?? xbot.animationGroups?.[0];
if (walk) { walk.loopAnimation = true; walk.speedRatio = 1.0; playAnimation(walk); }
```

| 前提 | 判定 |
|---|---|
| `Dude.babylon` をそのまま | ✕（静止表示のみ） |
| glTF（`Xbot.glb` 等）に置換 | ○（歩行/走行/ブレンドまで対応） |

---

## 3-07 村を歩き回る (A Walk Around The Village) — ○

**目的**：キャラをルートに沿って歩かせ、カメラで追う。**これがゴールの中核**です。

ポイントは 2 つ：
1. **ウェイポイント方式**で道路に沿って前進（`movePOV` 相当を「ヨー角スカラー」で表現）
2. **カメラをキャラに `parent`** して向きごと追従（Babylon.js の `camera.parent = dude` と同じ見え方）

完全なコードは [ゴール完成版](./99-goal-final.md) を参照してください。核だけ抜粋：

```typescript
// heading（進行方向のヨー角）→ Y 軸クォータニオン
const applyPose = () => {
  const h = heading * 0.5;
  root.rotationQuaternion.x = 0; root.rotationQuaternion.y = Math.sin(h);
  root.rotationQuaternion.z = 0; root.rotationQuaternion.w = Math.cos(h);
};

onBeforeRender(scene, () => {
  const t = waypoints[wp];
  let dx = t.x - root.position.x, dz = t.z - root.position.z;
  const d = Math.hypot(dx, dz);
  if (d < 0.3) { wp = (wp + 1) % waypoints.length; }
  else {
    dx /= d; dz /= d;
    heading = Math.atan2(dx, dz);
    applyPose();
    root.position.x += dx * speed; root.position.z += dz * speed;
  }
  if (camera.beta > Math.PI / 2.2) camera.beta = Math.PI / 2.2;   // upperBetaLimit 代替
});
```

> `FollowCamera` は未対応のため、**`camera.parent = root`**（親子付け）で追従します。
> RH→LH 変換の影響で前後が反転するので、カメラ角は `alpha = -Math.PI/2`（背後）にします。
