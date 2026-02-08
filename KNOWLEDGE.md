# Remotion + VOICEVOX 動画作成の知見

本ドキュメントは、実際の動画制作プロジェクト（海外の人から見た日本の驚く点を紹介する動画）での経験から得られた知見をまとめたものです。

---

## プロジェクト概要

- **テーマ**: 海外の人から見た日本の驚く点・おかしい点
- **形式**: ずんだもん×めたんの掛け合い解説動画
- **時間**: 約2分30秒（35セリフ）
- **トピック数**: 7つ（自動販売機、トイレ、電車、食品サンプル、コンビニ、お辞儀、現金主義）

---

## セットアップで遭遇した問題と解決策

### 1. TypeScriptファイル実行エラー

**問題**:
```
TypeError: Unknown file extension ".ts" for /home/yuta/Desktop/apps/ai/my-video/scripts/sync-settings.ts
```

**原因**: `tsconfig.json`が存在せず、ts-nodeがTypeScriptファイルを認識できない

**解決策**: プロジェクトルートに`tsconfig.json`を作成
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "jsx": "react",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true
  },
  "ts-node": {
    "compilerOptions": {
      "module": "commonjs"
    }
  },
  "include": ["src/**/*", "scripts/**/*"],
  "exclude": ["node_modules"]
}
```

**重要ポイント**:
- `ts-node`セクションで`commonjs`を指定することが重要
- テンプレートには本来含まれているべきファイルなので、欠損していた場合は追加が必要

---

## 正しいワークフロー

### 公式推奨のワークフロー

このテンプレートは**YAMLファースト**のアプローチを採用しています：

```
1. config/script.yaml を編集（セリフを書く）
   ↓
2. npm run voices（音声生成）
   ↓
3. npm run sync-script（durationsを反映）
   ↓
4. プレビュー確認（http://localhost:3000）
   ↓
5. npm run build（動画出力）
```

### ⚠️ やってはいけないこと

**NG**: `src/data/script.ts`を直接編集する
- このファイルは`config/script.yaml`から自動生成される
- 直接編集しても、`npm run sync-script`で上書きされる

**理由**: 
- YAMLファイルがソースコードの単一の真実（Single Source of Truth）
- TypeScriptファイルは生成物

### 正しい編集フロー

```yaml
# config/script.yaml を編集
- id: 1
  character: zundamon
  text: みんなこんにちは！ずんだもんなのだ！
  scene: 1
  pauseAfter: 10
  emotion: happy
  visual:
    type: text
    text: "海外の人が驚く\n日本の不思議"
    fontSize: 90
    color: "#FFD700"
    animation: zoomIn
```

---

## 音声生成のポイント

### VOICEVOXの確認

音声生成前に必ずVOICEVOXが起動しているか確認：
```bash
curl -s http://localhost:50021/version
```

成功すれば `"0.14.6"` のようなバージョンが返る。

### 音声生成の実行

```bash
npm run voices
```

このコマンドは以下を実行：
1. `scripts/generate-voices.ts`を実行
2. VOICEVOX APIで各セリフの音声を生成
3. `public/voices/`に`.wav`ファイルを保存
4. `public/voices/durations.json`に音声の長さを保存
5. 自動的に`npm run sync-script`を実行してdurationsを反映

### 処理時間の目安

- **1セリフあたり約2〜3秒**
- **35セリフの場合、約1.5〜2分**

進捗は以下のように表示される：
```
Generating: 01_zundamon.wav - "みんなこんにちは！ずんだもんなのだ！..."
  -> 2.93s, 106 frames
Generating: 02_metan.wav - "四国めたんです。今日は海外の人から見た..."
  -> 5.34s, 193 frames
```

---

## コンポジションの管理

### 複数のコンポジションを作成する方法

`src/Root.tsx`で複数のコンポジションを定義可能：

```typescript
import React from "react";
import { Composition } from "remotion";
import { Main } from "./Main";
import { scriptData } from "./data/script";
import { VIDEO_CONFIG } from "./config";

