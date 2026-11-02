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

## 守ること：upstream と同じファイルは変えない

jazmf-dx は「upstream ＋ 社内版だけのファイル」にしておく。upstream にあるファイル（`src/`・`src-tauri/src/`・`README.md`・`CLAUDE.md`・`build.yml` など）を jazmf-dx で直接変えると、upstream がそのファイルを変えるたびに取り込みで衝突し、差がだんだん広がっていく。

| やりたいこと | どこで、どうやるか |
|---|---|
| 機能の改修・バグ修正（社内で見つけたものも含む） | upstream で直して、取り込む |
| 社内版で使う種類や外す機能を変える（例：動画を足す） | `corp.yml` の環境変数を変える（`STOCKROOM_KINDS=image,video`） |
| 社内版で新しく外したい機能がある（例：フォルダの共有） | まず upstream に、外せる仕組みを汎用の形で足す（[LIMITS.md](LIMITS.md)）。社内版では `corp.yml` で使うだけにする |
| 製品名・ID・インストーラの設定 | `tauri.corp.conf.json` |
| 社内の運用の手順 | この文書 |

社内版にしか要らない変更も、コードに「社内版なら」と書くのではなく、「この機能を外せる」という形で upstream に入れる。

`main` への PR では `fork-guard` ワークフローが、upstream との差が下の「社内版だけのファイル」に収まっているかを確かめ、ほかのファイルが変わっていたら失敗する。

GitHub の画面に出る「This branch is N commits ahead of hamirilo/stockroom:main」は、社内版のコミットと取り込みのマージの数なので、増えていって問題ない。「Contribute」から upstream への PR は作らない（社内版のファイルが upstream に入ってしまう）。

## 社内版だけのファイル

社内版のために置くのは、次の新しいファイルだけにする。

- `src-tauri/tauri.corp.conf.json`：製品名・ID・インストーラの種類を決める。updater の設定、拡張機能の同梱、旧名（Image Library）を消すインストーラの処理を外す
- `.github/workflows/corp.yml`：外す機能の環境変数（`STOCKROOM_*`）を設定し、テストとビルドをする
- `.github/workflows/fork-guard.yml`：PR で、upstream との差がこれらのファイルだけかを確かめる
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

取り込んだあとで `git diff upstream/main main --stat` を見ると、上の「社内版だけのファイル」だけが出る。ほかのファイルが出たら、jazmf-dx で直接変えたものがあるので、upstream に入れ直す。

### 個人版のワークフロー

upstream から来る `.github/workflows/build.yml`（個人版のリリース）は、jazmf-dx では使わない。個人版の更新ファイルに署名する鍵（`TAURI_SIGNING_PRIVATE_KEY`）を会社のリポジトリに置かないので、`main` に push するたびに最後の署名で失敗する。

ファイルを変えると取り込みのたびに衝突するので、GitHub の画面で止める：Actions → build →「…」→ **Disable workflow**。

## リリース

1. upstream を取り込む（上の手順）
2. 個人版と同じ番号でタグを付ける：`git tag corp-vX.Y.Z && git push origin corp-vX.Y.Z`
3. GitHub Actions の `corp` ワークフローがテストし、Windows のインストーラを作る
   - GitHub のリリースは作らない。成果物は実行の Artifacts（`stockroom-corp-corp-vX.Y.Z`）にだけ出る
   - インストーラと `SHA256SUMS.txt` が入る。ハッシュは実行のサマリーにも出る
4. インストーラを検査に提出する。そのときに SHA-256 を控える
5. 検査を通ったら、社内の共有の場所に置いて配る。置く前に SHA-256 が控えと同じか確かめる。検査に出したファイルと配るファイルは、まったく同じでなければならない

社内版は自分では更新しない。検査に手間がかかるので、まとめて数か月に 1 回などのペースにする。

### 版の番号

社内版の番号は、取り込んだ upstream の番号と同じにする。アプリに出る番号は `tauri.conf.json` の `version` で決まり、このファイルは jazmf-dx では変えないので、自然に揃う。

社内版は検査を通す版だけを出すので、間の版は飛ばしてよい（例：個人版 0.38.1 → 0.39.0 → 0.40.0 のうち、社内版は 0.38.1 と 0.40.0 だけ）。番号が同じなら、社内版のある版は個人版の同じ番号の版と同じコードになる。

`corp.yml` や `tauri.corp.conf.json` だけを変えて（例：動画を足す）出し直すときは、コードの番号が変わらないので、中身の違うインストーラが同じ番号でできてしまう。そのときは次のどちらかにする。

- 基本：upstream の番号が上がるのを待つ。急ぐなら upstream でパッチ版（0.38.2 など）を出してから、社内版を作る。アプリに出る番号で区別できる
- 急ぎで番号を上げられないとき：タグに枝番を付ける（`corp-v0.38.1-2`）。アプリの表示は 0.38.1 のままなので、どちらの版かは控えた SHA-256 で見分ける

インストーラはコード署名をしていないので、初回は SmartScreen の警告が出る。会社でコード署名の証明書を用意できれば、検査と警告の両方が楽になる可能性がある。
