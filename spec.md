# 予算期間設定 仕様書 v1.2（最新master 2026-04-09 準拠）

## 概要

会社情報画面に「予算開始月」設定を追加し、予算関連の全画面で一貫した予算期間（年度）を適用する。

## 背景・課題

現状、予算の表示期間は「店舗作成日〜現在+3年」の全月リストから手動選択する形式で、**予算年度（会計年度）の概念がない**。月次予算設定は任意の期間を自由選択できるが、4月始まりや10月始まりなどの年度単位で管理する仕組みがない。

## 最新masterの現状（2026-04-09時点）

### データ構造

```
companies テーブル
├── id, name, representive, zipcode, prefecture_id, address, building, phone, email
├── billing_zipcode, billing_prefecture_id, billing_address, billing_building
├── billing_staff_email, billing_staff_name, billing_staff_phone
├── trial_expired_on, extra_trial_expired_on
├── force_enterprise_enabled, is_infomart_api, is_vendor_billing_on
├── scalebase_customer_id, infomart_client_id, infomart_client_secret
├── price_version
└── （budget_start_month は未存在）

monthly_budgets テーブル
├── shop_id, year_month (YYYYMM integer), profit
├── food_cost_rate, labor_cost_rate, sales_per_customer
└── sales_ratio_lunch, sales_ratio_dinner, sales_ratio_food, sales_ratio_drink, sales_ratio_other

monthly_cost_entries テーブル
├── shop_id, year_month (YYYYMM integer)

monthly_cost_items テーブル
├── shop_id, name, immutable

monthly_costs テーブル
├── shop_id, value, ratio, monthly_cost_item_id, monthly_cost_entry_id

shop_rents テーブル
├── shop_id, rent_type, condition_type, fixed_rent, rate, threshold
├── effective_from, effective_until, target_rate, min_guarantee

※ closing_fiscal_year_settings テーブルは schema に存在しない（migrate_temp のみ、未デプロイ）
```

### 現状のコード構造

| ファイル | 現状のロジック |
|---------|---------------|
| `annual_budget_summary.rb` | `generate_year_months_list` で店舗作成日〜now+3年の全月リスト生成。Sorbet型付き。`(1..12)` ハードコードは解消済み |
| `monthly_budgets_controller.rb` | `start_month` / `end_month` パラメータで表示範囲を選択。デフォルトは今年1月〜12月 |
| `budget_vs_actuals_controller.rb` | `set_term` concern で期間管理。`validate_budget_period` で月跨ぎ防止 |
| `monthly_cost_histories_controller.rb` | `beginning_of_year` 〜 `end_of_year` でカレンダー年固定 |
| `mq_analysis_controller.rb` | `set_month` で月選択。24ヶ月ヒストリー |
| `companies_controller.rb` | `company_params` に budget_start_month なし |

## 提案する変更

### 1. DB Migration

```ruby
class AddBudgetStartMonthToCompanies < ActiveRecord::Migration[7.2]
  def change
    add_column :companies, :budget_start_month, :integer, default: 1, comment: "予算開始月（1-12）"
  end
end
```

### 2. Company モデル追加（Sorbet対応）

```ruby
# typed: strict
class Company < ApplicationRecord
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
      { year: y, month: m, year_month: y * 100 + m, date: Time.zone.parse("#{y}-#{m}-01") }
    end
  end
end
```

### 3. Companies Controller

```ruby
def company_params
  params.require(:company).permit(
    :name, :representive, :zipcode, :prefecture_id,
    :address, :building, :phone, :email,
    :budget_start_month  # 追加
  )
end
```

## 影響範囲マップ

### 会社スコープ

| 画面 | 種別 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|------|-----------|----------------|--------------|
| **会社情報** | 設定 | **新規追加** | 予算開始月セレクト、予算期間プレビュー | 既存項目は全て維持 |
| **予実対比** | 表示 | **中** | 月セレクタの年度グルーピング（任意） | set_termロジック、予算/実績データ |

### 店舗スコープ — 設定機能

| 画面 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|-----------|----------------|--------------|
| **予算設定: 一覧** | **高** | `generate_year_months_list` のデフォルト表示範囲（今年1月〜12月→予算年度）、start_month/end_monthのデフォルト値 | 予算データの取得・保存ロジック |
| **費用設定: 履歴** | **高** | `beginning_of_year`/`end_of_year`→予算年度範囲、年ナビ | 費用データの取得・保存ロジック |
| **費用設定: 一覧** | **低** | なし（当月表示のみ） | 全項目維持 |

### 店舗スコープ — 表示機能

| 画面 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|-----------|----------------|--------------|
| **年間予算サマリー** | **高** | `build_annual_budget_summary` の from_date/to_date デフォルト、`@display_year_months` の範囲 | 各月のデータ計算ロジック（年跨ぎ対応済み） |
| **月次集計** | **中** | 月セレクタの年度グルーピング（任意） | 実績データ、目標比較 |
| **感度分析** | **中** | 月セレクタの年度グルーピング（任意） | MQ分析データ |
| **予算コントロール** | **低** | なし（月単位操作） | 日次カレンダー、バイアス設定 |

### フェーズ分け

- **フェーズ1（必須）**: 会社情報（新規）、予算設定:一覧のデフォルト範囲、費用設定:履歴の年範囲
- **フェーズ2（任意）**: 予実対比・月次集計・感度分析の月セレクタ年度グルーピング

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

### 最新版で影響が軽減された点

- `annual_budget_summary.rb`: `(1..12)`ハードコードが既に解消。`generate_year_months_list`は任意のfrom/to日付を受け取れるため、年跨ぎ対応済み
- `monthly_budgets_controller.rb`: `start_month`/`end_month`パラメータで任意範囲選択可能。デフォルト値のみ変更すればよい
- `budget_vs_actuals_controller.rb`: `set_term` concernで期間管理がリファクタリング済み。月単位操作のため影響小
