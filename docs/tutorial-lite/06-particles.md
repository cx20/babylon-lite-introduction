# 第6部：パーティクル効果

> 共通テンプレート・凡例は [README](./README.md) を参照。

---

## 6-00 パーティクル噴水 (Build a Particle Fountain) — ✕

**Lite に Particle System はありません**（機能比較表：Particle System —）。CPU/GPU パーティクル、`ParticleSystem` 相当が未実装のため、噴水の粒子表現は再現できません。

代替の方向性（近似）：
- 少数の球メッシュ＋ thin instances を `onBeforeRender` で放物運動させる自前の簡易パーティクル
- ただし本格的なエミッタ/寿命/ブレンド制御は無く、労力に見合いません

> パーティクルが必須のシーンは、現状 Babylon.js 側が適しています。

---

## 6-01 旋盤で回された噴水 (A Lathe Turned Fountain) — ✕

Particle System 未対応に加え、**`CreateLathe`（回転体生成）も未実装**です。

- 回転体の**形状のみ**なら `ribbon` / `tube`（parallel-transport frames）で近似可能
- ただし本章の主目的（噴水パーティクル）は達成できないため ✕

```typescript
// 参考：回転体の“形状”だけなら tube 等で近似（噴水の水そのものは不可）
// 追加 import: createTube 等（引数は要確認）
```

---

## まとめ

第6部の 2 章は、Lite が意図的にスコープ外としている **パーティクル**に依存するため未対応です。
ゴール完成版（村を歩く Xbot）はパーティクルを使わないため影響ありません。
