# app

Echo Show 5 (1st gen / `checkers` / MT8163 / armeabi-v7a / Android 11 / RAM 1GB)
上で動作する Android アプリを開発するディレクトリ。

## 対象環境 (実機実測)

- 画面: 5.5 インチ / 960x480
- ABI: **armeabi-v7a のみ** (64bit なし)
- OS: LineageOS 18.1 (Android 11 / API 30) + Kiosk Satellite Edition
- メモリ: 1GB (Kiosk Satellite 表示中も約 400MB 空き)
- 入力: タッチ / マイク / スピーカー / 上部ボタン (短押し=画面、長押し=プライバシー)

## 実機構成 (2026-11-19 実測・読み取り専用)

| 項目 | 値 |
|---|---|
| デバイス | `G0911B06037402HF` (Echo Show 5 / `checkers` / H23K37) |
| 画面 | **960x480 / density 195** (約 788x394dp) |
| ABI | **armeabi-v7a のみ** (32bit・64bit なし) |
| OS | Android 11 (SDK 30) / LineageOS 18.1 (ronnz ビルド 2026-09-10) |
| CPU | ARMv8 Processor rev 3 (Cortex-A53 クアッド・32bit userland) |
| RAM | **1GB** (MemFree 36MB / MemAvailable 約 344MB / Cached 276MB) |
| ストレージ | `/data` 3.7GB 中 2.8GB 使用 (**残 783MB**・79%) |
| パッケージ数 | **205** |
| デフォルト HOME | **`com.ss.launcher2` v3.0.4** (targetSdk 34・2026-11-04 導入) |
| サードパーティ | `org.videolan.vlc` / `app.morphe.android.youtube` / `anddea.youtube` /
  `app.revanced.android.gms` (microG) / `com.ss.launcher2` |
| Kiosk Satellite | **未インストール** |
| WebView | `com.android.webview` (標準) |
| adb | `ro.debuggable=1` / `ro.adb.secure=1` |
| 現在の画面 | 常時時計 (7:58 / 11月19日(木)) + 「充電が完了しました」通知 = `ss.launcher2` の常時表示 |

### 観察

- 現状でも**常時時計 + 充電表示**のスマートディスプレイとして動作している
  (`ss.launcher2` がランチャー兼スクリーンセーバー役)
- 205 パッケージ + GApps/microG が同居し、1GB RAM に対してかなり重い構成
- Kiosk Satellite は入っていない → app 側で情報表示/ダッシュボードを担う余地がある
- VLC / YouTube クライアントが既に入っており、メディア再生は既存アプリで賄える

## 最適化作業ログ (2026-11-19)

### バックアップ (削除前に取得済み)

- `adb root` 成功 → **uid=0(root)** (`/system/build.prop` も読める)
- パッケージ→APK パスマッピング: `packages-with-paths.txt` (205 件)
- 個別 APK pull: **193/205 成功・1.32GB** (12 件は root 化前の一時切断で失敗)
- **root 化後 `/system` 全体 pull: 3,655 ファイル / 2.02GB 完了**
  → 全システム APK を復元可能な状態で保持
