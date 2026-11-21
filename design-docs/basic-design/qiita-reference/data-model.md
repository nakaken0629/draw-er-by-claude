# データモデリング設計書（Qiita記事参考版）

## 1. 本書の位置づけ

本書は、[claude-only/data-model.md](../claude-only/data-model.md) で作成した「Claude単独モデリング」に対し、
以下のQiita記事（著者本人の過去の記事）で示されている設計思想・原則を踏まえて再モデリングを行った結果である。

- 記事1: https://qiita.com/nakaken0629/items/02633743068f217a0429 （テーブルの分類と関連の作り方）
- 記事2: https://qiita.com/nakaken0629/items/9bd9b519d66fd13f23e0 （ER図のレイアウト原則。本書ではなく後続のER図作成ステップで適用）
- 記事3: https://qiita.com/nakaken0629/items/1c67b983b200e19df55b （受注・出荷の多重関連と在庫金額計算の実務的課題）

対象システムの概要・前提は claude-only 版と同一のため、本書では差分となる設計方針を中心に記述する。

## 2. 参考記事から得た知見

### 2.1 記事1：テーブルの分類と関連構築の優先順位

- テーブルは「レコードを誰が追加するか」で **マスターテーブル**（管理者が登録）と
  **ユーザー主導テーブル** に分け、ユーザー主導テーブルはさらに
  **トランザクションテーブル**（追加後は基本的に更新されない）と
  **ステートテーブル**（ユーザー操作により何度も更新される）に分ける。
- 関連を構築する際は、正規化上の必要性だけでなく、**トランザクション同士を業務プロセスの発生順に
  直接関連づけることを優先する**（例：購入 → 入金 → 出荷）。これにより「どの購入に対応する入金か」が
  一意に追跡できる設計になる。

### 2.2 記事3：受注と出荷の多重関連、在庫金額計算

- 「受注1 : 出荷N」という単純なモデルは、実運用では **在庫不足による分割出荷** や
  **複数受注のまとめ出荷** が発生するため、実際には **受注と出荷は多対多の関係** になる。
- この多対多を解消するため、中間テーブル「**出庫**」を導入する。出庫は受注（明細）と出荷の両方への
  参照を持ち、「出庫は受注と出荷の両方が存在しないと作成できない」という制約を持たせることで、
  現場オペレーションの実態（分割・まとめ出荷）に適合させる。
- 在庫金額を計算する場合、先入先出法・総平均法・移動平均法・最終仕入原価法などの単価決定方法により
  金額が変動する点に注意が必要（本システムの仕様は数量ベースの在庫算出のみを要求しているため、
  本書では金額計算ロジックの詳細化は行わないが、将来拡張時の留意点として記録する）。

## 3. 設計方針（claude-only 版からの変更点）

### 3.1 テーブル分類の明示

| 分類 | テーブル |
|---|---|
| マスターテーブル | category, product, supplier |
| ステートテーブル | member（会員自身が住所・連絡先等を更新できるため） |
| トランザクションテーブル（仕入れ系列） | purchase_order, purchase_order_detail, stock_in, stock_in_detail |
| トランザクションテーブル（販売系列） | sales_order, sales_order_detail, shipment, stock_out |
| トランザクションテーブル（棚卸し系列） | stock_taking, stock_taking_detail |

### 3.2 販売系列：受注・出荷・出庫の三層構造への変更

claude-only 版では「販売 1 : N 出庫」という単純な親子関係としていたが、記事3の知見を踏まえ、
以下のように変更する。

- **sales_order（受注）**: 会員からの受注トランザクション。従来通り。
- **shipment（出荷）**: 配送単位のトランザクションとして新設。同一会員の複数受注をまとめて1回で
  配送する場合や、1受注を複数回に分けて配送する場合（分割出荷）を表現できるようにする。
- **stock_out（出庫）**: 「sales_order_detail（受注明細）」と「shipment（出荷）」の **両方を参照する
  中間テーブル** とする。1件の出庫は必ず1つの受注明細と1つの出荷に紐づき、かつ在庫を減少させる
  実際のトランザクションとして機能する。これにより、受注明細と出荷は出庫を介して多対多の関係になる。

