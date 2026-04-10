# 予算期間設定 仕様書 v1.4（最新master 2026-04-09 準拠 / 全画面影響分析済み）

## 概要

会社情報画面に「予算開始月」設定を追加し、予算関連の全画面で一貫した予算期間（年度）を適用する。

## 背景・課題

現状、予算の表示期間は「店舗作成日〜現在+3年」の全月リストから手動選択する形式で、**予算年度（会計年度）の概念がない**。月次予算設定は任意の期間を自由選択できるが、4月始まりや10月始まりなどの年度単位で管理する仕組みがない。

## データモデル設計

### 追加カラム（1つのみ）

```ruby
# Migration
add_column :companies, :budget_start_month, :integer, default: 1, comment: "予算開始月（1-12）"
```

### 現在の companies テーブル（最新master）

| カラム | 型 | 用途 |
|--------|---|------|
| id, name, representive, zipcode, prefecture_id, address, building, phone, email | - | 基本情報 |
| billing_zipcode, billing_prefecture_id, billing_address, billing_building | string/int | 請求先住所 |
| billing_staff_email, billing_staff_name, billing_staff_phone | string | 請求担当者 |
| trial_expired_on, extra_trial_expired_on | date | トライアル期限 |
| force_enterprise_enabled, is_infomart_api, is_vendor_billing_on | boolean | 機能フラグ |
| scalebase_customer_id, infomart_client_id, infomart_client_secret | string | 外部連携 |
| price_version | string | 価格体系 |
| **budget_start_month（新規追加）** | **integer** | **予算開始月（1-12）** |

### 関連テーブル（変更なし）

| テーブル | キーカラム | 予算期間との関係 |
|---------|---------|--------------|
| monthly_budgets | shop_id, year_month (YYYYMM int) | 年度範囲でフィルタ（デフォルト表示のみ変更） |
| monthly_cost_entries | shop_id, year_month (YYYYMM int) | 年度範囲でフィルタ（費用履歴の表示範囲変更） |
| monthly_costs | shop_id, value, ratio | 影響なし |
| monthly_cost_items | shop_id, name, immutable | 影響なし |
| shop_rents | shop_id, rent_type, effective_from/until | 影響なし |

### 地代家賃設定との関係（同期しない）

予算開始月と地代家賃の適用開始日（`effective_from`）は**独立した設定として管理し、同期させない**。

- **予算開始月**: 会社の管理方針で決まる予算年度の起点（例: 4月）
- **地代家賃の適用開始日**: 物件オーナーとの契約に基づく日付（例: 2026-01-01）。契約改訂は4月等の期初に行われることが多いが、店舗ごとの個体差が大きい
- **適用月モーダル**（デフォルト計算額の適用確認）: 地代家賃の適用開始日から算出された月リストを表示。予算開始月には連動しない
- **連動させない理由**: 予算開始月を変更するたびに地代家賃の適用月が変わると、実際の契約条件と乖離してしまうため

地代家賃の**計算結果**は年間予実対比・予実対比の費用行に表示されるが、これは `ShopRent#calculate(sales)` による月単位計算であり、予算開始月の影響は表示順序のみ（データ計算には影響なし）。

### 予算年度の計算

| 設定 | 2026年度の範囲 | year_month値 |
|------|-------------|-------------|
| 1月（デフォルト） | 2026年1月 〜 2026年12月 | 202601 〜 202612 |
| 4月 | 2026年4月 〜 2027年3月 | 202604 〜 202703 |
| 10月 | 2026年10月 〜 2027年9月 | 202610 〜 202709 |

## 全画面影響マップ

### 会社スコープ — 影響あり（5画面）

| # | メニュー | 画面 | 影響箇所 | 影響レベル |
|---|--------|------|--------|---------|
| 1 | 設定 | **会社情報** | 予算期間設定セクション新規追加（担当者メールアドレスの下） | **新規** |
| 2 | 集計 | **年間予実対比** | 期間セレクタ「2026年1月 〜 2026年4月」→予算年度がデフォルトに | **高** |
| 3 | 集計 | **予実対比** | 期間セレクタの月選択範囲 | **中** |
| 4 | 集計 | **月次レポート** | 月セレクタのグルーピング | **低** |
| 5 | 詳細分析 | **進捗分析** | 月セレクタ + 目標値（FD仕入費率、PA人件費率）はMonthlyBudget参照 | **中** |

### 会社スコープ — 影響なし（7画面）

| メニュー | 画面 | 理由 |
|--------|------|------|
| 売上分析 | ABC分析、商品別売上、曜日別売上・客数、支払種別分析 | 日付範囲で実績データのみ |
| 仕入分析 | 仕入先別分析、商品別分析 | 日付範囲で仕入データのみ |
| 設定 | お支払い | 予算概念なし |

