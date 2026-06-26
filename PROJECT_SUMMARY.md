# 79BOATRACE LIVE ― プロジェクト概要・制作まとめ

iPhoneで「ボートレースのレース映像」を見ながら「別配信の解説（YouTube）の音」を同時に聴くための、
1ファイル完結の軽量Webアプリ。

- 公開URL: https://live.79boatrace.com
- リポジトリ: `79boatrace/79boatrace-live`（GitHub Pages, default `main`, pushで自動デプロイ）
- 本体: `index.html` 1枚（外部依存はYouTube IFrame APIのみ）
- ローカル作業クローン: `C:\Users\USER\79boatrace-live`
- 正本メモ（開発用）: リポジトリ内 `project_race_pip_viewer.md`

---

## 1. 何ができるアプリか
- **PC / iPad**: レース映像とYouTubeを上下（横持ちは左右）の**2分割**で同時再生。
- **iPhone**: 端末の制約に合わせた**soloモード**。
  - レース映像を**全画面**で表示。
  - 解説（YouTube）は**音声だけ**を裏で再生。
  - `🔊音声切替` で「解説の音」↔「ライブ映像の音」をワンタップ切替。
  - `−／＋` で解説のラグ（遅延）を調整し、映像と音のズレを合わせる。
- 設定（レース場・YouTube URL・表示/音声）は**共有リンク**化でき、他人に同じ構成を渡せる。

## 2. 画面・操作の構成
- 2つのソース枠 **A / B**。各枠は `レース映像(boatcast)` か `YouTube` を選択。
  - 既定: A=レース映像（鳴門）, B=YouTube（解説）。
- レース映像は公式CDNプレイヤー（boatcast, DVR対応）を iframe 埋め込み。
- YouTube は IFrame Player API で制御（mute/unMute・seek 等）。
- ツールバー: ⚙設定 / 🔊音声切替 / ラグ −・＋ / ⇅入替 / 🔗共有。

## 3. iPhone(solo)モードの設計ポイント（肝）
iPhone Safari には「ユーザー操作（タップ）内でしか音を出せない」「裏の映像を止めやすい」等の制約がある。
それを踏まえた設計：
- 見る映像（visual）を全画面表示し、音声ソースのYouTubeは
  **画面外に隠さず**「全画面サイズ・透明・最背面（opacity:0）」で**生かしたまま**にする
  （`display:none` にすると止まるため使わない）。
- 音声切替は iframe を作り直さず、**既存プレイヤーの mute/unMute だけ**で切り替える＝高速・安定。
- 最初の音出しは `🔊解説の音を出す` ボタンのタップ内で `playVideo()+unMute()+setVolume()` を実行
  （iOSのgesture制限対策）。
- ラグ合わせは YouTube のDVRを `seekTo(ライブ位置 − lag秒)` でシーク。

> ※ 実機検証の結果、この「2本を生かして mute/unMute で切替」方式は **iPhoneで期待通り動作**することを確認済み。

## 4. 制作の経緯（時系列ダイジェスト）
1. **2画面ビューア**として公開（レース映像＋YouTubeの2分割）。
2. **iPhone soloモード**追加（映像1本＋音声だけ別ソース＋ラグ同期）。
3. **iOS音声対策**追加（タップで音解錠＋YouTubeを生かしっぱなしにして即切替）。
4. **「iPhoneで片方しか流れない／切替が効かない」と実機で発覚** → 原因調査。
5. **真因はHTTPS証明書エラー**と判明（後述）。音声切替ロジック自体は正常だった。
6. 証明書を再発行 → **Enforce HTTPS 有効化** → 正常化。
7. 調査用に **デバッグ表示（`?debug=1`）** を実装してデプロイ。

## 5. つまずきと解決：HTTPS証明書エラー（重要な学び）
- 症状: iPhoneで `https://live.79boatrace.com` が
  **「接続はプライベートではありません（なりすまし）」** 警告。サイトがまともに開けていなかった。
- 調査:
  - DNSは正常（live → 79boatrace.github.io → GitHubの正規IP 185.199.108–111.153）。
  - `http://` なら 200 で開ける。
  - GitHub Pages API が `https_enforced:false` かつ **証明書オブジェクト無し**
    ＝ **Let's Encrypt 証明書が未発行**。上流が既定の `*.github.io` 証明書を返し、
    `live.79boatrace.com` と一致せず Safari が警告していた。
- 解決:
  1. Pages のカスタムドメインを**一旦解除→再登録**（API: `PUT /repos/{owner}/{repo}/pages` の `cname`）。
     → これで証明書発行がトリガーされ `https_certificate.state = approved` に。
  2. **Enforce HTTPS を ON**（`https_enforced:true`）。`http://` は 301 で `https://` へ転送。
- 教訓: 「アプリの挙動がおかしい」の前に、**そもそも安全に開けているか（証明書/HTTPS）**を疑う。
  GitHub Pages カスタムドメインは、**ドメインの入れ直しで証明書を再発行**できる。

## 6. デバッグ機能（`?debug=1`）
- 通常URL（`https://live.79boatrace.com`）では非表示。**通常ユーザーには一切見えない。**
- `https://live.79boatrace.com/?debug=1` で開くと、左上に小さく状態を常時表示：
  - モード（iPhone/PC）／音声（解説・ライブ／armed／live）／visual／lag
  - A・Bの再生状態（YouTube: 再生中/停止・音あり/消音・現在t/duration、映像: iframe有無）
  - 解説音ONタップ 済/未／最後に実行した操作（unMute+play / mute / seekTo / プレイヤ無 / ERR…）
- 使い方: 不具合時は `?debug=1` のスクショを見れば、
  「音が出ない」のか「映像が止まる」のかなどを即切り分けられる。

## 7. 現在のステータス
- ✅ HTTPS 正常（証明書 approved、Enforce HTTPS 有効、http→https 301）
- ✅ iPhone でレース映像＋解説音の同時、映像/音声切替、ラグ調整が動作（実機確認済み）
- ✅ デバッグ版デプロイ済み
- 正本 `project_race_pip_viewer.md` をリポジトリに同梱（引き継ぎ用）

## 8. 運用・デプロイ手順（メモ）
- 編集: `C:\Users\USER\79boatrace-live\index.html`
- 反映: コミット → `git push origin main`（GitHub Pages が自動ビルド・公開）
- 証明書/HTTPSは GitHub Pages 任せ（Let's Encrypt 自動更新）。
  もし証明書がおかしくなったら Settings→Pages でカスタムドメインを入れ直す。

## 9. 今後の改善アイデア（任意）
- iPhoneで `⇅入替` 実行時に両ソースを作り直すため一瞬止まる点の最適化。
- ラグの現在値・同期状態のUI表示強化（debug以外でも軽く見せる）。
- レース場プリセットやお気に入りYouTubeの保存。
- PWA（ホーム画面追加）まわりの体験調整。

---

### 主要ファイル早見表
| 役割 | パス |
|---|---|
| アプリ本体 | `index.html` |
| カスタムドメイン定義 | `CNAME`（`live.79boatrace.com`） |
| 開発用 正本メモ | `project_race_pip_viewer.md` |
| この概要 | `PROJECT_SUMMARY.md` |
