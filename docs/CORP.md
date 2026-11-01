# 社内版

jazmf-dx/stockroom でだけ扱う、会社の PC（主に Windows）向けのビルド。中身は hamirilo/stockroom（upstream）と同じコードで、[機能を絞ったビルド](LIMITS.md)の仕組みを使って次を外している。

| | 個人版（upstream） | 社内版 |
|---|---|---|
| 扱える種類 | すべて | 画像だけ（動画は後で足す予定） |
| ブラウザ拡張・Claude（MCP） | 使える | ない。ポートを開かず、拡張機能も同梱しない |
| アプリ内アップデート | GitHub のリリースから | ない。検査を通した exe を手で入れ替える |
| 製品名・ID | `Stockroom` / `com.local.stockroom` | `Stockroom Corp` / `com.local.stockroom.corp` |
| インストーラ | macOS の .dmg、Windows の -setup.exe | Windows の -setup.exe だけ |

ID が違うので、設定・ライブラリの一覧・ログなどアプリのフォルダは個人版とは別になる。

フォルダの共有（LAN への待ち受け）は社内版にも残している。止めるかどうかは未決定。

## 社内版だけのファイル

社内版のために置くのは、次の新しいファイルだけにする。upstream にあるファイルには手を入れない。そうしておけば、upstream を取り込んでも衝突しない。

- `src-tauri/tauri.corp.conf.json`：製品名・ID・インストーラの種類を決める。updater の設定、拡張機能の同梱、旧名（Image Library）を消すインストーラの処理を外す
- `.github/workflows/corp.yml`：外す機能の環境変数（`STOCKROOM_*`）を設定し、テストとビルドをする
- `docs/CORP.md`：この文書

社内で見つけた不具合や、社内版にも個人版にも役立つ改善は、upstream に入れてから取り込む。外せる機能を増やすとき（たとえば種類に動画を足す）は、upstream の仕組み（`STOCKROOM_KINDS` など）で足りるなら、社内版では `corp.yml` の値を変えるだけで済む。

## 開発

```bash
STOCKROOM_KINDS=image STOCKROOM_NO_CONNECT=1 STOCKROOM_NO_UPDATE=1 STOCKROOM_NAME='Stockroom（社内版）' \
  bun run tauri dev --config src-tauri/tauri.corp.conf.json
```

## upstream の取り込み

```bash
git remote add upstream https://github.com/hamirilo/stockroom   # 初回だけ
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

社内版だけのファイルは upstream にないので、通常は衝突しない。

## リリース

1. upstream を取り込む（上の手順）
2. 個人版と同じ番号でタグを付ける：`git tag corp-vX.Y.Z && git push origin corp-vX.Y.Z`
3. GitHub Actions の `corp` ワークフローがテストし、Windows のインストーラを作る
   - GitHub のリリースは作らない。成果物は実行の Artifacts（`stockroom-corp-corp-vX.Y.Z`）にだけ出る
   - インストーラと `SHA256SUMS.txt` が入る。ハッシュは実行のサマリーにも出る
4. インストーラを検査に提出する。そのときに SHA-256 を控える
5. 検査を通ったら、社内の共有の場所に置いて配る。置く前に SHA-256 が控えと同じか確かめる。検査に出したファイルと配るファイルは、まったく同じでなければならない

社内版は自分では更新しない。検査に手間がかかるので、まとめて数か月に 1 回などのペースにする。

インストーラはコード署名をしていないので、初回は SmartScreen の警告が出る。会社でコード署名の証明書を用意できれば、検査と警告の両方が楽になる可能性がある。
