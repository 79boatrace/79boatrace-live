# project_race_pip_viewer（正本メモ）

> このファイルが正本。ChatGPT内部メモリより優先する。作業のたびにここを更新する。

## 何のプロジェクトか
- iPhone向け「レース映像 + 解説YouTube」同時視聴アプリ。
- 公開URL: https://live.79boatrace.com （GitHub Pages, CNAME）
- リポジトリ: `79boatrace/79boatrace-live`（デフォルトブランチ `main`）
- 本体は単一ファイル `index.html`（外部依存はYouTube IFrame APIのみ）。
- ローカル作業用クローン: `C:\Users\USER\79boatrace-live`
- デプロイ: `main` に push → GitHub Pages が自動反映。

## 画面構成（index.html）
- 2スロット A / B。各スロットは `boatcast`(レース映像) または `youtube`。
- 既定: A=レース映像(boatcast/鳴門), B=YouTube(解説)。`visual=A`, `audio=B`, `layout=auto`。
- レイアウト
  - PC/iPad: 2分割（上下、横持ちは左右）。
  - iPhone(`/iPhone|iPod/`): `solo`モード = 映像1本を全画面 + もう片方は裏で再生（音声用）。

## iPhone(solo)モードの設計意図
- `visual` スロット = 全画面表示（見る映像）。
- 音声ソース(`audio`)のYouTubeは **画面外ではなく** `position:absolute; inset:0; opacity:0; pointer-events:none`
  で **全画面サイズ・透明・最背面** に置き、iOSが裏で止めないようにしている（`body.solo .slot.solo-hidden`）。
