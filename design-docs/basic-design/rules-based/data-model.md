# データモデリング設計書（要件定義書 + CLAUDE.mdルール準拠版）

## 1. 本書の位置づけ

本書は、`design-docs/requirements/requirements.md`（要件定義書）と `design-docs/basic-design/CLAUDE.md`
（データモデリング・ER図作成ルール）の2文書のみを根拠として行ったデータモデリングの結果をまとめた
基本設計書である。他の設計書・参考資料は参照していない。

## 2. 対象システムの概要

オンラインで技術書・技術関連アイテム（USBメモリ、モバイルバッテリーなどの技術ガジェットを中心とする）
を販売するECシステム。要件定義書に挙げられた業務要件は以下の4点。

1. 商品・カテゴリの管理
2. オンライン販売履歴の管理、販売に先立つ利用者（会員）登録・管理
3. 商品の仕入れ・出荷の管理、仕入れ先の管理
4. 入庫（仕入れた商品）・出庫（売れた商品）の管理、月初の棚卸しとその記録管理
   （在庫は棚卸し記録を基準に当月の入出庫数で調整した値とする）

## 3. 用語の整理

要件定義書の業務要件には「仕入れ」「出荷」（要件3）と「入庫」「出庫」（要件4）という、意味の近い
4つの語が別々に列挙されている。これらは同一語の言い換えではなく、別々の業務プロセスを指すものと
解釈する。

| 語 | 解釈 | 層 |
|---|---|---|
| 仕入れ | 仕入れ先への発注という契約行為 | 発注層 |
| 入庫 | 発注した商品が実際に倉庫に搬入される実績 | 実績層 |
| 販売（受注） | 会員による購買の申込み行為 | 受注層 |
| 出荷 | 受注に応じて商品を配送のために送り出す行為（まとめ・分割が発生しうる） | 出荷層 |
| 出庫 | 出荷に対応して実際に倉庫から商品が搬出される実績 | 実績層 |

この整理に基づき、CLAUDE.mdのドメイン依存ルールに従って以下の構造を採用する。

- **仕入れ→入庫（2層構造）**: 仕様上「仕入れ」に対応する多重関連語（「出荷」に相当する語）が
  存在しないため、三層化しない。仕入れ（発注）1件に対し、納期のズレ等により入庫（実績）が複数回に
  分かれる（分納）ことを許容し、仕入れ 1 : N 入庫とする。
- **受注→出荷・出庫（3層構造）**: 受注（受注明細）と出荷は実務上多対多になる（1受注が複数回に分けて
  出荷される「分割出荷」、複数の受注が1回の出荷にまとめられる「まとめ出荷」）。CLAUDE.mdの
  「多対多は中間テーブルで解消する」ルールに従い、受注明細と出荷の両方を参照する中間テーブル
  「出庫」を新設してこの多対多を解消する。

## 4. テーブル分類

CLAUDE.mdの「テーブル分類を明示する」ルールに従い、各テーブルを分類する。

| 分類 | 定義 | 対象テーブル |
|---|---|---|
| マスターテーブル | 管理者が登録する | category（カテゴリ）, product（商品）, supplier（仕入れ先） |
| ステートテーブル | 利用者が何度も更新する | member（会員） |
| トランザクションテーブル | 追加後は基本的に更新されない | purchase_order, purchase_order_detail, stock_in, stock_in_detail, sales_order, sales_order_detail, shipment, stock_out（中間テーブル）, stock_taking, stock_taking_detail |

会員（member）は、要件上「利用者」自身が登録する対象であり、登録後も住所・連絡先等を利用者本人が
繰り返し更新しうるため、「管理者が登録するマスター」ではなく「利用者が何度も更新するステート」に
分類する。

## 5. エンティティ一覧

