# Gemini 向けセキュリティ監査スキル（gemini-security-audit-skill）

このリポジトリは、Gemini（Gemini CLI / Antigravity 等）でソースコードを起点としたセキュリティ監査を進めるためのスキルパッケージです。監査の進め方を定義した手順書（15ファイル）と、監査記録および検出結果の整合性を機械的に確認する検証スクリプトをまとめています。

Cloudflare 社が公開している [security-audit-skill](https://github.com/cloudflare/security-audit-skill) を基に、Gemini 環境での運用に向けて整備しました。元のスキルは、Cloudflare 社の技術ブログ記事 [Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness) で紹介された脆弱性探索ハーネスの出発点となったものです。

---

## 設計原則

本スキルは、次の5つの原則に基づいて誤検知（ハルシネーション）を抑え、修正に直結する監査結果を出力します。

1. **信頼境界の突破と実害の裏付け**:
   単なるベストプラクティスからの逸脱や推測は、脆弱性として報告しません。「どの権限のアクターが、どの入力経路から、どの防御制御を越えて、どの資産に具体的な影響を与えるか」がソースコード上で裏付けられたものだけを確定（`confirmed`）とします。
2. **発見者と検証者を分離した反証検証**:
   脆弱性候補を見つけたエージェント自身には最終判定を行わせません。独立した検証ステップ（Phase 3 / Phase 5）で該当コード・共通モジュール・設定・既存テストを1行単位で突き合わせ、成立しない候補を棄却（`rejected`）します。
3. **ソースコードから確認できる範囲の厳守**:
   クラウドの IAM 設定や WAF、リバースプロキシ、外部システム仕様など、リポジトリ内のコードだけでは確認できない要素を推測で断定しません。未確認の前提条件を明示した上で「要追加確認（`needs_validation`）」として切り分け、安全な確認手順を提示します。
4. **不確実性と深刻度の分離**:
   深刻度（`critical`〜`informational`）は、影響が裏付けられた `confirmed` の検出事項にのみ付与します。`needs_validation` には推測の深刻度を付けません。また、既存の防御層で防がれているものの多層防御として改善が望ましい箇所は、改善提案（`hardening`）として区別します。
5. **カバレッジ台帳による網羅性管理と継続的な監査**:
   サブシステムと攻撃分類を掛け合わせたカバレッジ台帳（`coverage-ledger.json`）を作成し、調査済み範囲と未調査範囲を追跡します。複数回の監査や差分監査では、前回の台帳と検出結果を引き継いで未調査領域や変更箇所を重点的に調査できます。

---

## 監査の6フェーズと出力ファイル

全体監査モード（Full audit mode）では、以下の6つのフェーズを順番に実行します。

| フェーズ | 名称 | 主な実施内容と出力ファイル |
| :--- | :--- | :--- |
| **Phase 1** | **事前調査**<br>（Reconnaissance） | リポジトリ全体の構成、アクター（権限種別）、外部入力を受け取るエンドポイント、信頼境界、認証・認可の仕組みを解析し、`architecture.md` と初期カバレッジ台帳 `coverage-ledger.json` を作成します。 |
| **Phase 2** | **カバレッジ主導の脆弱性探索**<br>（Coverage-led hunting） | カバレッジ台帳の調査単位ごとに探索を行い、8つの基本攻撃分類および技術領域別の専門ガイドに基づいて脆弱性候補を洗い出します。調査状況は随時 `coverage-ledger.json` に記録します。 |
| **Phase 3** | **独立した反証検証**<br>（Candidate validation） | 重複を排除した全候補に対し、探索時とは別の視点から実コード・設定・既存テストを照合して反証を試みます。防御済み・仕様どおりの候補を除外し、成立する問題と最小限の修正案を特定します。 |
| **Phase 4** | **構造化データの出力とスキーマ検証**<br>（Structured output） | すべての候補を `confirmed`（確認済み）、`needs_validation`（要追加確認）、`rejected`（棄却）の3区分で `findings.json` に出力し、`report-schema.json` および検証スクリプトで形式と整合性を検証します。 |
| **Phase 5** | **記録の独立再検証**<br>（Independent record verification） | 最終的な `findings.json` に記載されたファイルパス、行番号、コード根拠、修正案が現在のソースコードと一致しているかを再確認します。 |
| **Phase 6** | **レポート作成**<br>（Target-neutral reporting） | 検証済みの記録とカバレッジ台帳から、経営層・管理者から開発担当者まで参照できる `REPORT.md`、`FINDINGS-DETAIL.md`、`NEEDS-VALIDATION.md` を出力します。 |

### 判定区分（Verdict）と深刻度（Severity）

| 判定区分 | 内容 | 深刻度の付与 |
| :--- | :--- | :--- |
| `confirmed`<br>（確認済み） | ソースコード上の経路と反証検証により、信頼境界の突破と具体的な影響が裏付けられた脆弱性。発生条件、成立経路、最小限の修正コード案と回帰テスト案を記録します。 | あり（`critical` / `high` / `medium` / `low` / `informational`） |
| `needs_validation`<br>（要追加確認） | ソースコード上に懸念経路があるものの、リポジトリ外の環境設定（クラウド IAM、WAF、プロキシ設定等）の確認が必要な項目。未確認の事実と安全な確認手順を記録します。 | なし（推測での深刻度付与は行わない） |
| `rejected`<br>（棄却） | 反証検証により、既存のフィルタや設定等で安全に防御されていることが確認された項目。安全確認の根拠として記録します。 | なし |

---

## 動作モードと実行プロファイル

### 2つの動作モード

本スキルは、依頼内容に応じて2つのモードで動作します。詳しい仕様は [SKILL.md](SKILL.md) を参照してください。

**Guidance mode（既定）** は、特定のセキュリティ質問や一部コードの重点レビュー、脆弱性パターンの調査、修正方針の相談に使います。全6フェーズの自動実行や監査ディレクトリの生成は行わず、対象箇所の調査結果を直接回答します。

**Full audit mode（全体・本格監査）** は、コードベース全体のセキュリティ監査やペネトレーションテスト、または監査レポート一式の作成を明示的に依頼された場合に使います。全6フェーズを実行し、検証スクリプトを通過した構造化データとレポート一式を出力します。

### 実行プロファイルとスコープ指定

Full audit mode では、監査対象の規模や目的に応じて3つのプロファイルと対象範囲（全体監査 / 部分監査）を選択できます。どのプロファイルを選んだ場合でも、反証検証やスキーマ検証の基準は変わりません。

| プロファイル / 範囲 | 特徴と適した用途 |
| :--- | :--- |
| `standard`（既定） | 標準的な網羅性と独立検証を行う基本プロファイルです。 |
| `quick` | 小規模な対象や再監査、迅速な初回確認向けのプロファイルです。調査単位を集約して1ウェーブで走査します。 |
| `deep` | 大規模システムや高い安全性が求められる対象向けに、サブシステムやライフサイクル単位で細かく分割して多重検証を行うプロファイルです。 |
| スコープ指定（Scoped run） | 特定のディレクトリ、サブシステム、またはコミット間の差分（diff）に対象を絞って監査を実施します。対象外の領域は `out_of_scope` として台帳に記録します。 |

---

## 安全対策と実行境界

監査対象のソースコードはすべて読み取り専用（read-only）で扱い、リポジトリ内のファイルを直接書き換えることはありません。監査で生成するファイルは、原則として対象リポジトリ外の専用ディレクトリに保存します。既定の出力先は `~/security-audit-skill/<repo-name>/run-<N>` です。

また、稼働中の本番サーバーやテスト環境、外部サービス、課金が発生する外部 API への通信や攻撃試行は一切行いません。解析にはダミーの識別子とソースコード・設定定義のみを用い、本番の個人情報や認証情報は使用しません。

---

## 同梱ファイル一覧

| ファイル名 | 役割・内容 |
| :--- | :--- |
| `SKILL.md` | スキル全体の設定、基本原則、安全境界、動作モード、プロファイル、6フェーズの全体定義 |
| `RECONNAISSANCE.md` | Phase 1（事前調査）の調査項目、信頼境界マッピング、カバレッジ台帳の設計手順 |
| `HUNTING.md` | Phase 2（脆弱性探索）の進行手順、カバレッジ検証（Critic）、候補抽出ルール |
| `ATTACK-CLASSES.md` | 基本8分類（インジェクション、アクセス制御、ファイル操作、暗号・秘密情報、ビジネスロジック、機能悪用・情報漏洩、連鎖脆弱性、基本設定・既知脆弱性）の調査ガイド |
| `WEB-PROTOCOL-AND-AUTH.md` | Web プロトコル、HTTP リクエスト処理、キャッシュ、セッション・認証認可プロトコル向けの専門調査ガイド |
| `CLIENT-SIDE.md` | フロントエンド・ブラウザ向け（DOM XSS、PostMessage 等のメッセージ境界、UI 偽装、Prototype Pollution 等）の専門調査ガイド |
| `CLOUD-AND-DEPLOYMENT.md` | クラウド設定、IaC（Terraform / CloudFormation 等）、コンテナ、サーバーレス、Ingress 向けの専門調査ガイド |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | マルチテナント分離、キャッシュ、検索、エクスポート、バックアップ、データ削除ライフサイクル向けの専門調査ガイド |
| `AI-AND-LLM.md` | LLM・AI 連携機能（プロンプトインジェクション、ツール呼び出し権限、出力処理等）向けの専門調査ガイド |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、シリアライズ・デシリアライズ、キュー、Webhook、ストリーミング通信向けの専門調査ガイド |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | リソース枯渇（DoS）、クォータ制限、ワーカー・キュー詰まり、API 課金悪用向けの専門調査ガイド |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依存ライブラリ、CI/CD パイプライン、リリース署名、プラグイン機構向けの専門調査ガイド |
| `MEMORY-SAFETY-AND-BINARY.md` | ネイティブコード・バイナリ処理向けのメモリ安全性調査ガイド |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | デスクトップ・モバイルアプリ、Deep Link、WebView、ローカル IPC 向けの専門調査ガイド |
| `VALIDATION-AND-REPORTING.md` | Phase 3〜6 の反証検証、構造化出力、独立再検証、レポート作成手順 |
| `report-schema.json` | `findings.json` の全判定区分（`confirmed` / `needs_validation` / `rejected`）を定義する JSON Schema |
| `validate-coverage-ledger.cjs` | Phase 1〜5 で生成・更新される `coverage-ledger.json` の整合性検証スクリプト（外部依存なし） |
| `validate-findings.cjs` | Phase 4〜5 で生成される `findings.json` のスキーマ・整合性検証スクリプト（外部依存なし） |
| `validate-coverage-ledger.test.cjs` | `validate-coverage-ledger.cjs` 自体の単体テスト |
| `validate-findings.test.cjs` | `validate-findings.cjs` 自体の単体テスト |

---

## 導入方法と使い方

### 1. 導入方法

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
        ...その他の手順書（.md）とテスト...
```

`SKILL.md` は、同じディレクトリ内の手順書や検証スクリプトを相対パスで参照します。すべてのファイルを同じディレクトリにまとめて配置してください。

監査で生成する `coverage-ledger.json` や `findings.json`、各種レポート（`REPORT.md` 等）はこのスキルの配布ファイルではありません。監査ごとに出力ディレクトリを用意し、対象のソースコードや監査結果とともに適切に管理してください。

### 2. 監査の実行方法（プロンプト例）

対象プロジェクトのワークスペースで Gemini エージェントを起動し、目的に応じて以下のように指示します。

リポジトリ全体のセキュリティ監査（Full audit mode）を実行する場合:
```text
security-audit スキルを使って、このリポジトリ全体のセキュリティ監査を実施し、レポート一式を出力してください。
```

対象ディレクトリや機能を絞った部分監査（Scoped audit）を実行する場合:
```text
security-audit スキルを使って、./app/controllers と ./app/models を対象にスコープ指定のセキュリティ監査を実施してください。
```

特定の観点や質問に対する調査（Guidance mode）を行う場合:
```text
security-audit スキルを参照して、現在の認証・セッション管理の実装に権限昇格や認可漏れの懸念がないか確認してください。
```

---

## 監査結果を検証する

監査によって作成された2つの JSON ファイル（`coverage-ledger.json` と `findings.json`）は、付属の Node.js スクリプトで機械的に検証できます。`<skill-dir>` はこのスキルを配置したディレクトリ、`<output-dir>` は監査結果の出力ディレクトリです。

```sh
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
```

| コマンド | 主な確認内容 |
| :--- | :--- |
| `validate-coverage-ledger.cjs` | 調査単位に必要な項目、参照情報から生成される `coverage_id` の整合性、ID の重複や並び順、調査状態・試行履歴・確認したパスの記録を検証します。 |
| `validate-findings.cjs` | 同じディレクトリにある `report-schema.json` に照らして、検出事項の必須項目や判定区分別の形式を検証します。さらに、fingerprint の重複や並び順、ソースパスなども確認します。 |

どちらも JSON の構文や入力ファイルの安全性を確認します。成功すると `PASS` と件数を表示し、終了コード `0` を返します。問題があれば `ERROR` と `FAIL`、または読み込み・解析に失敗した理由を表示し、終了コード `1` を返します。表示された項目を修正してから再実行してください。

**検証に通っても、脆弱性の実在や監査の網羅性が証明されるわけではありません。** これらのコマンドは記録の形式と内部の整合性を確認するものです。検出事項の根拠や再現結果は、[VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md) に従って別途確認してください。

### 検証スクリプト自体をテストする

付属の `*.test.cjs` は、上記の検証スクリプトが想定どおり動作するかを確認するためのテストです。監査対象のリポジトリを調べるコマンドではありません。

```sh
node --test validate-coverage-ledger.test.cjs validate-findings.test.cjs
```

コマンドライン用の検証スクリプトは、OS に `O_NOFOLLOW` と `O_NONBLOCK` のサポートを求めます。Windows では該当する処理が安全側で失敗し、関連するコマンドラインテストも失敗します。監査結果の検証と全テストの実行には、これらをサポートする Unix 系 OS（Linux / macOS / WSL2 等）を使用してください。

---

## 出典とライセンス

このパッケージは [Cloudflare の security-audit-skill](https://github.com/cloudflare/security-audit-skill) を基にしています。元の MIT ライセンスと著作権表示は [LICENSE](LICENSE) に保持しています。

プロジェクト固有の監査メモや検出事項は含めていません。監査対象のソースコード、非公開の検出事項、認証情報、生成した監査結果は、この再利用可能なスキルのリポジトリとは別に管理してください。
