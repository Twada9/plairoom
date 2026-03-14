# AI生成サービス選定レポート

> 作成日：2026-03-14

---

## 結論（推奨）

| コンテンツ | 推奨サービス | 理由 |
|---|---|---|
| 画像生成 | **Hugging Face Inference API（FLUX.1-dev）** | 無料枠あり・品質高・Edge Functionsから呼びやすい |
| 音楽生成 | **MusicGen（Hugging Face Inference Endpoints）** | 無料枠あり・公式API・ただしライセンス注意 |

---

## 画像生成サービス比較

### 1. Hugging Face Inference API — FLUX.1 / Stable Diffusion XL

| 項目 | 内容 |
|---|---|
| 無料枠 | 毎月のクレジット付与あり（量は非公開） |
| 超過後の料金 | GPU稼働時間 × 単価（例：10秒 × $0.00012 = $0.0012/枚） |
| モデル品質 | FLUX.1-devは現行トップクラス |
| 制限 | 無料枠はFLUX.1のような重いモデルで消耗が早い |
| 公式ページ | https://huggingface.co/pricing |

**メリット：** Edge FunctionsからHTTP呼び出し可。モデルを選べる柔軟性。
**デメリット：** 無料枠の消耗が早く、多ユーザーのSNSでは早期に有料化が必要。

---

### 2. Replicate API

| 項目 | 内容 |
|---|---|
| 無料枠 | ごく少量のテスト実行のみ |
| 料金 | GPU秒課金。約 $0.002〜$0.003 / 1024×1024画像 |
| モデル品質 | FLUX.1、Recraft V3 など高品質モデルを多数ホスト |
| 制限 | ほぼ従量課金のみ（無料枠が非常に少ない） |
| 公式ページ | https://replicate.com/pricing |

**メリット：** 安定した従量課金・ドキュメントが充実。
**デメリット：** 実質的に無料で使えるフェーズが短い。

---

### 画像生成：開発フェーズ別推奨

| フェーズ | 推奨 | 理由 |
|---|---|---|
| 開発・テスト | Hugging Face（無料枠） | 費用ゼロで始められる |
| プロダクション（小規模） | Hugging Face PRO（$9/月） | 20倍のクレジット |
| プロダクション（スケール） | Replicate（従量課金） | ユーザー数に応じて線形スケール |

---

## 音楽生成サービス比較

### 1. MusicGen（Hugging Face）

| 項目 | 内容 |
|---|---|
| 無料枠 | Inference API 経由で無料枠あり（他モデルと共有） |
| 料金 | 超過後は GPU秒課金（画像と同様） |
| 最大生成時間 | **30秒**（ハードリミット） |
| モデルサイズ | small（300M）/ medium（1.5B）/ large（3.3B） |
| ライセンス | コード：MIT、**モデル重み：CC-BY-NC 4.0（商用利用不可）** |
| 公式ページ | https://huggingface.co/facebook/musicgen-large |

⚠️ **重要：** CC-BY-NC 4.0 のため、商用プロダクトでは利用不可。
個人利用・非商用のMVP段階では使用できるが、課金ユーザー向け機能（プレミアムプラン）には使えない可能性がある。

---

### 2. Suno API

| 項目 | 内容 |
|---|---|
| 公式API | **2026年現在も一般公開APIなし**（招待制ベータのみ） |
| サードパーティ経由 | 1曲あたり約 $0.02〜$0.05（中間業者のラッパーAPI） |
| 無料枠 | ウェブアプリで50クレジット/日（≒10曲/日）。個人利用のみ |
| 商用利用 | Proプラン（$10/月）以上が必要 |
| 注意 | WarnerMusicとの訴訟和解（2025年末）により利用規約が厳格化 |
| 公式ページ | https://suno.com/pricing |

**メリット：** 出力品質が高く、ボーカル付きの楽曲生成も可能。
**デメリット：** 公式APIなし・サードパーティ経由のみ・不安定・高コスト。

---

### 音楽生成：開発フェーズ別推奨

| フェーズ | 推奨 | 注意点 |
|---|---|---|
| 開発・テスト（非商用） | MusicGen（Hugging Face） | CC-BY-NC で商用不可 |
| プロダクション（商用） | MusicGen の商用対応モデルを探す、または AudioCraft の代替（Stable Audio など） | 要再調査 |
| 音質最優先 | Suno（サードパーティ経由） | 安定性・コストリスクあり |

---

## 実装上の注意点

### Supabase Edge Functions からの呼び出し

```typescript
// generate-image Edge Function のイメージ
const response = await fetch(
  "https://api-inference.huggingface.co/models/black-forest-labs/FLUX.1-dev",
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${Deno.env.get("HF_API_TOKEN")}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ inputs: fullPrompt }),
  }
);
```

- HF のトークンは Supabase の **Secrets** で管理する
- 生成は非同期（数秒〜数十秒かかる）→ Edge Function のタイムアウト（デフォルト150秒）に注意
- 生成完了後に `image_contents.status` を `generating → completed / failed` に更新する

---

## まとめ・次のアクション

1. **画像生成**：まず **Hugging Face FLUX.1-dev** で実装を進める
2. **音楽生成**：**MusicGen** をMVP段階で使用し、商用化フェーズで代替サービスを再選定する
3. `generate-image` / `generate-music` の **Edge Function 実装を開始** する
4. Hugging Face の API トークンを取得し、Supabase Secrets に登録する

---

## 参考リンク

- [Hugging Face Pricing](https://huggingface.co/pricing)
- [Hugging Face Inference Providers Pricing](https://huggingface.co/docs/inference-providers/en/pricing)
- [Replicate Pricing](https://replicate.com/pricing)
- [MusicGen (Hugging Face)](https://huggingface.co/facebook/musicgen-large)
- [MusicGen を API として動かす](https://huggingface.co/blog/run-musicgen-as-an-api)
- [Suno Pricing](https://suno.com/pricing)
