---
name: terraform-reference-tracer
description: Terraform内の参照関係（特にCloudFrontのbehavior→(Lambda@Edge/CloudFront Functions)→archive_file/local_file/templatefile/file等）をたどり、経由した要素を含めて表形式でリストアップする。参照関係、依存関係、lambda_function_association、function_association、archive_file、local_file、index.js などの紐づけを整理したいときに使用する。
---

# Terraform 参照トレーサー

Terraform の「利用元 → 経由 → 利用先」を、**根拠行付きの表**として整理するための手順。

## いつ使うか（トリガー）

- CloudFront の `ordered_cache_behavior` / `default_cache_behavior` の `lambda_function_association` / `function_association` が **どのJS/zipに繋がるか**知りたい
- `index.js` / `index.js.tmpl` / `path/to/*.js` が **どの `archive_file` / `aws_lambda_function` 経由で参照されるか**知りたい
- `archive_file` / `local_file` / `templatefile()` / `file()` / `data.template_file` の **参照チェーン**を表で出したい

## 入力（ユーザーに確認する情報）

最低限どれか1つを起点として扱う（複数あれば優先度順に使う）。

- **CloudFront側の起点**: `path_pattern`（例: `/*`）＋ `event_type`（例: `viewer-request`）
- **Lambda/Function側の起点**: `aws_lambda_function.<name>` / `aws_cloudfront_function.<name>` / `data.archive_file.<name>` などのアドレス
- **ファイル側の起点**: `function/**/index.js` や `*.tmpl` などのパス

## 出力テンプレ（Markdown表）

このテンプレをそのまま埋めて出力する。

```markdown
| No | 始点(利用元) | 経由(上流→下流) | 終点(利用先) | 根拠(ファイル:行) | メモ |
|---:|---|---|---|---|---|
| 1 |  |  |  |  |  |
```

### 詳細テンプレ（zip/生成物も明示したい場合）

zip 生成や「どのファイルが zip 内に入るか」まで追いたい場合は、こちらを使う。

```markdown
| No | 始点(利用元) | path_pattern | event_type | 経由(上流→下流) | 参照先ARN/ID | zip出力(生成物) | zip入力(実体) | 根拠(ファイル:行) | メモ |
|---:|---|---|---|---|---|---|---|---|---|
| 1 |  |  |  |  |  |  |  |  |  |
```

### 記入ルール

- **始点(利用元)**: 「どこから参照が始まるか」（例: `aws_cloudfront_distribution.front (path_pattern="/") viewer-request`）
- **経由**: 参照を 1段ずつ `→` で並べる（例: `aws_lambda_function.X → data.archive_file.Y → data.template_file.Z → file(".../index.js.tmpl")`）
- **終点(利用先)**: 最終的に辿り着いた“実体”（例: `.../index.js.tmpl`、`.../function/app_decision/`、`module.*` 等）
- **根拠**: 各段の「参照が書かれている行」を最低1つは含める（複数あるときは `,` 区切り）
  - 例: `common/cloudfront_front.tf:112-115, common/lambda.tf:214-222`
- **途中で止める場合**: `module.*` / `var.*` などに到達したら、終点にそれを書き、メモに「モジュール/変数境界のためここで停止」と記載

## ワークフロー（手順）

### 1) 起点を確定

- CloudFront起点の場合は **(path_pattern, event_type)** を特定する
- Lambda/ファイル起点の場合は **対象名（`aws_lambda_function.<name>` など）** を特定する

### 2) 起点の“参照文字列”を抽出

以下のいずれかを見つけ、参照されているアドレス/ファイルパスを抜き出す。

- **CloudFront → Lambda@Edge**
  - `lambda_function_association { ... lambda_arn = ... }`
  - 典型: `${aws_lambda_function.<name>.arn}:${aws_lambda_function.<name>.version}`
- **CloudFront → CloudFront Functions**
  - `function_association { ... function_arn = aws_cloudfront_function.<name>.arn }`
- **Terraform → ファイル**
  - `file("...")` / `templatefile("...", {...})`
- **Terraform → archive**
  - `data.archive_file.<name>.output_path` / `output_base64sha256`

### 3) 参照を“1段ずつ”展開（最大でも必要な分だけ）

次の「展開ルール」に従い、参照先を辿って **経由** を埋める。

#### 展開ルール: `aws_lambda_function`

- `lambda_arn` で参照されている `aws_lambda_function.<name>` を見つけたら、次を確認する
  - `filename = data.archive_file.<x>.output_path` のような **zipへの参照**
  - `source_code_hash = data.archive_file.<x>.output_base64sha256` のような **hash依存**
- 次の段は通常 `data.archive_file.<x>` になる

#### 展開ルール: `data.archive_file`

`data "archive_file" "<x>"` の中身から **入力ソース**と**出力zip**を拾う。

- `source_file = ".../index.js"` → 終点はそのファイルパス
- `source_dir = ".../function/foo/"` → 終点はそのディレクトリ（必要ならさらにその中の `index.js` などを探索）
- `source { content = ... filename = "index.js" }`
  - `content` が `data.template_file.<t>.rendered` 等なら次段は `data.template_file.<t>`
  - `filename` は zip 内のファイル名（表のメモ欄に残すと良い）
