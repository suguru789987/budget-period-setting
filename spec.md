# 予算期間設定 仕様書 v1.1

## 概要

会社情報画面に「予算開始月」設定を追加し、予算関連の全画面で一貫した予算期間を適用する。

## 背景・課題

現状、予算の設定側（月次予算設定・費用設定）と表示側（予実対比、年間予実対比、月次集計、感度分析）で一貫した予算開始月の設定がない。全画面がカレンダー年（1月〜12月）をハードコードしている。

## 影響範囲マップ

### 会社スコープ

| 画面 | 種別 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|------|-----------|----------------|--------------|
| **会社情報** | 設定 | **新規追加** | 予算開始月セレクト、予算期間プレビュー | 既存項目は全て維持 |
| **予実対比** | 表示 | **高** | 月セレクタの選択肢範囲 | 予算売上、FD仕入、人件費、固定費、利益 |

### 店舗スコープ — 設定機能

| 画面 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|-----------|----------------|--------------|
| **予算設定: 一覧** | **高** | 年ナビ（前年/翌年→前年度/翌年度）、月リスト範囲（1-12→開始月起点12ヶ月）、year_month範囲、設定リンク | 月次利益、仕入比率、PA人件費比率、客単価 |
| **費用設定: 履歴** | **高** | 年ナビ、year_month範囲、月行の表示順序 | 費用予算額、比率表示 |
| **費用設定: 一覧** | **低** | なし（当月表示のみ） | 全項目維持 |

### 店舗スコープ — 表示機能

| 画面 | 影響レベル | 影響を受ける項目 | 影響なしの項目 |
|------|-----------|----------------|--------------|
| **年間予算サマリー** | **高** | 月ループ `(1..12)`→開始月起点、ヘッダ列、year_month計算、全データ行のindex | （全項目が表示順序の影響を受ける） |
| **月次集計** | **中** | 月セレクタのグルーピング（任意） | 売上実績、客数、FD原価率、PA人件費率、目標比較 |
| **感度分析** | **中** | 月セレクタのグルーピング（任意） | MQ分析データ、計算ロジック |
| **予算コントロール** | **中** | 月ナビの範囲制限（任意） | 日次カレンダー、バイアス設定 |

### フェーズ分け

- **フェーズ1（必須）**: 会社情報（新規）、予算設定:一覧、年間予算サマリー、費用設定:履歴
- **フェーズ2（任意）**: 予実対比、月次集計、感度分析、予算コントロール（月セレクタの改善）

## 機能仕様

### 1. 設定画面

- **配置場所**: 会社情報画面（会社スコープ）
- **配置位置**: 担当者メールアドレスの下（更新ボタンの上）に独立セクションとして追加
- **設定項目**: 予算開始月（1月〜12月のセレクトボックス）
- **デフォルト値**: 1（1月 = カレンダー年）
- **権限**: 管理者（Master）/本部（Manager）のみ変更可能

### 2. DBカラム

```ruby
# Migration
add_column :companies, :budget_start_month, :integer, default: 1
# validation: 1..12
```

### 3. 予算年度の計算ロジック

予算開始月を `S` とした場合:

| 設定 | 予算年度の範囲 | 例（2026年度） |
|------|---------------|---------------|
| S=1 | Y年1月 〜 Y年12月 | 2026/01 〜 2026/12 |
| S=4 | Y年4月 〜 Y+1年3月 | 2026/04 〜 2027/03 |
| S=10 | Y年10月 〜 Y+1年9月 | 2026/10 〜 2027/09 |

### 4. 修正対象ファイル

#### コントローラ（6ファイル）

1. `app/controllers/manager/companies_controller.rb` — company_paramsに追加
2. `app/controllers/manager/shops/monthly_budgets_controller.rb` — 月リスト・year_month範囲
3. `app/controllers/concerns/annual_budget_summary.rb` — (1..12)ループ→予算年度
4. `app/controllers/manager/shops/monthly_cost_histories_controller.rb` — year_month範囲
5. `app/controllers/manager/budget_vs_actuals_controller.rb` — 月セレクタ範囲
6. `app/controllers/manager/shops/mq_analysis_controller.rb` — 月セレクタ

#### ビュー（7ファイル）

1. `app/views/manager/companies/profile.html.haml` — 予算期間設定セクション追加
2. `app/views/manager/shops/monthly_budgets/index.html.haml` — 月リスト・年ナビ
3. `app/views/manager/shops/monthly_budgets/annual_budget_summary.html.haml` — 月ヘッダ
4. `app/views/manager/shops/monthly_cost_histories/index.html.haml` — 年ナビ・月行
5. `app/views/manager/budget_vs_actuals/index.html.haml` — 月セレクタ
6. `app/views/manager/shops/mq_analysis/monthly_track.html.haml` — 月セレクタ
7. `app/views/manager/shops/mq_analysis/monthly.html.haml` — 月セレクタ

#### モデル（1ファイル）

1. `app/models/company.rb` — バリデーション、budget_period/budget_monthsメソッド追加

### 5. Company モデルに追加するメソッド

```ruby
validates :budget_start_month, inclusion: { in: 1..12 }, allow_nil: true

def budget_period(year)
  s = budget_start_month || 1
  if s == 1
    { from: year * 100 + 1, to: year * 100 + 12 }
  else
    { from: year * 100 + s, to: (year + 1) * 100 + (s - 1) }
  end
end

def budget_months(year)
  s = budget_start_month || 1
  (0..11).map do |offset|
    m = ((s - 1 + offset) % 12) + 1
    y = offset < (13 - s) ? year : year + 1
    { year: y, month: m, year_month: y * 100 + m }
  end
end
```

### 6. 参考: 既存 ClosingFiscalYearSetting

- 決算期設定として `ClosingFiscalYearSetting` モデルが存在（`fiscal_year_start_month`、デフォルト: 4月）
- 予算期間設定は決算期とは独立した設定としてCompanyモデルに直接追加
- 将来的に両者の連動も検討可能
