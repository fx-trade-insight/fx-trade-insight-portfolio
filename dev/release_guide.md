# リリース・バージョンアップ手順

GitHub: https://github.com/fx-trade-insight/fx-trade-insight

---

## バージョン番号のルール（セマンティックバージョニング）

```
1.2.3
│ │ └─ パッチ：バグ修正・軽微な修正
│ └─── マイナー：後方互換のある新機能追加
└───── メジャー：破壊的変更・大幅なリニューアル
```

---

## 1. 開発フロー（ブランチ戦略）

### 小さい修正（バグ修正・UI調整など）
main に直接コミットしてよい。

```bash
git add .
git commit -m "fix: カレンダーの日付ズレを修正"
git push origin main
```

### 新機能・大きな変更
feature ブランチを切って作業し、完成したら main にマージする。

```bash
# ブランチを作成して切り替え
git checkout -b feature/新機能名

# 作業・コミットを繰り返す
git add .
git commit -m "feat: 〇〇機能を追加"

# main に戻してマージ
git checkout main
git merge feature/新機能名

# ブランチを削除（後片付け）
git branch -d feature/新機能名

git push origin main
```

---

## 2. リリース前の作業

### 2-1. バージョン番号を上げる

`package.json` の `version` を変更する。

```json
{
  "version": "1.1.0"
}
```

### 2-2. AboutPage の更新履歴に追記する

`src/renderer/src/pages/AboutPage.tsx` の `changelog` 配列に追加する（新しいものを先頭に）。

```tsx
const changelog: ChangelogEntry[] = [
  {
    version: '1.1.0',
    date: '2026-XX-XX',
    changes: [
      '〇〇機能を追加',
      '〇〇のバグを修正',
    ],
  },
]
```

### 2-3. 変更内容をコミット

```bash
git add package.json src/renderer/src/pages/AboutPage.tsx
git commit -m "chore: v1.1.0 リリース準備"
git push origin main
```

### 2-4. GAS スクリプトプロパティを更新する

アプリ起動時にバージョンチェックを行い、古いバージョンのユーザーにバナー通知する仕組みがある。
ビルド・配布後に GAS のスクリプトプロパティを更新すること。

