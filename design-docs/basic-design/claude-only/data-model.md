# データモデリング設計書（Claude単独モデリング版）

## 1. 本書の位置づけ

本書は、与えられた仕様を元に Claude が単独で行ったデータモデリングの結果をまとめた基本設計書である。
この後のステップで、別途参考資料（Qiita記事）を元にしたモデリングを行い、両者を比較する予定であるため、
本書はあくまで「参考資料を見ていない状態での一次モデリング結果」として扱う。

## 2. 対象システムの概要

オンラインで技術書・技術関連アイテム（USBメモリ、モバイルバッテリーなどの技術ガジェットを中心とする）を
販売するECシステムを対象とする。管理対象は以下の通り。

- 商品、および商品が属するカテゴリ
- 販売履行（販売履歴）、および販売に先立って必要となる利用者（会員）登録
- 商品の仕入れ、仕入れ先
- 仕入れた商品の入庫、売れた商品の出庫（出荷）
- 月初に実施する棚卸しと、その結果の記録
- 在庫数は、棚卸し記録を基準に、当月発生分の入庫・出庫数量で調整した値として求める（在庫テーブル自体は持たない導出値とする）

## 3. 設計方針・前提

### 3.1 「仕入れ／出荷」と「入庫／出庫」の関係

仕様文中の「仕入れ」「出荷」という語と、「入庫」「出庫」という語は、以下のように整理して解釈する。

- **仕入れ（発注）**: 仕入れ先に対して商品を発注する行為・記録。契約・発注段階の情報。
- **入庫**: 仕入れた商品が実際に倉庫に到着し、在庫として積み上げられる実績記録。1件の仕入れに対して、
  納期のズレ等により複数回に分けて入庫される可能性があるため、仕入れと入庫は 1対多 の関係とする。
- **販売（受注）**: 会員が商品を購買する行為・記録。
- **出荷／出庫**: 販売された商品が実際に倉庫から出ていく実績記録。仕様文中の「出荷」は「出庫」と同義として扱い、
  本モデルでは「出庫」という用語に統一する。1件の販売に対して、商品ごとに出庫タイミングが分かれる可能性があるため、
  販売と出庫は 1対多 の関係とする。

### 3.2 在庫の扱い

在庫数は常時保持する「在庫テーブル」を持たず、以下の算出ルールによる **導出値** として扱う。

> ある時点の在庫数 ＝ その時点以前で直近に確定した棚卸しの実棚卸数量
> ＋（棚卸し実施以降、その時点までに発生した入庫数量の合計）
> －（棚卸し実施以降、その時点までに発生した出庫数量の合計）

棚卸しは月初に実施し、商品ごとの実棚卸数量を棚卸明細として記録する。これにより、月ごとに基準点（棚卸し）が
更新され、計算誤差の蓄積を一定周期でリセットできる。

### 3.3 価格のスナップショット

商品の販売単価・仕入単価はマスタ上で変動しうるため、販売明細・仕入明細にはそれぞれ受注／発注時点の単価を
スナップショットとして保持する（商品マスタの価格変更が過去の取引記録に影響しないようにするため）。

### 3.4 本書のスコープ

本書はデータモデル（エンティティ・属性・関連）の設計に限定する。ER図は別途、後続のステップで作成する。

## 4. エンティティ一覧

| # | エンティティ（論理名） | 物理名 | 概要 |
|---|---|---|---|
| 1 | 商品カテゴリ | category | 商品を分類するカテゴリ |
| 2 | 商品 | product | 販売対象となる商品（技術書・技術ガジェット等） |
| 3 | 会員 | member | 購買のために登録が必要な利用者 |
| 4 | 仕入れ先 | supplier | 商品の仕入れ元となる取引先 |
| 5 | 仕入れ | purchase_order | 仕入れ先への発注記録（ヘッダ） |
| 6 | 仕入れ明細 | purchase_order_detail | 仕入れの商品別内訳 |
| 7 | 入庫 | stock_in | 仕入れた商品が倉庫に入った実績記録（ヘッダ） |
| 8 | 入庫明細 | stock_in_detail | 入庫の商品別内訳 |
| 9 | 販売 | sales_order | 会員による購買記録（ヘッダ） |
| 10 | 販売明細 | sales_order_detail | 販売の商品別内訳 |
| 11 | 出庫 | stock_out | 販売された商品が倉庫から出た実績記録（ヘッダ） |
| 12 | 出庫明細 | stock_out_detail | 出庫の商品別内訳 |
| 13 | 棚卸し | stock_taking | 月初に実施する棚卸しの記録（ヘッダ） |
| 14 | 棚卸し明細 | stock_taking_detail | 棚卸しの商品別の実棚卸数量 |

## 5. エンティティ間の関連（概要）

