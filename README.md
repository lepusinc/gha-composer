# Composer GitHub Actions

PHP プロジェクトの「Composer キャッシュ + 依存インストール」を共通化する composite action 群です。
キャッシュキーは `vendor-build` が計算して出力し、`vendor-load` がそれを受け取って復元します。
両 action を常に同期更新できるよう同一リポジトリに同梱しています。

## アクション一覧

| アクション                            | 用途                                                                                     |
|----------------------------------|----------------------------------------------------------------------------------------|
| [`vendor-build`](./vendor-build) | ビルドジョブ用。Composer ダウンロードと `vendor` をキャッシュし、`vendor` キャッシュミス時に `composer install` を実行する。 |
| [`vendor-load`](./vendor-load)   | テスト・静的解析ジョブ用。`vendor-build` が保存した `vendor` キャッシュを復元する（ミス時は失敗）。                         |

## 前提

- **PHP / Composer は呼び出し側でセットアップ済みであること。** これらの action は `shivammathur/setup-php`
  を実行しません。caller が拡張・ini・coverage を自由に制御できるよう、PHP のセットアップは caller の責務です（ただし
  `vendor-load` 自体は PHP/Composer を必要としません）。
- `vendor-build` は `composer.lock` / `composer.json` からキャッシュキーを計算するため、build ジョブのチェックアウトにこれらが必要です。
  `vendor-load` はキーを **caller から input で受け取る**ため、`composer.lock` の有無に依存しません。
- キャッシュキーに PHP バージョンは含めません。`vendor/` の内容は `composer.lock`（固定された依存バージョン）だけで決まり、
  `composer install` を実行する PHP バージョンには依存しないためです。PHP バージョンによって解決結果が変われば
  `composer.lock` の内容（＝ハッシュ）自体が変わります。

## inputs

### `vendor-build`

| input               | required | default | 説明                                          |
|---------------------|----------|---------|---------------------------------------------|
| `working-directory` | no       | `.`     | `composer.json`（と `composer.lock`）があるディレクトリ |

### `vendor-load`

`actions/cache/restore@v5` の薄いラッパーで、inputs / outputs は `actions/cache/restore` と同一です。
`key` には `vendor-build` の `vendor-key` 出力を渡します。

| input                  | required | default | 説明                                             |
|------------------------|----------|---------|------------------------------------------------|
| `path`                 | yes      | —       | 復元するファイル・ディレクトリ・ワイルドカードのリスト（例: `src/vendor`）   |
| `key`                  | yes      | —       | 復元に使う明示的なキー（`vendor-build` の `vendor-key` を渡す） |
| `restore-keys`         | no       | —       | プレフィックス一致で stale キャッシュを復元するための順序付き複数行キー        |
| `enableCrossOsArchive` | no       | `false` | 他プラットフォームで保存したキャッシュを Windows runner で復元可能にする   |
| `fail-on-cache-miss`   | no       | `false` | キャッシュが見つからない場合にワークフローを失敗させる                    |
| `lookup-only`          | no       | `false` | ダウンロードせずキャッシュの存在のみ確認する                         |

## outputs

### `vendor-build`

| output         | 説明                                                               |
|----------------|------------------------------------------------------------------|
| `vendor-key`   | `vendor/` の保存に使った vendor キャッシュキー。`vendor-load` の `key` input に渡す |
| `composer-key` | Composer ダウンロードキャッシュのキー                                          |

### `vendor-load`

`actions/cache/restore@v5` と同一です。

| output              | 説明                              |
|---------------------|---------------------------------|
| `cache-hit`         | primary key で完全一致したかを示す boolean |
| `cache-primary-key` | 一致を試みた解決済みキャッシュキー               |
| `cache-matched-key` | 実際に復元されたキャッシュのキー                |

## キャッシュキー

`vendor-build` がキーを計算し、`vendor-key` として出力します。`vendor-load` はそれを受け取って復元するため、キー計算式は
`vendor-build` の一箇所にのみ存在します。

```bash
# Composer ダウンロードキャッシュ（composer.lock のみ。DL 内容は固定バージョンで決まる）
composer_hash=$(sha256sum composer.lock | awk '{print $1}')
# => ${RUNNER_OS}-composer-${composer_hash}

# vendor キャッシュ（composer.lock + composer.json。autoload 等 json の変更も反映するため）
vendor_hash=$( (sha256sum composer.lock; sha256sum composer.json) | sha256sum | awk '{print $1}')
# => ${RUNNER_OS}-vendor-${vendor_hash}
```

## 使用例

`vendor-build` の `vendor-key` 出力をジョブ出力経由で `vendor-load` の `key` に渡します。
こうすると `vendor-load` は `composer.lock` に依存せず、build が保存したキャッシュをキー一致で復元できます。

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      vendor-key: ${{ steps.vendor.outputs.vendor-key }}
    steps:
      - uses: actions/checkout@v6
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.2', tools: composer:v2 }
      - id: vendor
        uses: lepusinc/gha-composer/vendor-build@v1
        with: { working-directory: src-no-deposit }

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.2', tools: composer:v2, coverage: pcov }
      - uses: lepusinc/gha-composer/vendor-load@v1
        with:
          path: src-no-deposit/vendor
          key: ${{ needs.build.outputs.vendor-key }}
          fail-on-cache-miss: true
      - run: composer run test
        working-directory: src-no-deposit
```

## バージョニング

メジャータグ（`v1`）で参照してください。`vX.Y.Z` のリリースタグを push すると、
[`update-major-tag`](./.github/workflows/update-major-tag.yml) ワークフローが移動タグ `vX` を追従させます。

```yaml
- uses: lepusinc/gha-composer/vendor-build@v1
```

## 将来の拡張

composer 関連の action は `gha-composer/<action-name>/action.yml` として同リポジトリに追加します（例: `composer-audit`）。