1. [GAS エディタ](https://script.google.com/) を開く
2. 「プロジェクトの設定」→「スクリプト プロパティ」を開く
3. 以下のプロパティを更新する

| プロパティ名 | 値の例 | 説明 |
|---|---|---|
| `LATEST_VERSION` | `1.1.0` | リリースした最新バージョン番号 |

> ダウンロードURLはアプリ内定数（GitHub Releases `/latest`）のため、GASへの設定は不要。
>
> **注意**：`LATEST_VERSION` の更新はビルド・配布が完了してから行うこと。更新した瞬間に古いバージョンのユーザー全員にバナー通知が表示される。

---

## 3. ビルド

```bash
npm run build:win
```

`dist/` フォルダに `FxTradeInsight-Setup-x.x.x.exe` が生成される。

> **事前確認**：`resources/ffmpeg/ffmpeg.exe` が存在することを確認すること（ないとビルドエラーになる）。

---

## 4. テスト（クリーン環境）

VirtualBox を使ったテスト手順は [`docs/dev/virtualbox_test_guide.md`](./virtualbox_test_guide.md) を参照。

最低限確認すること：
- SmartScreen 警告が出て「詳細情報」→「実行」でインストールできるか
- ライセンス認証が通るか
- 主要機能（CSV取込・録画・設定）が動くか
- About 画面のバージョンが正しいか

テスト後は VirtualBox のマシン ID をスプレッドシートから削除すること。

---

## 5. VirusTotal スキャン

ビルドした `.exe` を [virustotal.com](https://www.virustotal.com) にアップロードしてスキャンする。
スキャン結果のURLをコピーしておき、GitHub Releases の説明欄・Note の販売ページに掲載する。

> 証明書なしの期間は誤検知が出やすい。掲載することでユーザーへの信頼材料になる。

---

## 6. GitHub にリリースを作成

### 6-1. タグを打つ

```bash
git tag v1.1.0
git push origin v1.1.0
```

### 6-2. GitHub Releases を作成

1. https://github.com/fx-trade-insight/fx-trade-insight/releases/new を開く
2. 「Choose a tag」で `v1.1.0` を選択
3. タイトル：`v1.1.0`
4. 説明欄に変更内容と VirusTotal スキャン結果を記載

```markdown
## 変更内容

- 〇〇機能を追加
- 〇〇のバグを修正

## セキュリティ

VirusTotal スキャン結果（0/72）：https://www.virustotal.com/gui/file/...
```

5. `FxTradeInsight-Setup-1.1.0.exe` をアップロード
6. 「Publish release」をクリック

---

## 7. 販売ページの更新

### Note
1. Note の有料記事を開いて編集モードに入る
2. バージョン表記・変更内容の記載を更新する
3. セキュリティセクションの VirusTotal スキャン結果URLを更新する
4. ダウンロードリンク（GitHub Releases `/latest`）は固定のため更新不要

---

## 8. GAS の再デプロイが必要な場合

`outside_function/gas/license_system/` に変更を加えた場合は再デプロイが必要。

手順は [`docs/dev/clasp_guide.md`](./clasp_guide.md) を参照。

---

## 9. Electron のバージョンアップ

### なぜ必要か

Electron は **最新 3 メジャーバージョンのみ**をセキュリティサポートする。サポート外になると Chromium・Node.js・V8 の CVE が修正されなくなる。

このアプリは外部 Web コンテンツを読み込まないローカルアプリのため、脆弱性が出ても実害につながるケースはほぼない。ただし放置しすぎるとバージョンが離れて一気に上げるのが大変になるため、**1〜2 年に 1 回**程度を目安に更新しておくと楽。

### アップグレード頻度の目安

Electron は約 6 週間ごとにメジャーバージョンをリリースする。3 バージョン前がサポート切れになるので、最新から 2 バージョン以内を目標に更新する。

### 手順

**1. バージョンを確認する**

```bash
# 現在のバージョン確認
npm list electron

# 利用可能なバージョン確認（最新安定版）
npm show electron version
```

**2. `package.json` を更新する**

```json
"electron": "^41.0.0",
"electron-builder": "^26.0.0"
```

electron-builder は Electron のメジャーに合わせて最新版に上げる。

**3. better-sqlite3 も最新版にする**

native module のプリビルドバイナリが新しい Electron バージョンに対応していることが多い。

```json
"better-sqlite3": "^12.10.0"
```

**4. インストールして確認**

```bash
npm install
npm run build
```

`better-sqlite3` のビルドに失敗した場合は VS Build Tools（ビルド環境にインストール済み）でソースからビルドされる。

### よくある問題

| 問題 | 原因 | 対処 |
|------|------|------|
| `better-sqlite3` がビルドエラー | 新 Electron の V8 API に未対応 | 一世代前の Electron を使う、または better-sqlite3 の新バージョンを待つ |
| `Could not find Visual Studio` | VS Build Tools が未インストール | `winget install Microsoft.VisualStudio.2022.BuildTools` |
| `npm audit` で新たな高リスク CVE | Electron 本体の脆弱性 | Electron をさらに新しいバージョンへ |

### アップグレード後の確認事項

```
[ ] npm run build でビルド成功
[ ] npm run dev でアプリが起動する
[ ] 録画・再生が正常に動作する
[ ] VirtualBox でインストールテスト
```

---

## チェックリスト（リリース時）

```
[ ] package.json のバージョンを更新
[ ] AboutPage の changelog に追記
[ ] git commit & push
[ ] npm run build:win でビルド成功
[ ] VirtualBox でインストール・認証テスト
[ ] VirusTotal スキャン実施・結果URL取得
[ ] git tag & push
[ ] GitHub Releases を作成・exe をアップロード
[ ] GitHub Releases の説明欄に VirusTotal スキャン結果URLを記載
[ ] Note の商品説明文のバージョン表記と VirusTotal URLを更新
[ ] GAS スクリプトプロパティの LATEST_VERSION を更新
[ ] （GAS コードに変更があれば）clasp push & 再デプロイ
```