| # | エンティティ（論理名） | 物理名 | 分類 | 概要 |
|---|---|---|---|---|
| 1 | カテゴリ | category | マスター | 商品を分類するカテゴリ |
| 2 | 商品 | product | マスター | 販売対象となる商品 |
| 3 | 仕入れ先 | supplier | マスター | 商品の仕入れ元となる取引先 |
| 4 | 会員 | member | ステート | 販売に先立って登録される利用者 |
| 5 | 仕入れ | purchase_order | トランザクション | 仕入れ先への発注記録（ヘッダ） |
| 6 | 仕入れ明細 | purchase_order_detail | トランザクション | 仕入れの商品別内訳 |
| 7 | 入庫 | stock_in | トランザクション | 仕入れた商品が倉庫に搬入された実績（ヘッダ） |
| 8 | 入庫明細 | stock_in_detail | トランザクション | 入庫の商品別内訳 |
| 9 | 受注 | sales_order | トランザクション | 会員による購買の申込み記録（ヘッダ） |
| 10 | 受注明細 | sales_order_detail | トランザクション | 受注の商品別内訳 |
| 11 | 出荷 | shipment | トランザクション | 受注に応じて配送のために送り出す記録（ヘッダ） |
| 12 | 出庫 | stock_out | トランザクション（中間） | 受注明細と出荷を結び付け、倉庫から搬出された実績を表す中間テーブル |
| 13 | 棚卸し | stock_taking | トランザクション | 月初に実施する棚卸しの記録（ヘッダ） |
| 14 | 棚卸し明細 | stock_taking_detail | トランザクション | 棚卸しの商品別の実棚卸数量 |

## 6. エンティティ間の関連

```
category         1 ─── N  product

product          1 ─── N  purchase_order_detail
product          1 ─── N  stock_in_detail
product          1 ─── N  sales_order_detail
product          1 ─── N  stock_out
product          1 ─── N  stock_taking_detail

supplier         1 ─── N  purchase_order
purchase_order   1 ─── N  purchase_order_detail
purchase_order   1 ─── N  stock_in                 （分納対応）
purchase_order_detail 1 ─── N  stock_in_detail
stock_in         1 ─── N  stock_in_detail

member           1 ─── N  sales_order
member           1 ─── N  shipment                 （まとめ出荷時、受注を介さず会員を直接参照）
sales_order      1 ─── N  sales_order_detail
sales_order_detail 1 ─── N  stock_out              （分割出荷対応）
shipment         1 ─── N  stock_out                （まとめ出荷対応）

stock_taking     1 ─── N  stock_taking_detail
```

出荷（shipment）が会員（member）を直接参照するのは、「まとめ出荷」により複数の受注（複数の
sales_order）の商品が1回の出荷にまとめられる場合があり、出荷の配送先は特定の受注1件に従属しない
ためである。出庫（stock_out）は受注明細と出荷の両方の親が揃わないと作成できない中間テーブルであり、
これによって「1受注明細が複数回の出荷に分かれる（分割出荷）」「複数の受注明細が1回の出荷にまとめ
られる（まとめ出荷）」の両方を表現できる。

## 7. テーブル定義

### 7.1 category（カテゴリ）─ マスター

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| category_id | カテゴリID | BIGINT | PK | NOT NULL | 主キー |
| category_name | カテゴリ名 | VARCHAR(100) | | NOT NULL | 例: 技術書／USBメモリ／モバイルバッテリー |

### 7.2 product（商品）─ マスター

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| product_id | 商品ID | BIGINT | PK | NOT NULL | 主キー |
| category_id | カテゴリID | BIGINT | FK → category | NOT NULL | 所属カテゴリ |
| product_code | 商品コード | VARCHAR(50) | | NOT NULL | 一意 |
| product_name | 商品名 | VARCHAR(200) | | NOT NULL | |
| standard_selling_price | 標準販売単価 | DECIMAL(10,2) | | NOT NULL | マスタ上の参考単価。実際の販売単価は受注明細にスナップショットする |
| standard_purchase_price | 標準仕入単価 | DECIMAL(10,2) | | NOT NULL | マスタ上の参考単価。実際の仕入単価は仕入れ明細にスナップショットする |

### 7.3 supplier（仕入れ先）─ マスター

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| supplier_id | 仕入れ先ID | BIGINT | PK | NOT NULL | 主キー |
| supplier_name | 仕入れ先名 | VARCHAR(200) | | NOT NULL | |

### 7.4 member（会員）─ ステート

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| member_id | 会員ID | BIGINT | PK | NOT NULL | 主キー |
| member_name | 会員名 | VARCHAR(100) | | NOT NULL | |
| email | メールアドレス | VARCHAR(255) | | NOT NULL | 一意 |
| shipping_address | 配送先住所 | VARCHAR(300) | | NOT NULL | 会員自身が繰り返し更新しうる |
| registered_on | 登録日 | DATE | | NOT NULL | 販売に先立つ会員登録の完了日 |

