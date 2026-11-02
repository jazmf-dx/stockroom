# 社内版

jazmf-dx/stockroom でだけ扱う、会社の PC（主に Windows）向けのビルド。中身は hamirilo/stockroom（upstream）と同じコードで、[機能を絞ったビルド](LIMITS.md)の仕組みを使って次を外している。

| | 個人版（upstream） | 社内版 |
|---|---|---|
| 扱える種類 | すべて | 画像だけ（動画は後で足す予定） |
| ブラウザ拡張・Claude（MCP） | 使える | ない。ポートを開かず、拡張機能も同梱しない |
| アプリ内アップデート | hamirilo/stockroom のリリースから | jazmf-dx/stockroom のリリースから。社内版専用の鍵で署名し、検査を通して Publish した版だけが届く |
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

- `src-tauri/tauri.corp.conf.json`：製品名・ID・インストーラの種類を決める。updater の配信先と公開鍵を社内版のものにする。拡張機能の同梱、旧名（Image Library）を消すインストーラの処理を外す
- `.github/workflows/corp.yml`：外す機能の環境変数（`STOCKROOM_*`）を設定し、テストとビルドをして、下書きのリリースを作る
- `.github/workflows/fork-guard.yml`：PR で、upstream との差がこれらのファイルだけかを確かめる
- `docs/CORP.md`：この文書

社内で見つけた不具合や、社内版にも個人版にも役立つ改善は、upstream に入れてから取り込む。外せる機能を増やすとき（たとえば種類に動画を足す）は、upstream の仕組み（`STOCKROOM_KINDS` など）で足りるなら、社内版では `corp.yml` の値を変えるだけで済む。

## 開発

```bash
STOCKROOM_KINDS=image STOCKROOM_NO_CONNECT=1 STOCKROOM_NAME='Stockroom（社内版）' STOCKROOM_TAG_PREFIX=corp- \
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

## 署名の鍵

アプリ内アップデートは、社内版専用の鍵で署名した更新しか受け付けない。公開鍵は `tauri.corp.conf.json` の `plugins.updater.pubkey`。個人版の鍵とは別のもので、個人版の更新が社内版に入ることはない（逆も同じ）。

- 秘密鍵とパスワードは、Environment `corp-release` の Secret（`CORP_SIGNING_PRIVATE_KEY`・`CORP_SIGNING_PRIVATE_KEY_PASSWORD`）にだけ置く。Repository secret にはしない
- `corp-release` は、タグ `corp-v*` からしか使えないようにしてある（Settings → Environments → Deployment branches and tags）。ほかのブランチのワークフローからは鍵を読めない
- 秘密鍵のファイルは手元に残さないか、パスワード管理ツールなど安全な場所に置く。漏れたら、その鍵で署名した更新が全員の PC に入りうる
- 鍵をなくしたり作り直したりすると、それまでの社内版には更新が届かなくなる。新しい公開鍵の版を、全員に手で入れ直してもらう

## リリース

1. upstream を取り込む（上の手順）
2. 個人版と同じ番号でタグを付ける：`git tag corp-vX.Y.Z && git push origin corp-vX.Y.Z`
3. GitHub Actions の `corp` ワークフローがテストし、Windows のインストーラを作って、**下書きのリリース** `corp-vX.Y.Z` に添付する
   - 添付されるのは `-setup.exe`・その署名（`.sig`）・`latest.json`。アプリ内アップデートが入れるのは、インストーラと同じ `-setup.exe`
   - 実行の Artifacts にも、インストーラと `SHA256SUMS.txt` が入る。ハッシュは実行のサマリーにも出る
   - 下書きのうちは、アプリ内アップデートからは見えない
4. 下書きのリリースから `-setup.exe` を取り、検査に提出する。そのときに SHA-256 を控える
5. 検査を通ったら、Publish する前に、下書きに添付されている `-setup.exe` の SHA-256 が控えと同じか確かめる。同じタグでワークフローを動かし直すと添付が差し替わるので、検査のあとは動かし直さない

   ```bash
   gh release download corp-vX.Y.Z -R jazmf-dx/stockroom -p '*-setup.exe' -D check
   sha256sum check/*-setup.exe   # Windows: Get-FileHash check\*-setup.exe
   ```

6. 下書きの本文に変更点を書き、**Publish** する。この時点で、社内版を使っている全員のアプリに更新が届く（起動したときに確かめる）
7. まだ社内版を入れていない人には、同じ `-setup.exe` を社内の共有の場所に置いて配る

Publish したあとで問題が見つかったら、リリースを下書きに戻す（Edit → Save as draft）。それ以降の配信は止まるが、すでに更新した PC は戻らない。直した版を、次の番号で出す。

アプリ内アップデートが入る前の社内版（v0.38.1 の `corp-v0.38.1`）は更新を確かめないので、アプリ内アップデートが入った最初の版だけは、全員に手で入れ直してもらう。

### 版の番号

社内版の番号は、取り込んだ upstream の番号と同じにする。アプリに出る番号は `tauri.conf.json` の `version` で決まり、このファイルは jazmf-dx では変えないので、自然に揃う。

社内版は検査を通す版だけを出すので、間の版は飛ばしてよい（例：個人版 0.38.1 → 0.39.0 → 0.40.0 のうち、社内版は 0.38.1 と 0.40.0 だけ）。番号が同じなら、社内版のある版は個人版の同じ番号の版と同じコードになる。

`corp.yml` や `tauri.corp.conf.json` だけを変えて（例：動画を足す）出し直すときは、コードの番号が変わらないので、中身の違うインストーラが同じ番号でできてしまう。そのときは次のどちらかにする。

- 基本：upstream の番号が上がるのを待つ。急ぐなら upstream でパッチ版（0.38.2 など）を出してから、社内版を作る。アプリに出る番号で区別できる
- 急ぎで番号を上げられないとき：タグに枝番を付ける（`corp-v0.38.1-2`）。アプリの表示は 0.38.1 のままなので、どちらの版かは控えた SHA-256 で見分ける。アプリ内アップデートは番号が上がらないと届かないので、この版は手で入れ替えてもらうことになる。Publish もしない（「更新内容を見る」は `corp-v0.38.1` のページを開く）

インストーラはコード署名をしていないので、初回は SmartScreen の警告が出る。会社でコード署名の証明書を用意できれば、検査と警告の両方が楽になる可能性がある。