- `output_path = ".../dst/foo.zip"` は「生成物」のため、表ではメモ扱い（参照チェーンの“終点”は入力ソース側に置くのが分かりやすい）

#### 展開ルール: `local_file`

`resource "local_file" "<x>"` を見つけたら、通常は次を辿る。

- `content` → `templatefile()` / `file()` / `data.template_file.*.rendered` 等
- `filename` → 生成されるファイルのパス（メモ欄に残す）

#### 展開ルール: `data.template_file` / `templatefile()`

- `data "template_file" "<t>"` の `template = file(".../*.tmpl")` に到達したら、終点はそのテンプレパス
- `templatefile(".../*.tmpl", {...})` に到達したら、終点はそのテンプレパス

#### 展開ルール: `aws_cloudfront_function`

- `code = file(".../*.js")` → 終点はそのファイル
- `code = templatefile(".../*.tmpl", {...})` → 終点はそのテンプレ

#### 展開ルール: `module.*`（モジュール境界を越える）

`module.<name>.<attr>` に到達したら、次の手順で「モジュール内の実体」に追跡する。

1. **呼び出し元で module ブロックを特定**する
   - `module "<name>" { source = ... }`
2. `source` を評価し、どれかに分類する
   - **ローカルパス**（例: `./modules/foo` や `${path.module}/modules/foo`）  
     → そのディレクトリ内を追跡対象に含める
   - **Git/Registry等のリモート**（例: `git::...` / `hashicorp/...`）  
     → このリポジトリからは実体が読めない可能性があるため、表のメモに「リモートmoduleのため追跡停止」と書いて止める
3. 参照している `<attr>` が何かで分岐
   - **`module.<name>.<output>` 形式**: モジュール内の `output "<output>" { value = ... }` を探し、`value` を終点としてさらに展開する
   - **`module.<name>` が resources を内包する設計**: モジュール内で直接使われている resource（例: `aws_*`）に辿りたい場合は、モジュール内の該当 resource を検索して「終点」または「経由」に追加する

#### 展開ルール: `var.*`（変数境界）

`var.<name>` に到達したら、基本はそこで止める（値は環境/呼び出し元で変わるため）。

- 同一ディレクトリで `variable "<name>" { ... }` が見つかる場合は、型/説明をメモ欄に残す
- もし `.tfvars` や `terraform.tfvars` 等で明示されている値まで追跡したい場合は、**追跡範囲に含めるファイル**をユーザーから指定してもらってから辿る

### 4) 表を完成させる（根拠行を必ず添える）

- チェーンの各段について、**その参照が書かれている箇所**の `ファイル:行` を最低1つは記載する
- 同じ終点に複数の始点がぶら下がる場合は、行を分けるか、Noを増やして列挙する

## 逆引き（ファイル/テンプレ起点 → 利用元へ辿る）

「この `index.js` / `*.tmpl` がどこから使われているか？」を知りたい場合の手順。

1. **終点（ファイルパス）を起点に検索**
   - `file("...")` / `templatefile("...")` / `source_file = "...")` / `source_dir = "..."` などで参照されていないか探す
2. 参照しているブロックを見つけたら、そのブロック種別ごとに上流へ
   - `data.archive_file.<x>` を見つけたら → それを参照している `aws_lambda_function.<y>`（`filename`/`source_code_hash`）へ
   - `aws_lambda_function.<y>` を見つけたら → それを参照している `lambda_function_association`（CloudFront）へ
   - `aws_cloudfront_function.<y>` を見つけたら → それを参照している `function_association`（CloudFront）へ
3. それぞれ表の **始点(利用元)** を埋める
   - CloudFront配下なら `path_pattern` / `event_type` を列に書くと一覧性が上がる

## 例（このリポジトリでの典型チェーン）

※あくまで「どう埋めるか」の例。実際の出力では必ず根拠行を埋める。

```markdown
| No | 始点(利用元) | 経由(上流→下流) | 終点(利用先) | 根拠(ファイル:行) | メモ |
|---:|---|---|---|---|---|
| 1 | aws_cloudfront_distribution.front (path_pattern=\"/\") viewer-request | aws_lambda_function.rewrite_benefit_disney_exclusion → data.archive_file.rewrite_benefit_disney_exclusion → data.template_file.rewrite_benefit_disney_exclusion → file(\"${path.module}/function/rewrite_benefit_disney_exclusion/index.js.tmpl\") | ${path.module}/function/rewrite_benefit_disney_exclusion/index.js.tmpl | common/cloudfront_front.tf:112-115, common/lambda.tf:204-222, common/lambda.tf:194-201 | zip内filenameは index.js |
```

## 仕上げのチェック（落とし穴）

- `ordered_cache_behavior` の中に **同名/同一lambda** が複数回出ることがある（`viewer-request` と `viewer-response` など）
- `dynamic` block / `for_each` で **複数挙動が生成**されている場合は、表の「始点」を具体化して書く（例: `env==stg` など）
- `module.*` / `var.*` に到達したら、そこから先は **このスキルのスコープ外**として止め、メモに境界を書いておく