```
category        1 ─── N  product
product         1 ─── N  purchase_order_detail
product         1 ─── N  sales_order_detail
product         1 ─── N  stock_in_detail
product         1 ─── N  stock_out_detail
product         1 ─── N  stock_taking_detail

supplier        1 ─── N  purchase_order
purchase_order  1 ─── N  purchase_order_detail
purchase_order  1 ─── N  stock_in
stock_in        1 ─── N  stock_in_detail
stock_in_detail N ─── 1  purchase_order_detail   (どの仕入明細に対する入庫かを示す)

member          1 ─── N  sales_order
sales_order     1 ─── N  sales_order_detail
sales_order     1 ─── N  stock_out
stock_out       1 ─── N  stock_out_detail
stock_out_detail N ─── 1  sales_order_detail      (どの販売明細に対する出庫かを示す)

stock_taking    1 ─── N  stock_taking_detail
stock_taking_detail N ─── 1 product
```

## 6. テーブル定義

### 6.1 category（商品カテゴリ）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| category_id | カテゴリID | BIGINT | PK | NOT NULL | 主キー |
| category_name | カテゴリ名 | VARCHAR(100) | | NOT NULL | 例: 技術書／USBメモリ／モバイルバッテリー |
| category_description | カテゴリ説明 | VARCHAR(500) | | NULL | 補足説明 |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.2 product（商品）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| product_id | 商品ID | BIGINT | PK | NOT NULL | 主キー |
| category_id | カテゴリID | BIGINT | FK → category | NOT NULL | 所属カテゴリ |
| product_code | 商品コード | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| product_name | 商品名 | VARCHAR(200) | | NOT NULL | |
| product_type | 商品区分 | VARCHAR(20) | | NOT NULL | 例: BOOK（技術書）／GADGET（技術ガジェット） |
| standard_unit_price | 標準販売単価 | DECIMAL(10,2) | | NOT NULL | マスタ上の参考単価（実販売単価は販売明細にスナップショット） |
| standard_cost_price | 標準仕入単価 | DECIMAL(10,2) | | NOT NULL | マスタ上の参考単価（実仕入単価は仕入明細にスナップショット） |
| description | 商品説明 | VARCHAR(1000) | | NULL | |
| is_active | 販売可否フラグ | BOOLEAN | | NOT NULL | 販売中止商品の管理用 |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.3 member（会員）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| member_id | 会員ID | BIGINT | PK | NOT NULL | 主キー |
| member_code | 会員コード | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| member_name | 会員名 | VARCHAR(100) | | NOT NULL | |
| email | メールアドレス | VARCHAR(255) | | NOT NULL | 一意（UNIQUE制約）、ログインIDとしても利用 |
| phone_number | 電話番号 | VARCHAR(20) | | NULL | |
| address | 住所 | VARCHAR(300) | | NULL | 配送先 |
| registered_at | 登録日時 | TIMESTAMP | | NOT NULL | 会員登録完了日時 |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.4 supplier（仕入れ先）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| supplier_id | 仕入れ先ID | BIGINT | PK | NOT NULL | 主キー |
| supplier_code | 仕入れ先コード | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| supplier_name | 仕入れ先名 | VARCHAR(200) | | NOT NULL | |
| contact_person | 担当者名 | VARCHAR(100) | | NULL | |
| phone_number | 電話番号 | VARCHAR(20) | | NULL | |
| address | 住所 | VARCHAR(300) | | NULL | |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.5 purchase_order（仕入れ）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| purchase_order_id | 仕入れID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_number | 仕入れ番号 | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約）、業務上の管理番号 |
| supplier_id | 仕入れ先ID | BIGINT | FK → supplier | NOT NULL | 発注先 |
| order_date | 発注日 | DATE | | NOT NULL | |
| status | ステータス | VARCHAR(20) | | NOT NULL | 例: 発注済／入庫中／入庫完了／キャンセル |
| total_amount | 合計金額 | DECIMAL(12,2) | | NOT NULL | 明細合計（冗長保持） |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.6 purchase_order_detail（仕入れ明細）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| purchase_order_detail_id | 仕入れ明細ID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_id | 仕入れID | BIGINT | FK → purchase_order | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| order_quantity | 発注数量 | INTEGER | | NOT NULL | |
| unit_price | 仕入単価 | DECIMAL(10,2) | | NOT NULL | 発注時点の単価（スナップショット） |
| subtotal_amount | 明細金額 | DECIMAL(12,2) | | NOT NULL | order_quantity × unit_price |

### 6.7 stock_in（入庫）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_in_id | 入庫ID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_id | 仕入れID | BIGINT | FK → purchase_order | NOT NULL | 対象の仕入れ（分納により1仕入れに対し複数の入庫が発生しうる） |
| stock_in_date | 入庫日 | DATE | | NOT NULL | |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.8 stock_in_detail（入庫明細）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_in_detail_id | 入庫明細ID | BIGINT | PK | NOT NULL | 主キー |
| stock_in_id | 入庫ID | BIGINT | FK → stock_in | NOT NULL | |
| purchase_order_detail_id | 仕入れ明細ID | BIGINT | FK → purchase_order_detail | NOT NULL | どの仕入れ明細に対する入庫かを示す |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | 検索・集計の簡便化のための冗長保持（purchase_order_detail経由でも導出可） |
| stock_in_quantity | 入庫数量 | INTEGER | | NOT NULL | |