この変更により、claude-only 版で sales_order と stock_out（出庫）の間に直接存在した 1:N の関連は廃止し、
代わりに sales_order_detail（受注明細）と stock_out、shipment と stock_out の関連に置き換える。

### 3.3 仕入れ系列は変更なし（非対称な設計の採用）

記事3の知見を仕入れ側にも対称的に適用すれば、「仕入れ（発注）」と「入荷（配送ロット）」を分け、
「入庫」を両者の中間テーブルにすることも考えられる。しかし、仕様文中では「出荷」という語は
販売側にのみ現れ、仕入れ側については「仕入れ」「入庫」の二層構造のみが言及されている。
また、仕入れ先からの納品は仕入れ先側の都合で分納されることはあっても、複数の仕入れをまとめて
1回で受け取るといった業務上の要求は仕様に記載がない。そのため、仕入れ側は claude-only 版と同様に
「purchase_order 1 : N stock_in」という単純な親子トランザクション構造を維持する（オーバーエンジニア
リングを避ける）。

### 3.4 ステータス管理の考え方の変更

claude-only 版では purchase_order・sales_order に `status`（進行状況）カラムを持たせていたが、
記事1の「トランザクションは追加後は基本的に更新されない」という原則を踏まえ、進行状況
（未着手／一部入庫・一部出庫／完了）は **関連するトランザクション（入庫・出庫）の集計から動的に
導出する** 方針に変更する。これにより、トランザクションテーブル自体は追加後に更新されない
immutable な記録として扱える。

ただし、発注・受注の取消しは数量の集計だけでは表現できない業務イベントであるため、
`is_cancelled`（取消フラグ）のみは状態更新用の属性として残す。

## 4. エンティティ一覧

| # | エンティティ（論理名） | 物理名 | 分類 | 概要 |
|---|---|---|---|---|
| 1 | 商品カテゴリ | category | マスター | 商品を分類するカテゴリ |
| 2 | 商品 | product | マスター | 販売対象となる商品 |
| 3 | 仕入れ先 | supplier | マスター | 商品の仕入れ元となる取引先 |
| 4 | 会員 | member | ステート | 購買のために登録が必要な利用者 |
| 5 | 仕入れ | purchase_order | トランザクション | 仕入れ先への発注記録（ヘッダ） |
| 6 | 仕入れ明細 | purchase_order_detail | トランザクション | 仕入れの商品別内訳 |
| 7 | 入庫 | stock_in | トランザクション | 仕入れに対する入庫実績（ヘッダ） |
| 8 | 入庫明細 | stock_in_detail | トランザクション | 入庫の商品別内訳 |
| 9 | 受注 | sales_order | トランザクション | 会員による受注記録（ヘッダ） |
| 10 | 受注明細 | sales_order_detail | トランザクション | 受注の商品別内訳 |
| 11 | 出荷 | shipment | トランザクション | 配送単位のトランザクション（ヘッダ） |
| 12 | 出庫 | stock_out | トランザクション | 受注明細と出荷を紐づける中間テーブル（在庫減少の実績） |
| 13 | 棚卸し | stock_taking | トランザクション | 月初に実施する棚卸しの記録（ヘッダ） |
| 14 | 棚卸し明細 | stock_taking_detail | トランザクション | 棚卸しの商品別の実棚卸数量 |

## 5. エンティティ間の関連

