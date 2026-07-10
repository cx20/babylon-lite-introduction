# 6-00 パーティクル噴水 (Build a Particle Fountain) — ✕

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**Lite に Particle System はありません**（機能比較表：Particle System —）。CPU/GPU パーティクル、`ParticleSystem` 相当が未実装のため、噴水の粒子表現は再現できません。

代替の方向性（近似）：
- 少数の球メッシュ＋ thin instances を `onBeforeRender` で放物運動させる自前の簡易パーティクル
- ただし本格的なエミッタ/寿命/ブレンド制御は無く、労力に見合いません

> パーティクルが必須のシーンは、現状 Babylon.js 側が適しています。

---

← [5-03 木のスプライト (Sprite Trees)](../05-environment/5-03-sprite-trees.md) ・ [6-01 旋盤で回された噴水 (A Lathe Turned Fountain)](./6-01-lathe-fountain.md) →
