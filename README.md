# Babylon Lite 入門 — Babylon.js チュートリアル移植集

[Babylon.js チュートリアル日本語版（chomado）](https://zenn.dev/chomado/books/babylonjs-tutorial-ja) の全 33 章を、
**[Babylon Lite](https://doc.babylonjs.com/lite/)（WebGPU 専用の軽量 3D エンジン）** で再現できるか検証し、
再現できたものを章別のサンプル集としてまとめたものです。

最終的に「**Xbot が村の道を歩く三人称シーン**」に到達します。

## 読む

**→ [章別サンプル集を読む](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/)**（GitHub Pages 版）

各章には **Lite Playground の実行プレビュー（iframe）を埋め込んで**あります。
プレビューが表示されるのは Pages 版だけです — GitHub のリポジトリ画面では `<iframe>` がサニタイズされて消えるため、
併記したスニペットのリンクから Playground を開いてください。

全 33 章を**章ごとのファイル**に分けています。各パートの索引から章を選んでください。

| パート | 章 | 内容 | GitHub で読む | プレビュー付き |
|---|:--:|---|---|---|
| 第1部 | 1-00〜1-04 | 基礎（Hello World / モデル読込 / セットアップ） | [01-basics/](./docs/tutorial-lite/01-basics/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/01-basics/) |
| 第2部 | 2-00〜2-11 | 村の構築（地面 / メッシュ / テクスチャ / コピー） | [02-village/](./docs/tutorial-lite/02-village/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/02-village/) |
| 第3部 | 3-00〜3-07 | アニメーション（親子 / 車 / キャラ歩行） | [03-animation/](./docs/tutorial-lite/03-animation/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/03-animation/) |
| 第4部 | 4-00〜4-01 | 衝突回避 | [04-collisions/](./docs/tutorial-lite/04-collisions/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/04-collisions/) |
| 第5部 | 5-00〜5-03 | 環境（丘 / 空 / スプライトの木） | [05-environment/](./docs/tutorial-lite/05-environment/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/05-environment/) |
| 第6部 | 6-00〜6-01 | パーティクル効果 | [06-particles/](./docs/tutorial-lite/06-particles/README.md) | [▶](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-lite/06-particles/) |

対応可否の全体像は **[tutorial-compatibility.md](./docs/tutorial-compatibility.md)**
（[プレビュー付き](https://cx20.github.io/babylon-lite-introduction/docs/tutorial-compatibility.html)）にまとめています。

## 対応可否のまとめ

導入章 5 つを除く 28 章の内訳です。

| 判定 | 章数 | 割合 | 主な内容 |
|---|:--:|:--:|---|
| ○ 対応可 | 20 | 71% | 造形・変換・マテリアル・ライト・カメラ・親子関係・スケルタルアニメ・地形・スカイボックス・オーディオ |
| △ 条件付き | 6 | 21% | `MergeMeshes` / `FollowCamera` / `movePOV` / box の `faceUV` などのヘルパー欠如を自作で回避 |
| ✕ 未対応 | 2 | 7% | パーティクル（6-00 / 6-01） |

村チュートリアルの中核はほぼ再現できます。詰まる 2 章はパーティクルに集約されます。

## 動作環境

- **WebGPU 対応ブラウザ**（Chrome / Edge 113+ など）
- 実行は [Babylon Lite Playground](https://liteplayground.babylonjs.com) が最速（インストール不要）
- アセットは Playground の相対パスでは解決しないため、すべて `https://` 絶対 URL で参照します

## ライセンス・出典

- 原典：[Babylon.js チュートリアル日本語版](https://zenn.dev/chomado/books/babylonjs-tutorial-ja)（chomado）／[Babylon.js Getting Started](https://doc.babylonjs.com/features/introductionToFeatures)
- 移植先：[Babylon Lite](https://github.com/BabylonJS/Babylon-Lite)
- アセットは `assets.babylonjs.com` / `playground.babylonjs.com` のものを参照しています
