# 予算期間設定 仕様書 v1.0

## 概要

会社情報画面に「予算開始月」設定を追加し、予算関連の全画面で一貫した予算期間を適用する。

## 背景・課題

現状、予算の設定側（月次予算設定）と表示側（予実対比、年間予実対比、月次集計、感度分析）で一貫した予算開始月の設定がない。全画面がカレンダー年（1月〜12月）をハードコードしており、4月始まりや10月始まりなどの会計年度に対応できない。

## 機能仕様

### 1. 設定画面

- **配置場所**: 会社情報画面（会社スコープ）
- **配置位置**: 担当者メールアドレスの下（更新ボタンの上）に独立セクションとして追加
- **設定項目**: 予算開始月（1月〜12月のセレクトボックス）
- **デフォルト値**: 1（1月 = カレンダー年）
- **権限**: 管理者（Master）のみ変更可能

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

### 4. 影響を受ける画面

| 画面 | 現状 | 変更後 |
|------|------|--------|
| 予算設定: 一覧（年間予算サマリー） | 1月〜12月固定 | 予算開始月〜12ヶ月 |
| 予実対比 | 月単位選択（制限なし） | 予算年度内の月を選択 |
| 年間予実対比 | カレンダー年 | 予算年度で集計 |
| 月次集計（月次実績トラック） | 24ヶ月ヒストリー | 予算年度単位で表示 |
| 感度分析 | 月選択 | 予算年度内の月を選択 |

### 5. コントローラ修正対象

| ファイル | 修正内容 |
|---------|---------|
| `app/controllers/manager/companies_controller.rb` | `budget_start_month` を `company_params` に追加 |
| `app/controllers/concerns/annual_budget_summary.rb` | `(1..12)` ループを予算開始月起点に変更 |
| `app/controllers/manager/shops/monthly_budgets_controller.rb` | year_month範囲を予算開始月起点に変更 |
| `app/controllers/manager/budget_vs_actuals_controller.rb` | 月選択範囲の予算期間適用 |
| `app/controllers/manager/shops/mq_analysis_controller.rb` | 月選択の予算期間適用 |

### 6. ヘルパーメソッド（Company モデルに追加）

```ruby
class Company < ApplicationRecord
  validates :budget_start_month, inclusion: { in: 1..12 }, allow_nil: true

  # 指定年の予算年度の開始・終了年月を返す
  def budget_period(year)
    start_month = budget_start_month || 1
    if start_month == 1
      { from: year * 100 + 1, to: year * 100 + 12 }
    else
      { from: year * 100 + start_month, to: (year + 1) * 100 + (start_month - 1) }
    end
  end

  # 予算年度の月リスト（表示順）を返す
  def budget_months(year)
    start_month = budget_start_month || 1
    (0..11).map do |offset|
      m = ((start_month - 1 + offset) % 12) + 1
      y = offset < (13 - start_month) ? year : year + 1
      { year: y, month: m, year_month: y * 100 + m }
    end
  end
end
```

## UI仕様

### 会社情報画面の変更

- 担当者メールアドレスの下に水平線で区切り
- 「予算期間設定」セクションヘッダーを追加
- 予算開始月のセレクトボックス（1月〜12月）
- 選択に連動して「予算期間プレビュー」を表示（例: 4月 → 「2026年4月 〜 2027年3月」）
- 既存の「更新」ボタンで一括保存（会社情報と同じフォーム内）