### 7.5 purchase_order（仕入れ）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| purchase_order_id | 仕入れID | BIGINT | PK | NOT NULL | 主キー |
| supplier_id | 仕入れ先ID | BIGINT | FK → supplier | NOT NULL | 発注先 |
| ordered_on | 発注日 | DATE | | NOT NULL | |
| is_cancelled | 取消フラグ | BOOLEAN | | NOT NULL | 数量集計だけでは表現できない取消の状態を個別に保持する |

進行状況（未入庫／一部入庫／入庫完了）はカラムに持たせず、入庫（stock_in_detail）の実績集計から
動的に導出する。

### 7.6 purchase_order_detail（仕入れ明細）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| purchase_order_detail_id | 仕入れ明細ID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_id | 仕入れID | BIGINT | FK → purchase_order | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| ordered_quantity | 発注数量 | INTEGER | | NOT NULL | |
| purchase_unit_price | 仕入単価 | DECIMAL(10,2) | | NOT NULL | 発注時点の単価のスナップショット |

### 7.7 stock_in（入庫）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_in_id | 入庫ID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_id | 仕入れID | BIGINT | FK → purchase_order | NOT NULL | 分納により1仕入れに対し複数の入庫が発生しうる |
| received_on | 入庫日 | DATE | | NOT NULL | |

### 7.8 stock_in_detail（入庫明細）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_in_detail_id | 入庫明細ID | BIGINT | PK | NOT NULL | 主キー |
| stock_in_id | 入庫ID | BIGINT | FK → stock_in | NOT NULL | |
| purchase_order_detail_id | 仕入れ明細ID | BIGINT | FK → purchase_order_detail | NOT NULL | どの発注内訳に対する入庫かを示す |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | 在庫算出時に商品別で集計するために保持 |
| received_quantity | 入庫数量 | INTEGER | | NOT NULL | |

### 7.9 sales_order（受注）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| sales_order_id | 受注ID | BIGINT | PK | NOT NULL | 主キー |
| member_id | 会員ID | BIGINT | FK → member | NOT NULL | 販売に先立ち会員登録が必須 |
| ordered_on | 受注日 | DATE | | NOT NULL | |
| is_cancelled | 取消フラグ | BOOLEAN | | NOT NULL | 数量集計だけでは表現できない取消の状態を個別に保持する |

進行状況（未出荷／一部出荷／出荷完了）はカラムに持たせず、出庫（stock_out）の実績集計から動的に
導出する。

### 7.10 sales_order_detail（受注明細）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| sales_order_detail_id | 受注明細ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_id | 受注ID | BIGINT | FK → sales_order | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| ordered_quantity | 受注数量 | INTEGER | | NOT NULL | |
| selling_unit_price | 販売単価 | DECIMAL(10,2) | | NOT NULL | 受注時点の単価のスナップショット |

### 7.11 shipment（出荷）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| shipment_id | 出荷ID | BIGINT | PK | NOT NULL | 主キー |
| member_id | 会員ID | BIGINT | FK → member | NOT NULL | まとめ出荷時に複数受注の商品を1回の出荷に集約するため、受注を介さず配送先会員を直接参照する |
| shipped_on | 出荷日 | DATE | | NOT NULL | |

### 7.12 stock_out（出庫）─ トランザクション（中間テーブル）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_out_id | 出庫ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_detail_id | 受注明細ID | BIGINT | FK → sales_order_detail | NOT NULL | 分割出荷・まとめ出荷を表現する中間テーブル。両方の親が揃わないと作成できない |
| shipment_id | 出荷ID | BIGINT | FK → shipment | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | 在庫算出時に商品別で集計するために保持 |
| shipped_quantity | 出庫数量 | INTEGER | | NOT NULL | |

### 7.13 stock_taking（棚卸し）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_taking_id | 棚卸しID | BIGINT | PK | NOT NULL | 主キー |
| target_month | 対象年月 | VARCHAR(7) | | NOT NULL | 例: 2026-11。月初に1回実施するため一意 |
| conducted_on | 実施日 | DATE | | NOT NULL | 通常は対象月の月初日 |

