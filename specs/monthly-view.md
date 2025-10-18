# 月別ビュー仕様（SBE: Specification by Example）

最終更新: 2025-10-14
対象: 月別（Month）集計ビュー。将来の年別・トータル仕様の土台にする。

## 1. 目的 / ゴール
- 今月（任意月）の飲酒状況を一目で把握し、日次・ラベル別の傾向と目標遵守を確認できる。
- 日次の記録編集・振り返りにスムーズに遷移できる。

## 2. スコープ
- 表示対象: 指定ユーザーの1カ月分の記録（タイムゾーン/締め時間適用後）。
- フィルタ対象: ラベル、時間帯、可視/不可視、タグ等（初期はラベル/可視に限定）。
- 除外: 年別、全期間の深い分析は本仕様の範囲外。

## 3. UI 構成
- KPIカード（上部）
  - 合計杯数、合計容量(ml/oz)、アクティブ日数、1日平均（全日/飲んだ日）、前月比（%）。
- カレンダー（グリッドまたはヒートマップ）
  - 各日の合計杯数/容量の色分け、目標超過マーカー、最頻ラベルアイコン、ツールチップで内訳。
  - クリックで日別詳細モーダル/ページへ遷移。
- 日次推移グラフ（棒/折れ線）
  - スタック棒（ラベル別）＋7日移動平均、累積曲線、目標ベースライン。
- 円グラフ（ドーナツ）
  - ラベル別構成比（Top N＋その他）。
- 平均の表
  - 曜日別平均、ラベル別平均/中央値/標準偏差、時間帯別平均（将来拡張）。

## 4. データ契約（Contract）
Input
```
{
  month: string; // YYYY-MM, 例: "2025-10"
  timezone: string; // IANA, 例: "Asia/Tokyo"
  dayCutoffHour: number; // 0-23, 例: 5
  filters?: {
    labelIds?: string[];
    visibility?: 'visible' | 'all';
  };
}
```
Output
```
{
  period: { start: string; end: string; days: number },
  kpi: {
    totalDrinks: number; // Σ(drink_counters.count)
    totalVolumeMl: number; // Σ(drinks.amount * drink_counters.count)
    activeDays: number;
    avgPerCalendarDay: number; // 合計/日数
    avgPerActiveDay: number;   // 合計/アクティブ日数
    momChangePct?: number;     // 前月比（%）
  },
  // calendar はギャップフィル済み（指定月の全日が必ず1件ずつ存在）
  calendar: Array<{
    date: string; // YYYY-MM-DD（締め時間適用後に属する日）
    count: number; // 当日の Σ(drink_counters.count)
    volumeMl: number;
    topLabelId?: string;
    overGoal?: boolean;
  }>,
  dailySeries: Array<{
    date: string;
    byLabel: Record<string, { count: number; volumeMl: number }>; // labelId -> 集計（count=Σ drink_counters.count）
    total: { count: number; volumeMl: number }
  }>,
  labelDonut: Array<{ labelId: string; count: number; volumeMl: number }>, // count=Σ drink_counters.count
  averages: {
    byWeekday: Array<{ weekday: number; // 0(日)~6(土)
      countAvg: number; volumeAvgMl: number; median?: number; stddev?: number }>,
    byLabel: Array<{ labelId: string; countAvg: number; volumeAvgMl?: number; sharePct: number }>,
    byTimeband?: Array<{ band: string; countAvg: number; sharePct: number }>
  },
  meta: {
    unit: 'ml' | 'oz';
    denseCalendar: true; // カレンダーはゼロ埋め済みで全日分を含むことを示す
  }
}
```

## 5. 集計ロジック
- 合計杯数: Σ(drink_counters.count)
- 合計容量(ml): Σ(drinks.amount × drink_counters.count)
- 1日平均は端数2桁で丸め（四捨五入）。
- 前月比: (今月合計 - 前月合計) / 前月合計 × 100。
- 締め時間の扱い: 記録の timestamp が dayCutoffHour 未満の時は前日として扱う。
 - 日次ギャップフィル: DBに日次レコードがない日も、期間の全日を生成し count/volume/sd を 0 で補完する（密な配列）。
  - 注: 本仕様では純アル/SDは計算しないため、ゼロ埋め対象は count と volume に限る。

## 6. 受入基準（Acceptance Criteria）
- [ ] 指定月の全日が calendar に1件ずつ出力される。DBにレコードが無い日も、アプリ側でゼロ埋め（count=0 など）して返す。
- [ ] フィルタの適用が全ウィジェットで一貫している。
- [ ] 合計杯数の一貫性: kpi.totalDrinks = Σ(calendar[].count) = Σ(dailySeries[].total.count) = Σ(labelDonut[].count)。
- [ ] 目標超過日は overGoal=true が立つ（ユーザー設定の目標値に依存）。
- [ ] カレンダーマスをクリックすると当日詳細に遷移できる。
- [ ] i18n キーのみでテキストが表示される（生文字列なし）。
- [ ] 単位切替で ml/oz の表示が更新される。

## 7. i18n キー（例）
- monthly.kpi.totalDrinks
- monthly.kpi.totalVolume
- monthly.kpi.activeDays
- monthly.kpi.avgPerDay
- monthly.kpi.avgPerActiveDay
- monthly.kpi.mom
- monthly.chart.dailySeries
- monthly.chart.labelDonut
- monthly.table.weekday
- monthly.table.label

## 8. エッジケース
- ABV/容量未入力: 当該指標は未確定扱いで別計上 or 推定値フラグ。
- DST/タイムゾーン跨ぎ: IANA tz で正規化し dayCutoffHour を適用。
- 非表示データ: visibility=visible のときは完全除外。

## 9. テスト観点（Vitest + @nuxt/test-utils）
- 月境界/締め時間ロジックの単体テスト。
- 前月比の計算（前月合計=0のとき NaN/∞ を避け 0% を返す仕様）。
- フィルタの一貫性（calendar/dailySeries/labelDonut の合計一致）。
- i18n キー存在チェック。
 - ゼロ埋め（ギャップフィル）テスト: 取引が0件の月でも calendar.length === period.days となり、全要素の count が 0。
- 杯数集計テスト: ダミーデータで drink_counters.count を変更すると kpi.totalDrinks と各ウィジェット合計が連動して変わる。

## 10. 実装方針（Nuxt/Pinia/ECharts）
- ストア: `store/pages/index/monthlySummary`（または `store/pages/index/` 以下）に集計アクション。
- コンポーネント:
  - `app/components/domain/monthly/KpiCards.vue`
  - `app/components/domain/monthly/CalendarMonthly.vue`
  - `app/components/domain/monthly/TimeseriesDaily.vue`
  - `app/components/domain/monthly/DonutByLabel.vue`
  - `app/components/domain/monthly/TableAverages.vue`
- 型定義: `app/utils/types/monthly.ts` に I/O 契約を定義。
- ロケールキー: `app/utils/locales.ts` に定数を追加。

## 11. 未決事項
- 目標値のデフォルト（ユーザー設定がない場合）。
- Top N の閾値（例: 5）と "その他" の扱い。
- 不明ABV/容量の推定ロジックを入れるか否か。

---
この仕様に合意できれば、続けて型定義の追加とストアの雛形、簡易モック（ダミーデータ）で UI プロトタイプを作成します。
