# 7-03 昼から夜へ (Day to Night) — △

> [第7部：光と影](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：スライダーでライトの強度を変え、昼夜を切り替える。

**Lite 対応方針**（v1.8 ソースで確認）：

- ライトの強度をアニメ／コードで変えること自体は ○ です（`onBeforeRender` や Property Animation）。
- ただし本家が使う **GUI（`AdvancedDynamicTexture` / スライダー）は Lite に無い**（公開 API に該当 export なし）。UI が要る場合は、canvas に重ねた **HTML の `<input type="range">`** など DOM 側で作り、値をライト強度へ渡すのが代替です。

> 動作確認済みサンプルは準備中です。

---

← [7-02 影を追加 (Adding Shadows)](./7-02-shadows.md) ・ [8-00 世界の見方 (Ways to See The World)](../08-cameras/8-00-cameras-intro.md) →
