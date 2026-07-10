# 8-02 キャラを追う (Follow That Character) — △

> [第8部：世界の見方](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：歩くキャラクターをカメラが追従する。

**Lite 対応方針**（v1.8 ソースで確認）：

- 本家の **`FollowCamera` は Lite に無い**（該当 export なし）。3-07／旧ゴール版で確立した **`camera.parent = キャラのルート`**（親子付け）で背後追従を再現します。
- ピッチの下限は `setCameraLimits(camera, { upperBetaLimit })` で制限します。

> 動作確認済みサンプルは準備中です。

---

← [8-01 見回す (Have a Look Around)](./8-01-look-around.md) ・ [8-03 VR の世界へ (Going Virtual)](./8-03-going-virtual.md) →