### 6.9 sales_order（販売）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| sales_order_id | 販売ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_number | 販売番号 | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約）、業務上の管理番号 |
| member_id | 会員ID | BIGINT | FK → member | NOT NULL | 購買した会員（販売に先立ち会員登録が必須） |
| order_date | 受注日 | DATE | | NOT NULL | |
| status | ステータス | VARCHAR(20) | | NOT NULL | 例: 受注済／出庫中／出庫完了／キャンセル |
| total_amount | 合計金額 | DECIMAL(12,2) | | NOT NULL | 明細合計（冗長保持） |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.10 sales_order_detail（販売明細）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| sales_order_detail_id | 販売明細ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_id | 販売ID | BIGINT | FK → sales_order | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| order_quantity | 受注数量 | INTEGER | | NOT NULL | |
| unit_price | 販売単価 | DECIMAL(10,2) | | NOT NULL | 受注時点の単価（スナップショット） |
| subtotal_amount | 明細金額 | DECIMAL(12,2) | | NOT NULL | order_quantity × unit_price |

### 6.11 stock_out（出庫）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_out_id | 出庫ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_id | 販売ID | BIGINT | FK → sales_order | NOT NULL | 対象の販売（商品ごとの出庫タイミング差により1販売に対し複数の出庫が発生しうる） |
| stock_out_date | 出庫日 | DATE | | NOT NULL | |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.12 stock_out_detail（出庫明細）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_out_detail_id | 出庫明細ID | BIGINT | PK | NOT NULL | 主キー |
| stock_out_id | 出庫ID | BIGINT | FK → stock_out | NOT NULL | |
| sales_order_detail_id | 販売明細ID | BIGINT | FK → sales_order_detail | NOT NULL | どの販売明細に対する出庫かを示す |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | 検索・集計の簡便化のための冗長保持（sales_order_detail経由でも導出可） |
| stock_out_quantity | 出庫数量 | INTEGER | | NOT NULL | |

### 6.13 stock_taking（棚卸し）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_taking_id | 棚卸しID | BIGINT | PK | NOT NULL | 主キー |
| target_year_month | 対象年月 | VARCHAR(7) | | NOT NULL | 例: 2026-11。一意（UNIQUE制約、月初に1回実施） |
| stock_taking_date | 棚卸し実施日 | DATE | | NOT NULL | 通常は対象月の月初日 |
| status | ステータス | VARCHAR(20) | | NOT NULL | 例: 実施中／確定済み |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.14 stock_taking_detail（棚卸し明細）

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_taking_detail_id | 棚卸し明細ID | BIGINT | PK | NOT NULL | 主キー |
| stock_taking_id | 棚卸しID | BIGINT | FK → stock_taking | NOT NULL | |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | |
| counted_quantity | 実棚卸数量 | INTEGER | | NOT NULL | 棚卸しによって確認された実在庫数 |

（stock_taking_id と product_id の組に一意制約を付与し、同一棚卸し内で同一商品の明細が重複しないようにする）

## 7. 在庫算出ロジック（補足）

在庫テーブルを持たない設計のため、任意時点 `T`（対象商品 `P`）の在庫数は以下の手順で算出する。

1. `T` 以前に確定（status=確定済み）した棚卸しのうち、最新のもの（`target_year_month` が最大のもの）を取得する。
   これを基準棚卸しとし、その `stock_taking_detail.counted_quantity` を基準在庫数とする。
2. 基準棚卸しの実施日 (`stock_taking_date`) より後、かつ `T` 以前に発生した、商品 `P` に関する
   `stock_in_detail.stock_in_quantity` の合計を加算する。
3. 同期間に発生した商品 `P` に関する `stock_out_detail.stock_out_quantity` の合計を減算する。

```
在庫数(P, T) = 基準棚卸しの counted_quantity
             + Σ stock_in_quantity（基準棚卸し実施日より後 〜 T）
             − Σ stock_out_quantity（基準棚卸し実施日より後 〜 T）
```

この方式により、月初の棚卸しごとに基準点が更新され、日々の入出庫明細の蓄積によって生じる誤差が
毎月リセットされる。

## 8. 今後の検討事項（本モデリングのスコープ外）

- 会員のログイン認証情報（パスワード等）の管理方法
- 決済・配送業者との連携情報
- 返品・キャンセルに伴う入出庫（逆伝票）の扱い
- 商品の書籍／ガジェットそれぞれに固有な属性（ISBN、著者、保証期間等）の詳細管理
- 複数倉庫（拠点）運用時の在庫管理（本モデルは単一倉庫を前提）

上記は仕様に明記がないため、本モデリングでは対象外としている。
