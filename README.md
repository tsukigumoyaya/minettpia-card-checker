# MinettePia Card Checker

写真にフレーム（SR / R / N）を重ねてトリミングし、セリフを書き込んでカード画像を作成する Web アプリです。

## 使い方

1. **写真をアップロード** — ドラッグ＆ドロップ、またはファイル選択（JPEG / PNG / WebP、最大 20MB）
2. **フレームを選択** — 右パネルから SR / R / N を選択。フレームはトリミング画面にリアルタイムで重なります
3. **トリミング** — Cropper.js の操作でドラッグ・リサイズして 3:4 の範囲を調整
4. **セリフ入力**（SR フレーム時のみ） — テキストを入力するとフレーム下部にリアルタイム反映。フォントサイズはスライダーで調整可能
5. **テンプレート文言コピー**（SR フレーム時のみ） — 入力したセリフが埋め込まれた定型文をワンクリックでクリップボードにコピー
6. **ダウンロード** — 「フレーム付きでダウンロード」チェックで合成有無を選択し、1200×1600 の PNG を保存

## フレーム素材の差し替え

`public/frames/` に以下の PNG ファイルを配置してください（3:4、透過推奨）:

- `SR.png` — SR フレーム
- `R.png` — R フレーム
- `N.png` — N フレーム

フレーム一覧は `public/frames.json` で管理しています。

## 開発

```sh
npm install
npm run dev      # 開発サーバー起動
npm run build    # 本番ビルド（dist/ に出力）
npm run preview  # ビルド結果のプレビュー
```

## デプロイ

GitHub Pages 向けに設定済みです。`main` または `master` ブランチへの push で GitHub Actions が自動デプロイします。

## 技術スタック

- Vue 3（Composition API）
- Vite
- Tailwind CSS
- Cropper.js
- HTML Canvas API
