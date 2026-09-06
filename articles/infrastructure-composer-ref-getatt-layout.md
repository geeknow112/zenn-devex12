---
title: "なぜInfrastructure Composerは「縦一列」になるのか - Ref/GetAttとレイアウトの関係を検証する"
emoji: "🧩"
type: "tech"
topics: ["aws", "cloudformation", "iac", "個人開発"]
published: false
---

## この記事で得られること

- Infrastructure Composer（旧Application Composer）が「ノードを線で繋いだ関連図」になる条件・ならない条件
- CloudFormationの `Ref` / `GetAtt` が、Composerのレイアウトエンジンにとって何を意味しているか
- AWSの「IaCジェネレーター」がなぜあえて依存関係の薄いテンプレートを吐くのか、という設計意図の推測
- 自前でリソース関連図を作るとしたら、どう設計するかのラフスケッチ

Qiitaの方に「[アカウント全体を1枚の構成図にしようとして詰んだ話](https://qiita.com/)」として、実際に踏んだ罠を時系列でまとめました。この記事はその「なぜそうなるのか」の部分だけを掘り下げます。手順そのものより、Composerというツールの中身の挙動に興味がある人向けです。

## 前提：Composerは何を見てレイアウトを決めているのか

Infrastructure Composerは、CloudFormation/SAMテンプレートを読み込んでキャンバス上にリソースをカードとして並べるツールです。カード同士が線や入れ子構造で繋がるかどうかは、テンプレート中の `Ref` や `Fn::GetAtt` といった**組み込み関数による参照**の有無で決まります。

```yaml
# 依存関係が「ある」書き方
Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      Role: !GetAtt MyExecutionRole.Arn   # ← これがComposerにとっての「線」になる
  MyExecutionRole:
    Type: AWS::IAM::Role
```

`!GetAtt MyExecutionRole.Arn` の一文があるだけで、ComposerはMyFunctionとMyExecutionRoleに関連があると認識します。逆に言えば、この記述がなければComposerにとって2つのリソースは無関係な点でしかありません。

## 実際に起きたこと：2種類のテンプレートを比べる

個人のAWSアカウントで、CloudFormationの「IaCジェネレーター」機能を使い、既存リソースをスキャンしてテンプレート化してみました（1819件検出→標準リソース除外で451件に絞り込み）。これをComposerに読み込ませたところ、期待していたような相関図にはならず、**カードが縦一列に積み上がるだけ**でした。

原因を切り分けるために、生成元が異なる2種類のテンプレートを比較しました。

| テンプレートの出どころ | 取得コマンド | `Ref`/`GetAtt`の有無 | Composerでの見え方 |
|---|---|---|---|
| IaCジェネレーター（アカウントスキャン由来） | `get-generated-template` | ほぼ無し | 縦一列 |
| 既存スタック（人間 or SAM CLIが書いたテンプレート） | `get-template --output json` | 保持されている | 親カードに子リソースがネスト表示 |

IaCジェネレーターは「今アカウントに存在するリソースを漏れなく列挙する」ことを目的にしているため、リソース同士の関係性までは推論してくれません。一方、人間が書いた（あるいはSAM CLIがビルドした）テンプレートは、`!Ref` や `!GetAtt` を使って初めて動くように書かれているので、自然と依存関係が埋め込まれています。

つまり、**Composerが見ているのは「AWS上の実際のリソース同士の関係」ではなく、あくまで「テンプレートのテキストに書かれた参照」**です。これは実装として妥当ではありますが、「アカウントの実態を俯瞰したい」という当初の目的とは噛み合いませんでした。

なお、既存スタックのテンプレートを読み込ませた場合も、期待していた「矢印線で繋がるノードグラフ」ではなく、親リソースのカード内に子リソース（IAM Role、Permission、Deploymentなど）がリスト形式でネストされる、という表示でした。Composerの関連表現は「グラフ」というより「親子の入れ子」に近い、という点は実際に触ってみるまで分かりませんでした。

## なぜIaCジェネレーターは依存関係を書かないのか（推測）

公式ドキュメント上では、IaCジェネレーターの主目的は「既存リソースをCloudFormation管理下に**取り込む**（import）ための土台を作ること」とされています。取り込んだ後、人間がテンプレートを整理してResource間の参照を明示的に書き直すことが前提になっている、と読めます。

つまりIaCジェネレーターの出力は「完成品」ではなく「叩き台」であり、そこにComposerで一発で綺麗な相関図を求めるのは、そもそもの用途がズレていた、というのがここまでの検証で得た結論です。

## 自前で関連図を作るなら、という設計メモ

「矢印で繋がった関連図」がどうしても欲しい場合、テンプレートの `Ref` / `GetAtt` / `DependsOn` を自前でパースしてMermaidやGraphvizに変換するのが確実そうです。ラフに設計するなら:

1. テンプレートをJSONとしてロードし、`Resources` を走査
2. 各リソースのプロパティ値を再帰的に見て `{"Ref": "..."}` / `{"Fn::GetAtt": [...]}` / `DependsOn` を抽出
3. `リソースA → リソースB` のエッジリストを作る
4. Mermaidの `graph LR` 記法に変換して出力

```mermaid
graph LR
  MyFunction --> MyExecutionRole
```

この部分は今回未実装で、今後の課題として残しています。実装したら別記事にする予定です。

## 実務Tips（補足）

検証中に踏んだ、地味だが再現性の高い罠を2つだけ補足します。

- `aws cloudformation get-template --output yaml` は、PythonのYAMLライブラリ由来の `!!omap` タグ付きYAMLを出力するため、Composerに貼り付けても**エラーなく無視されます**。`--output json` を使ってください。
- 数十万文字級の大きいテンプレートをComposerに流し込むには、`Get-Content -Raw <file> | Set-Clipboard` でクリップボードにコピーし、テンプレートエディタでCtrl+A→Ctrl+Vする方法が確実でした。ブラウザのネイティブファイル選択ダイアログ経由よりも安定します。

## 参考リンク

- [AWS CloudFormation ユーザーガイド - 既存リソースからのIaC生成](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/generate-IaC.html)
- [Infrastructure Composer ドキュメント](https://docs.aws.amazon.com/infrastructure-composer/latest/dg/what-is-composer.html)

※本記事は2026年8月時点で個人アカウントを使って検証した実体験と、それに基づく推測を含みます。IaCジェネレーターの設計意図に関する記述は公式ドキュメントの記載からの推測であり、AWSの公式見解ではありません。