- 保存先: `C:\Users\x\AppData\Local\Temp\opencode\echo5-apk-backup\`

### phone/SMS 関連の最適化 (ユーザー指示: 削除やダミー化を検討)

**重要な発見**: 現行 ronnz ビルドでは.phone/SMS 関連 10 件は
**既に `enabled=0` (pm disable-user で無効化＝ダミー化済み)** だった。

ユーザー判断で「一部だけ削除」を実施:

| 扱い | パッケージ |
|---|---|
| **削除 (4 件・完了)** | `com.android.phone`(TeleService) / `com.android.server.telecom` / `com.android.mms.service` / `com.android.smspush` |
| **無効化のまま残す (6 件)** | `com.android.providers.telephony` / `com.android.providers.blockednumber` / `com.android.contacts` / `com.android.providers.contacts` / `com.android.simappdialog` / `com.android.companiondevicemanager` |

削除は `pm uninstall -k --user 0 <pkg>` (root 権限下)。4 件とも `Success`、
`pm list packages` で該当なしを確認。依存の強い provider / contacts 系は
無効化のまま残し、Settings/SystemUI の不安定化を回避した。
各 APK はバックアップ済みなので再インストールで復元可能。

**再起動検証 (完了)**:
- `adb reboot` → 約 20 秒で `sys.boot_completed=1`
- Settings / SystemUI / PackageInstaller / WebView / `com.ss.launcher2` すべて生存
- 削除 4 件は再起動後も消えたまま = **削除は永続的**
- HOME ランチャー `com.ss.launcher2/.MainActivity` 正常
- 画面: 常時時計「8:09 / 11月19日(木)」+ 充電完了表示を確認 (スクリーンショット)
- パッケージ数 **205 → 201** に減少
- 再起動で adb root は解除 (uid=2000) → 以降の削除作業には再 `adb root` が必要

### Kiosk Satellite 導入の準備

- 最新版 **`2026.11.23`** を確認
- 32bit 用 APK **`kiosk-satellite-2026.11.23.armeabi-v7a.apk`** (97.8MB) を
  GitHub releases からダウンロード済み
  - SHA-256 `cf01028aae4db168352d1f7db18dbd4bbebfc18a8c38ba607f5636bfb2e83fb7`
    = GitHub 公表値と一致 (改ざんなし)
- 保存先: `...echo5-apk-backup\kiosk-satellite-2026.11.23.armeabi-v7a.apk`
- 要件: Android 7.0+ / Home Assistant インスタンス + 長期アクセストークン
- 公式推奨は「Kiosk Satellite Edition of LineageOS」イメージだが、
  アプリ単体でも通常の LineageOS 18.1 にインストール可能
  (ただし raw マイク / エコーキャンセラは Edition イメージ固有の機能)

### 残りアプリ削除 (aggressively・実施済み)

- 再 `adb root` (uid=0) のうえ、remove-list の残り **126 件を一括削除、
  OK=126 / FAIL=0**。phone/SMS 4 件と合わせ計 130 件削除。
- **パッケージ数 205 → 75** (基盤 + 残存 VLC/YouTube クライアント + ss.launcher2 等)
- 残存アプリ例: `com.android.settings` / `systemui` / `webview` / `packageinstaller` /
  `com.ss.launcher2` / `org.videolan.vlc` / `anddea.youtube` / `app.morphe.android.youtube`
- GApps/microG (`com.google.android.gms` / `gsf` / `app.revanced.android.gms` /
  `org.microg.gms.droidguard` / `com.android.vending`) も削除済み
  → YouTube クライアントは未検証だが動作不能の可能性大

### Google アカウント残骸問題 (未解決)

- アカウント `nsmr0604.ryo@gmail.com` (type=`app.revanced`) が
  AccountManager に残存。GApps 本体は消えたが DB 残骸から復活する
- `/data/system_ce/0/accounts_ce.db` をバックアップ取得後に root で削除 →
  再起動しても **アカウントが復活** (他ストア or authenticator 残骸から再登録)
- **Kiosk Satellite を device owner にするには「アカウント無いこと」が必須**。
  未解决。Kiosk Satellite 側のドキュメント (`permissions.md`) に従い対処が必要

### Kiosk Satellite 導入 (インストール完了・設定は未完了)

- `adb install -r kiosk-satellite-2026.11.23.armeabi-v7a.apk` → **Success**
- **実際のパッケージ名は `me.jxl.kiosk_satellite`** (予想の org.kiosksatellite ではない)
- versionName `2026.11.23` / versionCode 366 / minSdk 24 / targetSdk 36
- `am start -n me.jxl.kiosk_satellite/.MainActivity` で起動成功
  (ResumedActivity で確認)。ただし画面はまだ `ss.launcher2` の常時時計のまま
- **セットアップ未完了**: Home Assistant の長期アクセストークン + ダッシュボード選択
  が必要。セットアップは画面のウィザード or `http://<show ip>:2324` の remote admin で実施

## 破壊的環境整備の実施 (2026-11-19)

### 実施内容 (ユーザー判断: Fossify 交代 + 安全な削除 → 途中で Total Launcher 復帰)

1. **安全な追加削除 15 件** (すべて Success、バックアップ済み):
   `com.android.launcher3` / `org.lineageos.updater` / `org.protonaosp.deviceconfig` /
   `com.android.traceur` / `com.android.wallpaper` / `wallpaperbackup` / `wallpapercropper` /
   `storagemanager` / `backupconfirm` / `localtransport` / `onetimeinitializer` /
   `modulemetadata` / `statementservice` / `settings.intelligence` / `managedprovisioning`
