# 2-02 サウンドを追加 (Adding Sound) — ○

> [第2部：村の構築](./README.md) ・ [全体の目次](../README.md)（共通テンプレート・凡例）

**目的**：効果音や BGM を鳴らす。

Lite には **AudioV2 ポートのオーディオエンジンがあります**。本家の `new BABYLON.Sound(...)` は、
`createAudioEngineAsync()` でエンジンを作り、`createSoundAsync(engine, url, options)` で読み込む形に置き換えます。

**Lite 移植時の注意点**：

- **オーディオ API は async** — `createAudioEngineAsync()` と `createSoundAsync()` はどちらも `await` します。
- **自動再生ポリシー** — ブラウザは、ユーザー操作があるまで音を鳴らさないことがあります。Lite のオーディオエンジンは
  既定で **`resumeOnInteraction: true`** なので、ページ（Playground）を一度クリックすれば以降は鳴ります。
- **音声ファイルは絶対 URL で** — 本家の `"sounds/bounce.wav"` は Playground 相対パスなので Lite では解決できません。

### 3 秒ごとに鳴らす

本家は `new BABYLON.Sound(...)` を作って `setInterval(() => s.play(), 3000)` で繰り返します。
Lite では `createSoundAsync` で読み込んだ音を `playSound` で鳴らします。

追加 import：`createAudioEngineAsync, createSoundAsync, playSound`

```typescript
const BOUNCE_URL = "https://playground.babylonjs.com/sounds/bounce.wav";

const audioEngine = await createAudioEngineAsync();
const bounce = await createSoundAsync(audioEngine, BOUNCE_URL);
setInterval(() => playSound(bounce), 3000);   // 3 秒ごとに再生
```

<iframe src="https://liteplayground.babylonjs.com/snippet/TLH0R6/v/1?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-02 サウンドを追加（3 秒ごと）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/TLH0R6/v/1
>
> 自動再生ポリシーのため、プレビューを一度クリックしてから 3 秒ごとの音を確認してください。

### BGM をループ自動再生する

本家 `new BABYLON.Sound("cello", url, scene, null, { loop: true, autoplay: true })` の `loop` / `autoplay` は、
Lite でも `createSoundAsync` の options にそのままあります。`autoplay: true` なら準備でき次第自動で鳴るので、
`setInterval` も明示的な `playSound` も要りません。

```typescript
const MUSIC_URL = "https://playground.babylonjs.com/sounds/cellolong.wav";

const audioEngine = await createAudioEngineAsync();
await createSoundAsync(audioEngine, MUSIC_URL, { loop: true, autoplay: true });   // ループ自動再生
```

<iframe src="https://liteplayground.babylonjs.com/snippet/TLH0R6/v/0?embed=runner&embedOrigin=https://cx20.github.io"
        title="Babylon Lite Playground: 2-02 サウンドを追加（ループ自動再生）"
        loading="lazy" allow="fullscreen"
        style="width: 100%; height: 480px; border: 0"></iframe>

> 動作確認済みサンプル（Lite Playground）: https://liteplayground.babylonjs.com/snippet/TLH0R6/v/0
>
> ほかに `pauseSound` / `resumeSound` / `stopSound` / `setSoundVolume`、ストリーミング音源（`createStreamingSoundAsync`）、
> **3D 空間定位（`SpatialSoundOptions` / `updateSpatialAudio`）** も揃っています。本家 `BABYLON.Sound` の spatial 相当も再現できます。

---

← [2-01 地面 (Grounding)](./2-01-grounding.md) ・ [2-03 メッシュを設置 (Place and Scale)](./2-03-place-and-scale.md) →
