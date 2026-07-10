# 第6部：パーティクル効果

> [← 全体の目次](../README.md)（共通テンプレート・凡例もこちら）

| 章 | タイトル | 判定 |
|---|---|:--:|
| 6-00 | [パーティクル噴水 (Build a Particle Fountain)](./6-00-particle-fountain.md) | 導入 |
| 6-01 | [旋盤で回された噴水 (A Lathe Turned Fountain)](./6-01-lathe-fountain.md) | △ |
| 6-02 | [パーティクルのスプレー (Particle Spray)](./6-02-particle-spray.md) | △ |
| 6-03 | [スイッチ オン イベント (The Switch On Event)](./6-03-switch-on-event.md) | △ |

## まとめ

**Lite にパーティクルは実装されています**が、本家の命令的な `new ParticleSystem` とは作り方が違い、
**Node Particle Editor (NPE)** でグラフを組んでスニペットで読み込む形です（`parseNodeParticleSetFromSnippet` ＋ `registerNodeParticleSet`）。
噴水の器（回転体）は `createRibbon` で再現できます。この作り方の差があるため、両章とも判定は △ です。

> 動作確認済みサンプルは準備中です（NPE スニペットを用意して反映します）。