2. **Fossify Launcher 1.10.0 を導入** (APK SHA-256 検証済み・minSdk 26・5.2MB、
   pkg `org.fossify.home`)。**HOME ロール取得に成功**
   (`cmd role add-role-holder` が通った — Kiosk Satellite では category.HOME が無く失敗したが
   Fossify は category.HOME を持つため成功)
3. **Total Launcher を削除** → しかし **Fossify の MainActivity が縦向き固定**
   (480x960) で Echo5 の横パネル (960x480) に合わなかった。
   `user_rotation` 変更 (0/1/2/3) は WindowManager の mRotation は追従するが
   Fossify Activity は縦のまま描画。apktool 改造も選択肢だったが…
4. **ユーザー判断で Total Launcher に復帰**:
   - Fossify 削除 → Total Launcher APK 再インストール (バックアップから)
   - HOME ロール `com.ss.launcher2/.MainActivity` に復帰
   - `user_rotation=0` (mRotation=0・横) に戻す
   - キオスク構成 (時計中央 `kioskclock01` + ドロワー `kioskdrawer1`) を
     JSON 直接編集で再構築、`locked=true` / `lockLevel=2` を確認

### 最終状態

- **58 パッケージ** (73 → 15 削除)、MemAvailable 約 **366MB**、ストレージ残 1.7G
- HOME = Total Launcher、横画面 (mRotation=0)、キオスクロック有効
- 画面: 中央の八角形アナログ時計のみ (Fossify の縦問題は解消)
- 学び: **Fossify Launcher は HOME ロールは取れるが縦固定のため Echo Show 5 には不向き**。
  Total Launcher (横対応・キオスクロック・JSON 設定) が引き続き最適

## 環境整備候補の調査 (2026-11-19)

破壊的変更を含む次の整備方針を検討中。現状: 73 パッケージ / MemAvailable 約 357MB /
ストレージ残 1.7G / HOME = Total Launcher (`com.ss.launcher2` v3.0.4) / root 取得可。

### 残存パッケージのうち追加削除候補

| パッケージ | 役割 | 削除の影響 |
|---|---|---|
| `com.android.launcher3` | AOSP 標準ランチャー | 未使用 (Total Launcher が HOME)。削除可 |
| `com.android.bluetooth` | Bluetooth | 起動時に「繰り返し停止」Dialog が出た原因の可能性。削除で Dialog 消えるが BT 機能も消える |
| `com.android.dreams.basic` | スリープ時計/ドリーム | 消灯時の時計表示が消える (キオスクでは不要な可能性) |
| `com.android.wallpaper*` (3 件) | 壁紙 | 単色壁紙なら削除可だが Total Launcher が壁紙を使うため注意 |
| `org.lineageos.updater` | ROM 更新 | 更新不可になる (Actions ビルド運用なら不要) |
| `org.protonaosp.deviceconfig` | 端末設定拡張 | 削除可 |
| `com.android.traceur` / `com.android.storagemanager` 等 | デバッグ/管理 | 削除可 |
| `me.jxl.kiosk_satellite` | Kiosk Satellite | set up 未完了。HOME 化しないなら削除候補 |

### 代替ランチャー候補 (GitHub 実測)

| ランチャー | 特徴 | APK |
|---|---|---|
| **Fossify Launcher 1.10.0** | FOSS・軽量・フルウィジェット対応・実績あり (12.6k DL) | `launcher-16-foss-release.apk` 5.2MB |
| OlauncherCF / MOOLauncher | 超ミニマル・テキストのみ・ウィジェットなし | 小さい |
| Escape-Launcher | ミニマル・集中向け (481★) | 中 |
| rama-io/mako | プライバシー重視ミニマル (292★) | 中 |

- **Fossify Launcher** が最も現実的 (軽量 + ウィジェット + HOME ロール取得可 =
  Total Launcher の問題を回避できる)。ただしキオスクロック機能は Total Launcher の方が上
- Kiosk Satellite を HOME にするには **category.HOME フィルタが無いため不可**
  (Edition イメージ flash が必要 → ROM 側作業・このセッション対象外)

## ホームランチャー再構築・キオスク端末化 (2026-11-19・完了)

### 実施内容

