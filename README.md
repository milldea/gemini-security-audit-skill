# Gemini 向けセキュリティ監査スキル

このリポジトリは、Gemini でソースコードを起点としたセキュリティ監査を進めるためのスキルです。監査の進め方を示す手順書と、監査結果の形式を確認する検証スクリプトをまとめています。

Cloudflare の [security-audit-skill](https://github.com/cloudflare/security-audit-skill) を基に、事前調査、調査範囲の記録、脆弱性の探索、攻撃者視点での検証、検出事項の整理、独立した再確認、報告という流れを扱います。通常は監査の進め方を案内し、監査全体の実行は明示的に依頼された場合に限ります。詳しい動作モードと実行範囲は [SKILL.md](SKILL.md) を参照してください。

## 導入方法

このリポジトリのファイル一式を、監査したいプロジェクトの `.gemini/skills/security-audit/` に配置します。

```text
対象プロジェクト/
  .gemini/
    skills/
      security-audit/
        SKILL.md
        report-schema.json
        validate-coverage-ledger.cjs
        validate-findings.cjs
        ...その他の手順書とテスト...
```

`SKILL.md` は、ほかの手順書や検証スクリプトを相対パスで参照します。必要なファイルを同じディレクトリにまとめて置いてください。

監査で生成する `coverage-ledger.json` や `findings.json` は、このスキルの配布ファイルではありません。監査ごとに出力先を用意し、対象のソースコードや監査結果とともに適切に管理してください。

## 監査結果を検証する

監査が作成した2つの JSON ファイルは、付属のコマンドで確認できます。Node.js を利用できる環境で、次を実行してください。`<skill-dir>` はこのスキルを置いたディレクトリ、`<output-dir>` はその監査の出力ディレクトリです。

```sh
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
```

| コマンド | 主な確認内容 |
| --- | --- |
| `validate-coverage-ledger.cjs` | 調査単位に必要な項目、参照情報から作る `coverage_id` の整合性、ID の重複や並び順、調査状態・試行履歴・確認したパスの記録を確認します。 |
| `validate-findings.cjs` | 同じディレクトリにある `report-schema.json` に照らして、検出事項の必須項目や判定別の形式を確認します。さらに、fingerprint の重複や並び順、ソースパスなども確認します。 |

どちらも JSON の構文や入力ファイルの安全性を確認します。成功すると `PASS` と件数を表示し、終了コード `0` を返します。問題があれば `ERROR` と `FAIL`、または読み込み・解析に失敗した理由を表示し、終了コード `1` を返します。表示された項目を修正してから再実行してください。

**検証に通っても、脆弱性の実在や監査の網羅性が証明されるわけではありません。** これらのコマンドは記録の形式と内部の整合性を確認するものです。検出事項の根拠や再現結果は、[VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md) に従って別途確認してください。

### 検証スクリプト自体をテストする

付属の `*.test.cjs` は、上記の検証スクリプトが想定どおり動くかを確認するためのテストです。監査対象のリポジトリを調べるコマンドではありません。

```sh
node --test validate-coverage-ledger.test.cjs validate-findings.test.cjs
```

コマンドライン用の検証スクリプトは、OS に `O_NOFOLLOW` と `O_NONBLOCK` のサポートを求めます。Windows では該当する処理が安全側で失敗し、関連するコマンドラインテストも失敗します。監査結果の検証と全テストの実行には、これらをサポートする Unix 系 OS を使用してください。

## 出典とライセンス

このパッケージは [Cloudflare の security-audit-skill](https://github.com/cloudflare/security-audit-skill) を基にしています。元の MIT ライセンスと著作権表示は [LICENSE](LICENSE) に保持しています。

プロジェクト固有の監査メモや検出事項は含めていません。監査対象のソースコード、非公開の検出事項、認証情報、生成した監査結果は、この再利用可能なスキルのリポジトリとは別に管理してください。