```
category        1 ─── N  product
product         1 ─── N  purchase_order_detail
product         1 ─── N  sales_order_detail
product         1 ─── N  stock_in_detail
product         1 ─── N  stock_out        (検索性のための冗長FK)
product         1 ─── N  stock_taking_detail

[仕入れ系列：トランザクション同士の直接の親子関連（変更なし）]
supplier        1 ─── N  purchase_order
purchase_order  1 ─── N  purchase_order_detail
purchase_order  1 ─── N  stock_in
stock_in        1 ─── N  stock_in_detail
stock_in_detail N ─── 1  purchase_order_detail

[販売系列：受注・出荷・出庫の三層構造（claude-only版から変更）]
member              1 ─── N  sales_order
member              1 ─── N  shipment              (出荷先の会員。複数受注をまとめて出荷できる)
sales_order         1 ─── N  sales_order_detail
sales_order_detail  1 ─── N  stock_out             (受注明細の数量を複数回に分けて出庫できる＝分割出荷)
shipment            1 ─── N  stock_out             (1回の出荷に複数の受注明細の出庫をまとめられる)
   ※ sales_order_detail と shipment は stock_out を介して多対多の関係

[棚卸し系列：変更なし]
stock_taking         1 ─── N  stock_taking_detail
stock_taking_detail  N ─── 1  product
```

## 6. テーブル定義

claude-only 版（[data-model.md](../claude-only/data-model.md) 6章）と共通のテーブルは属性変更点のみ記載し、
新設・再設計したテーブルは全属性を記載する。

### 6.1 category / product / supplier（マスターテーブル）

claude-only 版と同一構成。差分なし。

### 6.2 member（会員・ステートテーブル）

claude-only 版と同一構成。差分なし（分類を「ステートテーブル」と明示）。

### 6.3 purchase_order（仕入れ）【変更】

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| purchase_order_id | 仕入れID | BIGINT | PK | NOT NULL | 主キー |
| purchase_order_number | 仕入れ番号 | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| supplier_id | 仕入れ先ID | BIGINT | FK → supplier | NOT NULL | 発注先 |
| order_date | 発注日 | DATE | | NOT NULL | |
| is_cancelled | 取消フラグ | BOOLEAN | | NOT NULL | 進行状況（未着手／一部入庫／入庫完了）は stock_in_detail の集計から導出するため status カラムは廃止 |
| total_amount | 合計金額 | DECIMAL(12,2) | | NOT NULL | 明細合計（冗長保持） |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | 取消フラグ更新時のみ更新される |

purchase_order_detail / stock_in / stock_in_detail は claude-only 版と同一構成（差分なし）。

### 6.4 sales_order（受注）【変更】

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| sales_order_id | 受注ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_number | 受注番号 | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| member_id | 会員ID | BIGINT | FK → member | NOT NULL | 受注した会員 |
| order_date | 受注日 | DATE | | NOT NULL | |
| is_cancelled | 取消フラグ | BOOLEAN | | NOT NULL | 進行状況（未出庫／一部出庫／出庫完了）は stock_out の集計から導出するため status カラムは廃止 |
| total_amount | 合計金額 | DECIMAL(12,2) | | NOT NULL | 明細合計（冗長保持） |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | 取消フラグ更新時のみ更新される |

### 6.5 sales_order_detail（受注明細）

claude-only 版の sales_order_detail と同一構成（差分なし）。

### 6.6 shipment（出荷）【新設】

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| shipment_id | 出荷ID | BIGINT | PK | NOT NULL | 主キー |
| shipment_number | 出荷番号 | VARCHAR(50) | | NOT NULL | 一意（UNIQUE制約） |
| member_id | 会員ID | BIGINT | FK → member | NOT NULL | 配送先の会員（同一会員の複数受注をまとめて出荷できる） |
| shipping_address | 配送先住所 | VARCHAR(300) | | NOT NULL | 出荷時点の配送先住所（スナップショット） |
| shipment_date | 出荷日 | DATE | | NOT NULL | |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |
| updated_at | 更新日時 | TIMESTAMP | | NOT NULL | |

### 6.7 stock_out（出庫）【新設・中間テーブル】

| 列名 | 論理名 | 型 | PK/FK | NULL | 説明 |
|---|---|---|---|---|---|
| stock_out_id | 出庫ID | BIGINT | PK | NOT NULL | 主キー |
| sales_order_detail_id | 受注明細ID | BIGINT | FK → sales_order_detail | NOT NULL | どの受注明細に対する出庫かを示す |
| shipment_id | 出荷ID | BIGINT | FK → shipment | NOT NULL | どの出荷に含まれるかを示す（両方のFKが必須＝受注と出荷の両方が存在しないと出庫は作れない） |
| product_id | 商品ID | BIGINT | FK → product | NOT NULL | 検索・集計の簡便化のための冗長保持（sales_order_detail経由でも導出可） |
| stock_out_quantity | 出庫数量 | INTEGER | | NOT NULL | 受注明細の受注数量を上限として、分割して複数回登録されうる |
| created_at | 作成日時 | TIMESTAMP | | NOT NULL | |

