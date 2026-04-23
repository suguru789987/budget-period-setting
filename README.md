# budget-period-setting

ラクミー経営管理 予算期間設定機能(予算開始月) の仕様モックとspec.md。

> **制作:** [@suguru789987](https://github.com/suguru789987) (PdM)
> **担当範囲:** 要件整理 / DB設計 / 18画面影響マップ / Sorbet型付きRubyメソッドの実装コード設計
> **ツール:** Claude Code / HTML / CSS / Markdown (Ruby on Rails / Sorbet / HAML 準拠)

---

## ハイライト

- **companies** テーブルへの `budget_start_month` カラム追加設計
- **影響マップ**: 会社スコープ5画面/7画面非影響、店舗スコープ11画面/8画面非影響を網羅
- **Sorbet型付きRubyメソッド** (`budget_period` / `budget_months`) を仕様書内に記述
- **修正対象ファイル**(コントローラ4+ビュー2+モデル1)をパス付きで列挙
- **フェーズ分け**: 必須4画面(P1) / 任意5画面(P2) の段階リリース設計

## ファイル

| ファイル | 内容 |
|---|---|
| `index.html` | インタラクティブ仕様モック |
| `spec.md` | 詳細仕様書(DB設計/影響マップ/Sorbetコード) |

---

*本リポジトリは在職中の学習/共有目的で公開しており、ファイルパス等はRails標準構成に基づく汎用表現にしています。*