const calculateTotalFrames = () => {
  let total = 60;
  for (const line of scriptData) {
    total += line.durationInFrames + line.pauseAfter;
  }
  total += 60;
  return total;
};

export const RemotionRoot: React.FC = () => {
  const totalFrames = calculateTotalFrames();

  return (
    <>
      <Composition
        id="Main"
        component={Main}
        durationInFrames={totalFrames}
        fps={VIDEO_CONFIG.fps}
        width={VIDEO_CONFIG.width}
        height={VIDEO_CONFIG.height}
      />
      <Composition
        id="JapanSurprises"
        component={Main}
        durationInFrames={totalFrames}
        fps={VIDEO_CONFIG.fps}
        width={VIDEO_CONFIG.width}
        height={VIDEO_CONFIG.height}
        defaultProps={{
          title: "海外の人が驚く日本の不思議"
        }}
      />
    </>
  );
};
```

**ポイント**:
- Reactのインポートを忘れずに追加（`import React from "react";`）
- 各コンポジションには一意の`id`を指定
- 同じ`Main`コンポーネントを使い回せる
- `defaultProps`でタイトルなどを変更可能

---

## 5分程度の動画を作るセリフ数の目安

### 今回の実績データ

- **セリフ数**: 35個
- **実際の長さ**: 約2分30秒（150秒）
- **平均1セリフの長さ**: 約4.3秒

### 計算式

```
5分（300秒）の動画を作るには:
300秒 ÷ 4.3秒/セリフ ≈ 70セリフ
```

### トピック構成の目安

5分動画のトピック構成例：
- オープニング: 2〜3セリフ
- メイントピック: 10〜15個（各5〜6セリフ）
- エンディング: 3〜4セリフ
- **合計: 65〜80セリフ**

---

## コンテンツ構成のベストプラクティス

### 効果的なトピック選定

**良い例**（今回の動画）:
- 🤖 自動販売機天国（具体的な数字: 500万台）
- 🚽 ハイテクトイレ（具体的機能: ウォシュレット、暖房、音姫）
- 🚄 時刻表は秒単位（具体的な数字: 平均遅延18秒）
- 🍜 超リアルな模型（視覚的に分かりやすい）
- 🏪 24時間なんでも屋（機能列挙: ATM、宅配、支払い）
- 🙇 お辞儀いろいろ（具体的な角度: 15度、30度、45度）
- 💴 キャッシュ大国（対比: 海外のカード社会）

**ポイント**:
- 具体的な数字を入れる
- 比較・対比を使う
- 絵文字で視覚的に分かりやすく
- 外国人の視点を入れる

### セリフの掛け合いパターン

```
パターン1: 紹介→驚き
ずんだもん: 「〇〇なのだ！」（紹介）
めたん: 「すごいね」（驚き）

パターン2: 説明→補足
ずんだもん: 「△△なのだ！」（メイン説明）
めたん: 「□□なんだね」（補足説明）

パターン3: 疑問→解説
めたん: 「なぜ〇〇なの？」（疑問）
ずんだもん: 「△△だからなのだ！」（解説）
```

---

## visualプロパティの活用

### テキストビジュアル

トピックの変わり目で大きなタイトルを表示：

```yaml
visual:
  type: text
  text: "🤖 自動販売機天国"
  fontSize: 80
  color: "#FF6B6B"
  animation: slideUp