### 7.14 stock_taking_detail（棚卸し明細）─ トランザクション

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_taking_detail_id | 棚卸し明細ID | BIGINT | PK | NOT NULL | 主キー |
| stock_taking_id | 棚卸しID | BIGINT | FK → stock_taking | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| counted_quantity | 実棚卸数量 | INTEGER | | NOT NULL | 棚卸しによって確認された実在庫数 |

## 8. 在庫算出ロジック

CLAUDE.mdのドメイン依存ルール「在庫数は常時テーブルに保持せず、動的に算出する」に従い、在庫数を
保持する専用テーブルは設けない。棚卸し（stock_taking）は他のどのトランザクションとも直接関連せず、
商品マスタのみを介して間接的に在庫算出ロジックと結びつく独立した系列として扱う。

任意時点 `T` における商品 `P` の在庫数は、以下の式で算出する。

```
在庫数(P, T) = 直近の棚卸し明細の実棚卸数量(counted_quantity)
             + Σ 入庫明細の入庫数量(received_quantity)   （直近棚卸しの実施日より後 〜 T）
             − Σ 出庫の出庫数量(shipped_quantity)          （直近棚卸しの実施日より後 〜 T）
```

月初の棚卸しごとに基準点が更新されるため、日々の入出庫明細の蓄積によって生じる誤差が毎月リセット
される。

## 9. ER図

本データモデルを表すER図は、CLAUDE.mdのレイアウトルール（A4横比率、直角線のみ、鳥の足記法、
テーブル分類別の色分け、日本語論理名表示、発生順の横配置、ヘッダ・明細の縦配置）に従い、以下の
5枚に分割して作成した（Graphvizのrankおよびortho splinesにより配置・線形状を明示制御している。
ソースは同名の `.dot` ファイル）。

| # | ファイル | 内容 | 列情報の粒度 |
|---|---|---|---|
| 1 | [er-diagram-01-master-state.png](er-diagram-01-master-state.png) | マスター・ステート一覧図 | PK/FK＋業務属性 |
| 2 | [er-diagram-02-purchase.png](er-diagram-02-purchase.png) | 仕入れ系列図（仕入れ→入庫） | PK/FK＋業務属性 |
| 3 | [er-diagram-03-sales.png](er-diagram-03-sales.png) | 受注・出荷系列図（受注→出庫←出荷） | PK/FK＋業務属性 |
| 4 | [er-diagram-04-stock-taking.png](er-diagram-04-stock-taking.png) | 棚卸し系列図 | PK/FK＋業務属性 |
| 5 | [er-diagram-05-overview.png](er-diagram-05-overview.png) | トランザクション関連の統合図（系列横断） | PKのみ |

図2・図3・図4・図5では、複数のトランザクションから参照される商品（product）マスタおよび会員
（member）ステートを、参照元のトランザクションの近くにそれぞれ複製して配置している（CLAUDE.mdの
「複数から参照されるマスタ・ステートは複製配置するべきである」ルールに基づく）。図5では商品を
5箇所（仕入れ明細・入庫明細・受注明細・出庫・棚卸し明細）、会員を2箇所（受注・出荷）に複製した
結果、3系列（仕入れ系列／受注・出荷系列／棚卸し系列）がそれぞれ独立した横方向の列として並ぶ構成に
なっている。なお、カテゴリ（category）は商品マスタ経由でしか繋がらずどのトランザクションにも
直接関連しないため、「トランザクションに直接関連しないエンティティは描画を省略してよい」ルールに
基づき図5への描画を省略している。図5は系列を横断して全体を一覧するための統合図であり、列情報を
PKのみに絞ることで図の目的（全体の関連構造の把握）に合わせた情報量としている。

## 10. 本書のスコープ外

要件定義書5章に明示されている通り、以下は本モデリングの対象外とする。

- 会員のログイン認証情報の管理方法
- 決済・配送業者との連携情報
- 返品・キャンセルに伴う入出庫（逆伝票）の扱い
- 商品の書籍／ガジェットそれぞれに固有な属性（ISBN、著者、保証期間等）の詳細管理
- 複数倉庫（拠点）運用時の在庫管理
- 在庫金額の計算方法（先入先出法・総平均法等の単価決定方式）
