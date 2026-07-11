# 7-03 影を追加 (Adding Shadows) — ○

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：ライトに影を落とさせる。

**Lite 対応方針**（v1.8 ソースで確認）：

- シーンを `registerSceneWithShadowSupport` で登録し、`createEsmDirectionalShadowGenerator` / `createPcfDirectionalShadowGenerator` / `createPcfSpotlightShadowGenerator` / `createCsmDirectionalShadowGenerator` のいずれかで影ジェネレータを作ります。
- 影を落とすメッシュは `setShadowTaskCasterMeshes` で指定します。
- 本家の `ShadowGenerator` ＋ `shadowGenerator.getShadowMap().renderList` に相当します。

> 動作確認済みサンプルは準備中です。

---

← [7-02 昼から夜へ (Day to Night)](./7-02-day-to-night.md) ・ [8-00 世界の見方 (Ways to See The World)](../08-cameras/8-00-cameras-intro.md) →
