# 5-02 頭上の空 (Skies Above) — ○（要アセット）

> [第5部：環境改善](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：スカイボックスを表示する。Babylon.js の `CubeTexture` + `SKYBOX_MODE` は、Lite では **`loadEnvironment`**（スカイボックス＋IBL 環境光）に集約されています。

追加 import：`loadEnvironment`

```typescript
await loadEnvironment(scene, "https://assets.babylonjs.com/environments/environmentSpecular.env", {
  brdfUrl: "https://<到達可能なホスト>/brdf-lut.png",   // ★必須：BRDF LUT の PNG
  skyboxUrl: "https://assets.babylonjs.com/environments/backgroundSkybox.dds",
  skyboxSize: 1000,
});
```

- `.env`（specular IBL）＋ `.dds` スカイボックス＋ BRDF LUT で、PBR マテリアルが正しく光ります。
- `brdfUrl` は**必須**。到達可能な URL を用意できない場合はスカイボックスを省略してください（背景色のみになります）。
- 6 枚 JPG のキューブスカイボックス（`SKYBOX_MODE`）も「Cubemap Skybox ✅」として対応していますが、`loadEnvironment` が最も手軽です。

---

← [5-01 遠くの丘 (Distant Hills)](./5-01-distant-hills.md) ・ [5-03 木のスプライト (Sprite Trees)](./5-03-sprite-trees.md) →
