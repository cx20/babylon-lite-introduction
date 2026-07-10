# 7-02 影を追加 (Adding Shadows) — ○

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：ライトに影を落とさせる。

**Lite 対応方針**（v1.8 ソースで確認）：

- シーンを `registerSceneWithShadowSupport` で登録し、`createEsmDirectionalShadowGenerator` / `createPcfDirectionalShadowGenerator` / `createPcfSpotlightShadowGenerator` / `createCsmDirectionalShadowGenerator` のいずれかで影ジェネレータを作ります。
- 影を落とすメッシュは `setShadowTaskCasterMeshes` で指定します。
- 本家の `ShadowGenerator` ＋ `shadowGenerator.getShadowMap().renderList` に相当します。

> 動作確認済みサンプルは準備中です。

---

← [7-01 ライトを灯す (Light the Night)](./7-01-lights.md) ・ [7-03 昼から夜へ (Day to Night)](./7-03-day-to-night.md) →