### 店舗スコープ — 影響あり（11画面）

| # | メニュー | 画面 | 影響箇所 | 影響レベル |
|---|--------|------|--------|---------|
| 1 | 予算設定 | **予算設定: 一覧（予算設定値）** | 期間セレクタ「2026年1月 〜 2026年12月」→予算年度がデフォルトに | **高** |
| 2 | 予算設定 | **予算設定: 一覧（予算構成）** | 同上。費用項目の月列もセレクタに連動 | **高** |
| 3 | 予算設定 | **月次予算設定** | 月セレクタのデフォルト | **低** |
| 4 | 集計 | **年間予実対比（サマリー）** | 期間セレクタ「2026年1月 〜 2026年4月」→予算年度がデフォルトに | **高** |
| 5 | 集計 | **年間予実対比（予実対比）** | 同上。費用明細行の月列も連動 | **高** |
| 6 | 集計 | **予実対比** | 期間セレクタの月選択範囲 | **中** |
| 7 | 集計 | **月次集計（MQ分析）** | 月セレクタのグルーピング。目標値はMonthlyBudget参照 | **中** |
| 8 | 集計 | **月次レポート** | 月セレクタ + 12ヶ月テーブルの範囲 | **中** |
| 9 | 集計 | **日次レポート** | 期間セレクタ | **低** |
| 10 | 詳細分析 | **進捗分析** | 月セレクタ + 目標値（利益率、FD仕入費率、PA人件費率） | **中** |
| 11 | 詳細分析 | **感度分析** | 月セレクタ + MonthlyBudget目標値（固定費等） | **中** |

### 店舗スコープ — 影響なし（8画面）

| メニュー | 画面 | 理由 |
|--------|------|------|
| 売上分析 | ABC分析 | 日付範囲で実績データのみ |
| 詳細分析 | FD分析 | 日時範囲で実績データのみ |
| 詳細分析 | 流入分析 | 予約/ウォークインの実績データのみ |
| 設定 | 店舗情報、連携設定、休業日設定 | 予算概念なし |
| 設定 | 費用設定（当月） | 当月表示のみ |

### フェーズ分け

- **フェーズ1（必須 4画面）**: 会社情報（新規）、予算設定:一覧、年間予実対比（会社/店舗共通）
- **フェーズ2（任意）**: 予実対比、月次集計、月次レポート、感度分析、進捗分析の月セレクタ改善

## 設定UI仕様

- **配置場所**: 会社情報画面（会社スコープ）
- **配置位置**: 担当者メールアドレスの下（更新ボタンの上）に独立セクションとして追加
- **設定項目**: 予算開始月（1月〜12月のセレクトボックス）
- **デフォルト値**: 1（1月 = カレンダー年）
- **連動プレビュー**: 選択変更で「予算期間: 2026年4月 〜 2027年3月」を表示
- **権限**: 管理者（Master）/本部（Manager）のみ変更可能

## 修正対象ファイル（最新master準拠）

### コントローラ（4ファイル）

1. `app/controllers/manager/companies_controller.rb` — company_paramsに`:budget_start_month`追加
2. `app/controllers/manager/shops/monthly_budgets_controller.rb` — indexのデフォルトstart/end計算をbudget_start_month起点に
3. `app/controllers/concerns/annual_budget_summary.rb` — `generate_year_months_list`のデフォルト範囲に予算年度考慮
4. `app/controllers/manager/shops/monthly_cost_histories_controller.rb` — `beginning_of_year`/`end_of_year`→予算年度範囲

### ビュー（2ファイル）

1. `app/views/manager/companies/profile.html.haml` — 予算期間設定セクション追加
2. `app/views/manager/shops/monthly_cost_histories/index.html.haml` — 年ナビの表示

### モデル（1ファイル）

1. `app/models/company.rb` — バリデーション、`budget_period`/`budget_months`メソッド追加（Sorbet対応）

## Company モデル追加メソッド（Sorbet対応）

```ruby
validates :budget_start_month, inclusion: { in: 1..12 }, allow_nil: true

sig { params(year: Integer).returns(T::Hash[Symbol, Integer]) }
def budget_period(year)
  s = budget_start_month || 1
  if s == 1
    { from: year * 100 + 1, to: year * 100 + 12 }
  else
    { from: year * 100 + s, to: (year + 1) * 100 + (s - 1) }
  end
end

sig { params(year: Integer).returns(T::Array[T::Hash[Symbol, T.untyped]]) }
def budget_months(year)
  s = budget_start_month || 1
  (0..11).map do |offset|
    m = ((s - 1 + offset) % 12) + 1
    y = offset < (13 - s) ? year : year + 1
    { year: y, month: m, year_month: y * 100 + m,
      date: Time.zone.parse("#{y}-#{m}-01") }
  end
end
```
