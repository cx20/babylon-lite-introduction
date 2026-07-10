# 6-00 パーティクル噴水 (Build a Particle Fountain) — △

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：パーティクルで噴水の水しぶきを表現する。

**Lite にパーティクルは実装されています**（v1.8 ソースで確認）。ただし本家とは**作り方が違います**。

- 本家は命令的に `new ParticleSystem(...)` を作り、`emitter` / `direction1,2` / `minEmitBox` / `gravity` / `color1,2` などのプロパティを設定します。
- **Lite は Node Particle Editor (NPE) 前提**です。パーティクルの挙動（放出・寿命・方向・色・重力）は**ノードグラフで組み**、
  そのスニペットを読み込んで使います。個々の粒子挙動を埋めるのは NPE のビルドで、`createParticleSystem` は空のシステムを作る低レベル基盤です。

そのため章の主目的（噴水）は達成できますが、チュートリアルのコードをそのまま移植する形にはなりません。判定は △ です。

**Lite での作り方**（公開 API）：

追加 import：`parseNodeParticleSetFromSnippet, registerNodeParticleSet`

```typescript
// NPE で作ったパーティクルグラフをスニペット ID で読み込む
const set = await parseNodeParticleSetFromSnippet(engine, scene, "<NPE スニペット ID>");

// シーンへ登録（内部で billboard 生成 → addFacingBillboardSystem → 自動 start）
registerNodeParticleSet(scene, set, { autoStart: true });
```

- `registerNodeParticleSet` は各システムに対して `createParticleBillboard` ＋ `addFacingBillboardSystem` を行い、既定で再生を始めます。
- 低レベルに自前で動かす場合は `createParticleSystem` → `startParticleSystem` / `stopParticleSystem` → 毎フレーム `animateParticleSystem` → `createParticleBillboard` / `syncParticleBillboard` の順で描画します。

> 動作確認済みサンプルは準備中です（NPE スニペットを用意して反映します）。
>
> 本家チュートリアルのように「エミッタ・方向・重力をコードで直接設定して噴水を作る」書き味ではなく、
> **NPE でグラフを組む**のが Lite の流儀です。この差があるため ○ ではなく △ としています。

---

← [5-03 木のスプライト (Sprite Trees)](../05-environment/5-03-sprite-trees.md) ・ [6-01 旋盤で回された噴水 (A Lathe Turned Fountain)](./6-01-lathe-fountain.md) →