1. **全スクリーン設定を削除**
   - 既存ページ `179a8b6e2a0` (時計) / `15d5e0ee48f` (ドロワー) を削除
   - `pageList`・`pages/*`・`pinBoard*` を削除
   - 削除前に `total-launcher-backup\` へ全設定をバックアップ済み
2. **新規 2 ページ構築** (JSON 直接編集・base64 経由で書き込み)
   - `kioskclock01` — 時計ウィジェット (`T:15` / `bg_clock_widget`) を**中央配置**
     (`X:299.5, Y:145.6, W:188.7, H:188.7`)。`.l` ロック版も同一内容
   - `kioskdrawer1` — アプリドロワー (`T:10`)
   - `pageList = [{"id":"kioskclock01","l":"","fh":true},{"id":"kioskdrawer1","l":"","fh":true}]`
3. **編集ロック** — 左下「編集オン」ボタンをタップして
   `locked=true` / `lockLevel=2` に戻す (キオスク状態に固定)

### 重要なノウハウ (実測・ハマりどころ)

- **`pageList` の `l` フィールドは空文字 (`""`) が正**。
  `"l":"1"` にするとランチャー起動時に `pageList` が削除される (実測)
- ページファイル (`.l` 含む) は **`chown u0_a173:u0_a173` + `chmod 600`** が必要。
  root で書くと owner が root:root になり読み込まれない
- PowerShell から adb 経由で JSON を書く場合、パス区切りに `/` を使う
  (`\` だと `filespages` のように消える)。内容は base64 経由が確実
- 一時編集解除は prefs XML の `locked` を `false` にして
  `am force-stop` → HOME 起動で反映 (base64 書き込みが確実)

### 最終画面 (確認済み)

- **ページ 1**: オレンジ壁紙 + 中央の八角形アナログ時計のみ。
  編集ボタン/ページ操作ボタンは消え、ステータスバーとナビバーのみ
- **ページ 2**: アプリドロワー — Kiosk Satellite / Files / 設定 / トータルランチャー
  の 4 アイコンのみ (不要アプリ削除済み)
- パッケージ数 73 / `locked=true` / `lockLevel=2` を確認

## VLC / YouTube 削除 (2026-11-19・ユーザー指示)

- 削除 (3 件・`pm uninstall -k --user 0`、root 下):
  `org.videolan.vlc` (47MB) / `anddea.youtube` (120MB) / `app.morphe.android.youtube` (146MB)
- すべて `Success`、`pm list packages` で消滅確認
- 各 APK はバックアップ済み (`echo5-apk-backup\apks\` に保存)
- **パッケージ数 76 → 73**、MemAvailable 約 383MB に改善
- ホーム画面から VLC/YouTube アイコンが消え、時計ウィジェット + 不明アイコン 2 つ
  ("?" = ラベル未設定の残存ショートカット) のみのシンプル構成に
- これにより **VLC/YouTube のウィジェットプロバイダも消滅** →
  Total Launcher に追加できる AppWidget は事実上なし
  (ランチャー内蔵の `T:15` 時計ウィジェットのみ)

## 最終状態 (2026-11-19・ユーザー判断で「現状で完了」)

- **`ss.launcher2` を再有効化** (`pm enable --user 0`) → HOME ランチャー復帰、
  常時時計表示に戻ったことをスクリーンショットで確認
- **パッケージ数 76** (開始時 205 → 130 件削除 + Kiosk Satellite 1 件追加)
- **MemAvailable 約 373MB** (開始時 344MB → 改善、 Hydrogen 再起動直後変動あり)
- Kiosk Satellite はインストール + 全権限付与済みだが、**HOME ロール未取得・set up 未完了**。
  ユーザー判断でここで完了とする (アンインストールはしていない)

### HOME ロール取得が出来なかった理由 (実測・重要)

- この Kiosk Satellite APK は **category.HOME の intent-filter が manifest に無い**
  (`aapt2 dump badging` で launchable-activity は MainActivity のみ)
- `adb shell cmd role add-role-holder android.app.role.HOME me.jxl.kiosk_satellite`
  は **「not qualified ... missing RequiredComponent{...category.HOME}」で失敗**
- アプリ自身が `RoleManager.requestRole` で取得する設計 (-ui の
  「Act as the home screen」スイッチ ON が必要、Android 10+ は確認ダイアログ要)
- → **単体 APK インストールでは HOME ロールを取れない**。
  公式推奨の「Kiosk Satellite Edition of LineageOS」イメージを flash するか、
  画面でスイッチを ON にする必要がある

## ステータス

- [ ] アプリの種類・技術スタックを決定
- [ ] プロジェクト作成
- [ ] ビルド (Actions で prerelease へ APK 添付)
- [ ] 実機インストール・動作確認