- iOSは「ユーザー操作(gesture)内」でしか `unMute()` を許さないため、
  YouTube iframe は破棄せず生かしっぱなしにし、`🔊解説の音を出す`(#unlock) / `🔊音声切替`(flipAudio)
  のタップ内で `unMute()+setVolume(100)+playVideo()` を実行する設計。
- ラグ合わせ: 解説(YouTube)が音声のとき、`−/＋` で `cfg.lag` 秒だけライブedgeから遅らせて `seekTo`。

## 主要関数（index.html）
- `effectiveSolo()` … iPhone or layout=solo
- `slotRole(slot)` … solo時に render/hidden を決定。audioソースのYTは hidden で生存
- `renderSlot/renderAll` … iframe/プレイヤ生成。**音声切替では呼ばない**（再生成で止まるため）
- `applyAudio()` … 既存プレイヤに対し mute/unMute のみ。タップから同期到達が必須
- `armAudio()` … #unlock タップ。audioArmed=true にして applyAudio
- `flipAudio()` … 解説↔ライブ 切替（audioをA↔B）
- `applyLag()/nudgeLag()` … YTのDVRシーク

## 既知のiOS制約（重要・要検証）
- iPhoneは歴史的に「同時に音付きで再生できる動画は実質1本」。
  裏のYouTubeを音声目的で生かしても、iOSから見れば **動画2本同時** であり、
  片方再生でもう片方が一時停止される可能性が高い。これが今回の不具合の最有力仮説。
  - もし「解説音は出るが**レース映像が止まる**」→ この制約に該当（分岐B）。
  - もし「**解説音が出ない**」→ gesture到達 or プレイヤ未readyの問題（分岐A）。

## ★解決(2026-06-26): 真因はHTTPS証明書、音声切替ロジックは正常
- 「iPhoneで片方しか流れない/切替が効かない」の**真因は音声ロジックではなくTLS証明書エラー**だった。
  - 症状の元: iPhone Safariで `https://live.79boatrace.com` が「接続はプライベートではありません(なりすまし)」警告。
  - 検証: DNSは正常(live→79boatrace.github.io→185.199.108-111.153)。`http://` は200で開ける。
    GitHub Pages API が `https_enforced:false` かつ **https_certificate オブジェクト無し=Let's Encrypt未発行**。
    上流が既定の `*.github.io` 証明書を返す→`live.79boatrace.com` と不一致でSafari警告。
  - 応急策(http または警告→「このWebサイトを閲覧」で先へ)で開いたところ、
    **解説音を裏で再生・映像/音声の切替とも期待通りに動作することをユーザー実機で確認済み**。
    → solo/keep-alive/mute-unMute 設計はiPhoneで正しく機能している（2本同時制約の心配は杞憂だった）。
- 恒久対応(証明書再発行): Settings→Pages でカスタムドメインを一旦消して保存→再入力して保存→
  発行後に「Enforce HTTPS」をON。通常数分〜最大24h。CAA等のブロックは未確認だが標準構成。

## PROJECT_SUMMARY.md（引き継ぎ資料）
- リポジトリ内 `PROJECT_SUMMARY.md` を作成・push済み（2026-06-26）。
- 用途: ChatGPTや別セッションにプロジェクト全体像を渡すための引き継ぎ読み物。
- 開発用の本ファイル(`project_race_pip_viewer.md`)＝作業正本、`PROJECT_SUMMARY.md`＝対外まとめ、の住み分け。

## 2026-06-26 の作業（このセッション）
### 追加: デバッグ表示（`?debug=1` または URLに `#debug`）
- 画面左上に小さく以下を常時表示（通常時は非表示）:
  - mode: iPhone(solo)/PC
  - audio希望 / armed / live
  - visual / lag
  - A・Bスロットの状態（YT: 再生中/停止/buffer・消音/音あり・現在t/duration、映像: iframe有無）
  - 解説音ONタップ: 済/未
  - 最後の操作（unMute+play / mute / seekTo / no player yet / ERR…）
- 実装: `DEBUG` 判定 + `dbgRender()` を600msポーリング、各操作で `dlog()`。
- ねらい: 「音が出ない」のか「映像が止まる」のかを実機の表示だけで切り分ける。

### まだ未確定
- 実機の具体的な失敗症状が未共有（「あかんかった」のみ）。分岐A〜Dの確定待ち。

## 実機テスト手順（debug付き）
1. iPhoneで `https://live.79boatrace.com/?debug=1` を開く（左上にログが出る）。
2. ⚙設定でレース場＋解説YouTube URLを入れ「適用して再生」。
3. `🔊解説の音を出す` をタップ。
4. 左上ログを読む:
   - B[youtube] が「再生中/音あり」になり、A[boatcast]の映像が止まらないか。
   - A の映像が「停止」になったら → 分岐B（2動画同時制約）。
   - B が「プレイヤ無 / 未開始」のまま → 分岐A（ready未達/gesture）。
5. `🔊音声切替`・`−/＋` を操作し、最後の操作ログと t=現在/duration を確認。

## 症状別の次手（方針）
- A. 解説音が出ない: gesture内で unMute/setVolume/playVideo 到達確認。プレイヤready前なら
  onReady で audioArmed を見て自動unMute。hidden要素のブロック要因確認（opacity:0は維持、display:none回避済み）。
- B. 解説音は出るが映像が止まる: iPhoneは2動画同時不可が本質。
  → 「映像=通常再生 / 音声=YouTube」の同時維持が無理なら、
    (a) 音声ソースを映像と同じ1本にして“ラグ”概念を捨てる、
    (b) もしくは解説を映像なし(音声優先)で割り切り、レース映像は静止/低fpsを許容、等の割り切りを検討。
- C. 切替が遅い/不安定: iframe再生成・URL差し替えが切替経路に無いことを確認（現状mute/unMuteのみ=OK）。
- D. ラグが効かない: 対象がYTスロットか、live/DVRで getDuration()>0 か、seekToログ確認。

## デプロイ手順（メモ）
- `C:\Users\USER\79boatrace-live` で編集 → コミット → `git push origin main`。
- push はユーザー承認後に行う。