### 6.8 stock_taking / stock_taking_detail（棚卸し）

claude-only 版と同一構成（差分なし）。

## 7. 在庫算出ロジック

基本的な算出式は claude-only 版と同一だが、販売側の出庫参照先が `stock_out_detail` から
`stock_out`（中間テーブル自体が明細を兼ねる）に変わる。

```
在庫数(P, T) = 基準棚卸しの counted_quantity
             + Σ stock_in_quantity（基準棚卸し実施日より後 〜 T、stock_in_detail より集計）
             − Σ stock_out_quantity（基準棚卸し実施日より後 〜 T、stock_out より集計。
                                     product_id は stock_out.product_id で直接絞り込み）
```

## 8. claude-only 版との主な差分（要約）

| 観点 | claude-only 版 | qiita-reference 版 |
|---|---|---|
| テーブル分類の明示 | なし | マスター／ステート／トランザクションを明示 |
| 販売側の構造 | 販売（受注）→出庫 の1:N | 受注→受注明細、出荷、出庫（中間テーブル）の三層。受注明細と出荷は出庫を介して多対多 |
| 分割出荷・まとめ出荷 | 表現できない（出庫は受注1件に従属） | 出庫が受注明細×出荷の組を個別に表現するため対応可能 |
| 仕入れ側の構造 | 仕入れ→入庫 の1:N | 変更なし（対称化は見送り、仕様の記述に忠実に非対称構造を採用） |
| ステータス管理 | status カラムで状態を保持 | is_cancelled のみ保持し、進行状況は関連トランザクションの集計から導出 |

詳細な比較・考察は別途、比較ドキュメントにまとめる。

## 9. 今後の検討事項（本モデリングのスコープ外）

claude-only 版の「8. 今後の検討事項」と同様。加えて以下を追加する。

- 在庫金額の計算が必要になった場合の単価決定方法（先入先出法／総平均法／移動平均法／最終仕入原価法）の選定
- 仕入れ側に分納・まとめ入荷の多重関連が将来的に必要になった場合の、入荷（配送ロット）・入庫の三層構造化

## 10. ER図

本モデルのER図を、以下の2段階で作成した。

### 10.1 全体図（Claude独自レイアウト）

[er-diagram.png](er-diagram.png)（元データ: [er-diagram.mmd](er-diagram.mmd), Mermaid記法）

全14エンティティを1枚にまとめた全体像。配置はMermaidの自動レイアウトエンジンに任せており、
記事2（ER図のレイアウト原則）はまだ適用していない。全体の関連構造を俯瞰する用途に用いる。

### 10.2 記事2のレイアウト原則を適用した分割図

記事2 https://qiita.com/nakaken0629/items/9bd9b519d66fd13f23e0 で示されている以下の原則を適用し、
Graphviz（dot記法）で4枚に分割して再作成した。

- A4横を意識してデザインする（出力画像のアスペクト比を297:210に統一。エンティティのボックス自体のサイズは変えず、余白を追加することで比率を合わせている）
- 注目するモデルで分割する（マスター・ステートの図とトランザクションの図を分ける）
- 親子関係のモデルは縦に並べる
- トランザクションは発生順に横に並べる
- 多重度はER図の方式（鳥の足記法）で表記する（直線＋短い直角線＝1、鳥の足＝多。数字のラベルではなく記号で表現）
- エンティティ間を結ぶ線は斜め線を禁止し、直角に曲がる線（鍵型）のみを用いる

色分けは、テーブルの分類（2.1節）に対応させ、マスターテーブルは緑、ステートテーブルは黄色、
トランザクションテーブル（出庫のような中間テーブルを含む）は赤で統一した。