```

**色の使い分け**:
- ゴールド（#FFD700）: オープニング・まとめ
- カラフル: 各トピック（#FF6B6B, #4ECDC4, #95E1D3, #FFB6C1, #FFA07A, #DDA0DD, #98D8C8）

### アニメーションの種類

- `zoomIn`: インパクトのある登場
- `slideUp`: 下から上へ
- `slideLeft`: 右から左へ
- `fadeIn`: ゆっくり表示
- `bounce`: 跳ねる動き（楽しい雰囲気）

---

## ファイル構造の理解

### 重要なファイルとその役割

```
プロジェクト/
├── config/
│   ├── script.yaml          # ★ この編集 - セリフの定義
│   ├── characters.yaml      # キャラクター設定
│   └── defaults.yaml        # デフォルト値
├── src/
│   ├── data/
│   │   └── script.ts        # ✗ 編集禁止 - YAMLから生成
│   ├── settings.generated.ts # ✗ 自動生成
│   └── Root.tsx             # コンポジション定義
├── public/
│   └── voices/
│       ├── 01_zundamon.wav  # 音声ファイル
│       ├── 02_metan.wav
│       └── durations.json   # 音声の長さ情報
├── video-settings.yaml      # スタイル設定
└── tsconfig.json            # TypeScript設定（必須）
```

---

## トラブルシューティング

### 音声と字幕が一致しない

**原因**: 古い音声ファイルが残っている

**解決策**:
1. `config/script.yaml`を確認
2. `npm run voices`で音声を再生成
3. `npm run sync-script`でdurationsを反映（自動実行される）

### プレビューに変更が反映されない

**原因**: ブラウザのキャッシュまたは同期の失敗

**解決策**:
1. ブラウザをリロード（Ctrl+R / Cmd+R）
2. `npm run sync`を手動で実行
3. `npm start`を再起動

### 動画の長さが予想と違う

**確認ポイント**:
- `durationInFrames`が正しく設定されているか
- `pauseAfter`の合計時間
- `calculateTotalFrames()`の計算ロジック

---

## パフォーマンス最適化

### 音声生成の高速化

**既存ファイルをスキップ**（オプション）:
`scripts/generate-voices.ts`の以下のコメントを外す：
```typescript
// 既存ファイルがあればスキップ（オプション）
if (fs.existsSync(outputPath)) {
  console.log(`Skip: ${line.voiceFile} (already exists)`);
  continue;
}
```

**注意**: セリフを変更した場合は削除してから再生成が必要

### メモリ使用量の削減

大量のセリフがある場合：
- セリフを複数のシーンに分割
- 不要なvisualを削除
- 画像サイズを最適化

---

## 本番環境での注意点

### 動画出力時のリソース

```bash
npm run build
```

**所要時間の目安**:
- 2分30秒の動画: 約5〜10分
- 5分の動画: 約10〜20分
- CPUとメモリを大量に消費

**推奨環境**:
- RAM: 8GB以上
- CPU: マルチコア推奨
- ストレージ: 十分な空き容量（動画は数百MB）

---

## 今後の改善案

### 1. スクリプト生成の自動化

LLMを使ってトピックから自動的にYAMLを生成：
```
入力: 「AIの歴史を解説する動画」
↓
出力: config/script.yaml（完全版）
```

### 2. ビジュアル素材の自動収集

- Webスクレイピング
- AI画像生成
- ストックフォト連携

### 3. 多言語対応

- 英語字幕の自動生成
- 翻訳APIとの連携
- VOICEVOXの多言語版

---

## まとめ

### 成功のポイント

1. **YAMLファーストのワークフロー**を守る
2. **VOICEVOX起動確認**を忘れずに
3. **具体的な数字と比較**でコンテンツを充実
4. **visualとアニメーション**で視覚的魅力を追加
5. **トピック数と総セリフ数**のバランスを考える

### 学んだこと

- テンプレートには欠損ファイル（tsconfig.json）がある可能性
- 音声生成は時間がかかる（35セリフで約2分）
- 5分動画には約70セリフが必要
- コンポジションは複数作成可能
- YAMLとTypeScriptの役割分担を理解することが重要

---

## 参考リンク

- テンプレート公式: https://github.com/nyanko3141592/remotion-voicevox-template
- 使い方ガイド: https://note.com/electrical_cat/n/n9be53194cb5c
- VOICEVOX: https://voicevox.hiroshiba.jp/
- Remotion: https://www.remotion.dev/

---
