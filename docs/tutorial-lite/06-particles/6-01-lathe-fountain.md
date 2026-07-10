# 6-01 旋盤で回された噴水 (A Lathe Turned Fountain) — ✕

> [第6部：パーティクル効果](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

Particle System 未対応に加え、**`CreateLathe`（回転体生成）も未実装**です。

- 回転体の**形状のみ**なら `ribbon` / `tube`（parallel-transport frames）で近似可能
- ただし本章の主目的（噴水パーティクル）は達成できないため ✕

```typescript
// 参考：回転体の“形状”だけなら tube 等で近似（噴水の水そのものは不可）
// 追加 import: createTube 等（引数は要確認）
```

---

← [6-00 パーティクル噴水 (Build a Particle Fountain)](./6-00-particle-fountain.md)
