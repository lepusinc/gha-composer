# gha-composer 設計メモ

このリポジトリの composite action（`vendor-build` / `vendor-load`）の設計判断と内部ロジックをまとめる。
利用者向けの使い方は [README.md](./README.md) を参照。

## 設計判断

### 1 リポジトリに 2 action を同梱

vendor キャッシュキーの計算は `vendor-build` が担い、`vendor-load` はそのキーを受け取って復元するだけ。
両者はキー仕様で密結合なので、常に同期更新できるよう同一リポジトリに置く。
参照は `lepusinc/gha-composer/vendor-build@v1` のサブディレクトリ形式。

### setup-php は caller 責務（Option A）

action 内で `shivammathur/setup-php` を実行しない。理由は以下:

- caller が先に PHP を立てるケース（例: lock 生成のため先に PHP をセットアップする）で二重 setup を避ける
- PHP 構成（拡張・ini・coverage）を caller が自由に制御できる

そのため `vendor-load` に coverage 入力は持たせない（coverage は caller の setup-php で指定する）。

### vendor-load は actions/cache/restore の薄いラッパー

`vendor-load` は inputs / outputs を `actions/cache/restore@v5` と同一にし、復元キーは caller から受け取る。
`vendor-build` が出力した `vendor-key` をジョブ出力経由で渡す想定。
これにより `vendor-load` は `composer.lock` / PHP に依存せず、`composer.lock` を持たない fresh checkout でも動く。

## キャッシュキー設計

計算式は `vendor-build` の一箇所のみに存在する。

```bash
composer_hash=$(sha256sum composer.lock | awk '{print $1}')
vendor_hash=$( (sha256sum composer.lock; sha256sum composer.json) | sha256sum | awk '{print $1}')
# composer-key = ${RUNNER_OS}-composer-${composer_hash}
# vendor-key   = ${RUNNER_OS}-vendor-${vendor_hash}
```

`vendor-build` の Composer ダウンロードキャッシュには restore-keys `${{ runner.os }}-composer-` を付け、
lock 変更時もダウンロード済みアーカイブを部分的に再利用できるようにする。

### OS を先頭に置く

`${{ runner.os }}-<用途>-<hash>` の順にする。`restore-keys` はプレフィックス一致なので、
OS を先頭にするとフォールバックが必ず同一 OS 内に収まる（別 OS のキャッシュにマッチしない）。GitHub 公式サンプルの慣習でもある。

### PHP バージョンをキーに含めない

`vendor/` の内容は `composer.lock`（固定された依存バージョン）だけで決まり、`composer install` を実行する
PHP バージョンには依存しない。PHP バージョンで解決結果が変われば `composer.lock` の内容（＝ハッシュ）自体が
変わるため、PHP 差は自動的にキーへ反映される。

### composer ダウンロードキャッシュは composer.lock のみ

ダウンロードされる dist アーカイブは固定バージョン（= `composer.lock`）だけで決まり、`composer.json` の
autoload 等の変更では中身が変わらない。json を足すと無駄に miss が増えるため lock 単体にする。

### vendor キャッシュは composer.lock + composer.json

`composer.lock` の `content-hash` は `composer.json` の一部（`require` / `require-dev` / `repositories` /
`extra` 等）しか反映せず、`autoload` / `autoload-dev` は含まれない。autoload だけを変更すると `composer.lock`
は不変なのに、生成される autoloader（`vendor/composer/autoload_*.php`）は変わる。
本 action は cache-hit 時に `composer install` をスキップするため、`composer.json` も含めて vendor キャッシュを
無効化しないと、stale な autoloader を復元してしまう。

## 検証

`.github/workflows/test.yml` が PHP 8.2 / 8.3 / 8.4 / 8.5 のマトリクスで build → load を自己検証する。
fixture は `tests/fixture`（`psr/log` を要求、`composer.lock` をコミット済み）。

`load` ジョブは fixture の `composer.lock` / `composer.json` からキーをインラインで再計算して `vendor-load` に
渡す（マトリクスでは job outputs を per-leg で渡せないため）。キーは PHP 非依存なので、各レグは同一の
`${RUNNER_OS}-vendor-<hash>` を共有する（最初の build レグが保存、残りは "cache already exists" 警告のみ）。

php-ci の `run-php-test.yml`（reusable workflow）には統合しない:

- reusable workflow 内の `uses: ./vendor-build` は呼び出し側リポジトリを指すため、外部 caller で壊れる
- test ジョブが fresh checkout で `composer.lock` を持たず、当初案ではキー再計算ができなかった

検証は `./` 参照が正しく解決する gha-composer 自身の通常ワークフローで行う。

## 命名規約・拡張

- `gha-` プレフィックス（既存 `gha-report-code-coverage` に準拠）
- composer 関連の action は `gha-composer/<action-name>/action.yml` として同リポジトリに追加する（例: `composer-audit`）

## リリース手順

1. main マージ後、`test.yml` が 4 PHP バージョンで緑になることを確認
2. `git tag v1.0.0 && git push origin v1.0.0`
3. `.github/workflows/update-major-tag.yml`（`lepusinc/.github` の再利用ワークフローを呼ぶ）が走り、
   `v1` 移動タグが作られる → consumer は `@v1` で参照できる
