# Composer GitHub Actions

PHP プロジェクトの「Composer キャッシュ + 依存インストール」を共通化する composite action 群です。

> 設計判断・キャッシュキーの内部仕様・リリース手順などコントリビュート向けの情報は [AGENTS.md](./AGENTS.md) を参照してください。

## 前提

- **PHP / Composer は呼び出し側でセットアップ済みであること。** これらの action は `shivammathur/setup-php`
  を実行しません。先に `shivammathur/setup-php` 等で PHP / Composer を用意してください
  （`vendor-load` 自体は PHP / Composer を必要としません）。
- `composer.lock` をコミットしているリポジトリ向けです。`vendor-build` は build ジョブのチェックアウトにある
  `composer.lock` / `composer.json` からキャッシュキーを計算します。

## アクション

### [`vendor-build`](./vendor-build)

ビルドジョブ用。Composer ダウンロードと `vendor` をキャッシュし、`vendor` キャッシュミス時に `composer install`
を実行します。使ったキャッシュキーを outputs として公開します。

#### inputs

| input               | required | default | 説明                                          |
|---------------------|----------|---------|---------------------------------------------|
| `working-directory` | no       | `.`     | `composer.json`（と `composer.lock`）があるディレクトリ |

#### outputs

| output         | 説明                                                               |
|----------------|------------------------------------------------------------------|
| `vendor-key`   | `vendor/` の保存に使った vendor キャッシュキー。`vendor-load` の `key` input に渡す |
| `composer-key` | Composer ダウンロードキャッシュのキー                                          |

### [`vendor-load`](./vendor-load)

テスト・静的解析ジョブ用。`vendor-build` が保存した `vendor` キャッシュを復元します。
`actions/cache/restore@v5` の薄いラッパーで、inputs / outputs は `actions/cache/restore` と同一です。
`key` には `vendor-build` の `vendor-key` 出力を渡します。

#### inputs

| input                  | required | default | 説明                                             |
|------------------------|----------|---------|------------------------------------------------|
| `path`                 | yes      | —       | 復元するファイル・ディレクトリ・ワイルドカードのリスト（例: `src/vendor`）   |
| `key`                  | yes      | —       | 復元に使う明示的なキー（`vendor-build` の `vendor-key` を渡す） |
| `restore-keys`         | no       | —       | プレフィックス一致で stale キャッシュを復元するための順序付き複数行キー        |
| `enableCrossOsArchive` | no       | `false` | 他プラットフォームで保存したキャッシュを Windows runner で復元可能にする   |
| `fail-on-cache-miss`   | no       | `false` | キャッシュが見つからない場合にワークフローを失敗させる                    |
| `lookup-only`          | no       | `false` | ダウンロードせずキャッシュの存在のみ確認する                         |

#### outputs

| output              | 説明                              |
|---------------------|---------------------------------|
| `cache-hit`         | primary key で完全一致したかを示す boolean |
| `cache-primary-key` | 一致を試みた解決済みキャッシュキー               |
| `cache-matched-key` | 実際に復元されたキャッシュのキー                |

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

メジャータグ（`v1`）で参照してください。パッチ・マイナーリリースに自動追従します。

```yaml
- uses: lepusinc/gha-composer/vendor-build@v1
```