| # | 図 | ファイル | 内容 |
|---|---|---|---|
| 1 | マスター・ステート一覧図 | [er-diagram-layout-01-master-state.png](er-diagram-layout-01-master-state.png) | category, product, supplier（マスター、緑）と member（ステート、黄） |
| 2 | 仕入れ系列トランザクション図 | [er-diagram-layout-02-purchase.png](er-diagram-layout-02-purchase.png) | 仕入れ→入庫を横方向（発生順）、各ヘッダ→明細を縦方向に配置（トランザクションは赤） |
| 3 | 販売系列トランザクション図 | [er-diagram-layout-03-sales.png](er-diagram-layout-03-sales.png) | 受注→受注明細（縦）、受注明細と出荷が出庫（中間テーブル）を介して多対多になる様子を図示 |
| 4 | 棚卸し系列トランザクション図 | [er-diagram-layout-04-stock-taking.png](er-diagram-layout-04-stock-taking.png) | 棚卸し→棚卸し明細を縦方向に配置 |

全体図（10.1）では14エンティティの関連線が交錯し、特に販売系列（受注・受注明細・出荷・出庫）の
多対多構造が読み取りにくかったのに対し、分割図（10.2）では各トランザクション系列の発生順序と
親子構造が一目で追えるようになっている。特に図3では、記事3の知見である「受注明細と出荷が出庫を
介して多対多になる」構造を、2本の矢印（「分割して出庫されうる」「まとめて出庫されうる」）が
1つの出庫テーブルに収束する形で視覚的に表現できた。

### 10.3 全トランザクションの関連に注目した統合図

10.2の4枚は系列ごとに分割しているため、仕入れ系列・販売系列・棚卸し系列を横断して全体の
トランザクションの繋がりを一望することができない。そこで、10.2のレイアウト方式（A4横、直角線、
鳥の足記法、色分け）は維持しつつ、10個のトランザクションテーブル全てを1枚に統合した図を追加した。
マスター・ステートは関連先として登場させるが、主役はあくまでトランザクション同士の関連であるため、
以下のように描き分けている。

- トランザクション同士の関連: 黒の太い実線
- マスター・ステートへの参照: 灰色の細い点線（脇役として後景に退かせる）
- 各エンティティの列情報: PKのみ（必要最低限。FKや業務属性は10.2の分割図を参照）
- トランザクションに直接関連しないエンティティは描画を省略する。category は product を介して
  間接的にトランザクションへつながるだけで、トランザクションからの直接参照を持たないため省略した
  （category → product の関連自体は10.1のマスター・ステート一覧図に表示済み）
- 複数のトランザクションから参照される product・member は、1つのノードに集約せず、参照元の
  トランザクションの近くにそれぞれ複製して配置する（product は仕入れ・販売・棚卸しの3系列分、
  member は受注・出荷の2系列分を複製）。これにより1つのマスタに全方向から長い参照線が集中する
  ことを避け、系列ごとの関連を局所的に読み取れるようにしている
- 仕入れ系列・販売系列・棚卸し系列は、縦に積まず同じ列構成を共有させることで、全体を横に長く
  展開しつつ、A4横のアスペクト比に自然に収まるようにレイアウトしている

| # | 図 | ファイル | 内容 |
|---|---|---|---|
| 5 | トランザクション関連の全体図 | [er-diagram-layout-05-tx-overview.png](er-diagram-layout-05-tx-overview.png) | 仕入れ系列（仕入れ→入庫）・販売系列（受注→出荷／出庫）・棚卸し系列の3系統を1枚に統合。列はPKのみ、category等トランザクションに直接関連しないエンティティは省略 |

この図により、「仕入れ系列と販売系列は商品マスタを介してのみ間接的につながっており、トランザクション
同士が直接関連することはない」こと、また「棚卸し系列は他のどのトランザクションとも直接の関連を
持たず、商品マスタのみを介して間接的に在庫算出ロジック（9節）と結びつく」ことが、1枚の図から
一目で確認できる。詳細な列定義が必要な場合は10.2の系列別の図を、全体の関連構造を俯瞰したい場合は
本図を参照する、という使い分けを想定している。
