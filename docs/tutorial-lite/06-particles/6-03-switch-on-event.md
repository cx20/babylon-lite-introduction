# 6-03 スイッチ オン イベント (The Switch On Event) — △

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：クリックでパーティクルシステムを開始／停止する。

本家は、水を出す/止めるボタンのメッシュを置き、`scene.onPointerObservable` のピックで `particleSystem.start()` / `.stop()` を呼びます。
Lite でも **開始/停止 API とピッキングが揃っている**ので同じことができます（作り方の土台は [6-02](./6-02-particle-spray.md) の NPE パーティクルなので、判定は △）。

**Lite 対応方針**（v1.8 ソースで確認）：

- **開始/停止** — `startParticleSystem(system)` / `stopParticleSystem(system)`。`stopParticleSystem` は新規放出を止め、残りの粒子を出し切ってから消えます。
  NPE 経由なら `registerNodeParticleSet(scene, set, { autoStart: false })` で止めた状態から始め、クリックで開始できます。
- **クリック判定** — `pickBillboardSprite` や、メッシュに対する Ray Casting / Picking（4-01 で使用）でスイッチのメッシュを拾います。

```typescript
// autoStart:false で止めた状態から始め、スイッチのメッシュがクリックされたらトグルする
const set = await parseNodeParticleSetFromSnippet(engine, scene, "<NPE スニペット ID>");
registerNodeParticleSet(scene, set, { autoStart: false });

let on = false;
// スイッチメッシュのクリックを拾って開始/停止（ピッキングは 4-01 と同じ考え方）
const toggle = () => {
  on = !on;
  for (const system of set.systems) {
    if (on) startParticleSystem(system);
    else stopParticleSystem(system);
  }
};
```

> 動作確認済みサンプルは準備中です（6-02 の NPE スニペットとあわせて反映します）。

---

← [6-02 パーティクルのスプレー (Particle Spray)](./6-02-particle-spray.md) ・ [7-00 光と影 (Light the Night)](../07-lights/7-00-light-intro.md) →
