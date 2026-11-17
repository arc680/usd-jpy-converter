# 仕様書: ドル円変換（買い物用）

単一ファイル `index.html` のみを作成すること。外部リソース（CDN・フォント・画像）は読み込まない。ただし為替レート取得のための下記2つのAPIへの fetch は機能要件として行う。日本語UI、スマートフォン幅で使える1カラムレイアウト。大きな金額表示と大きな入力欄で、店頭で片手で使えることを優先する。

## 目的・前提

- アメリカでの買い物専用。店頭表示は税別が前提で、表示価格（USD）から円の「実感額／実払い額」を出す。
- 単品のみ。チップ計算・複数品の合計は入れない。

## 画面要素

1. **方向切替**: `USD → JPY`（初期）／ `JPY → USD`（予算の円から棚価格ドルを逆引き）
2. **モード切替**: ［シンプル］／［実払い］。切り替えても入力額は保持する。
3. **金額入力**: 数値（小数可）。入力が変わるたびに即時再計算。空・不正なら結果欄にメッセージ。
4. **結果表示**: 大きく表示。USD→JPY なら円（1円単位に丸め、カンマ区切り）、JPY→USD ならドル（小数2桁）。
5. **実払いモードの内訳**（USD→JPY のとき）: 商品価格USD／税額USD／税込合計USD／基準レート／実効レート（手数料込み）／円合計 を行で表示する。JPY→USD のときは「その円で買える棚価格（税別USD）」として逆算し、内訳に税込合計USDと実効レートを表示する。
6. **税率**: 都市プリセットのセレクト＋数値入力。プリセットを選ぶと入力に反映され、入力を手で変えたらプリセット表示は「カスタム」になる。
   - ホノルル 4.712（初期選択）／ニューヨーク 8.875／ロサンゼルス 9.75／サンフランシスコ 8.625／サンディエゴ 7.75／ラスベガス 8.375／シカゴ 10.25／シアトル 10.35／オーランド 6.5／ヒューストン 8.25／ポートランド 0
   - プリセット近くに「税率は都市・地区で異なる目安です」の注意書きを置く。
7. **カード手数料**: プリセットのセレクト＋数値入力（税率と同じ連動方式）。プリセット: 0（現金）／1.60／2.20／3.63（初期選択）／3.85／4.50／カスタム。手数料は国際ブランド基準レートへの上乗せとして扱い、実効レート = 基準レート × (1 + 手数料%/100) とする。
8. **レート欄**: 基準レート（1 USD = ¥◯◯）の表示と数値入力（手入力で上書き可）、更新ボタン、出所と更新日時の表示、取得状態の表示。
9. **保存**: モード・方向・都市/税率・手数料・手入力レートを localStorage に保存し、次回起動時に復元する（localStorage が使えない環境ではセッション内のみでよい）。
10. **フッター**: レート出所の表記（ExchangeRate-API へのリンク https://www.exchangerate-api.com を含む）と「日次レートのため実際の請求額とは差が出ます」の注意書き。

## レート取得

- 起動時に自動取得する。取得元は3つで、**同時問い合わせして最初に届いた妥当な値を採用**する（2026-11-17 の安定化修正で変更。旧仕様の逐次フォールバックは廃止）:
  - ExchangeRate-API: `https://open.er-api.com/v6/latest/USD` の `rates.JPY`（更新日時は `time_last_update_utc`、`result: 'error'` は失敗扱い）
  - Frankfurter: `https://api.frankfurter.dev/v1/latest?base=USD&symbols=JPY` の `rates.JPY`（日付は `date`）
  - Currency-API: `https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/usd.min.json` の `usd.jpy`（日付は `date`）
- 各リクエストには **8秒タイムアウト**（AbortController）を付ける。全部失敗したら **1回だけ自動再試行**する。
- 妥当性チェック: レートは有限な数値で 50〜500 の範囲内のときだけ採用する。
- 全部失敗したら、前回保存したレートを使い「前回のレートを表示中」と状態欄に出す。保存もなければ手入力を促す。
- 更新ボタンで再取得する。手入力した場合は自動取得値より手入力を優先し、その旨が分かる表示にする（再度 更新 を押したら自動取得に戻る、でよい）。手入力後に裏で進んでいた自動取得が完了しても、手入力レートを上書きしない（requestId で無効化）。

## ロジック契約（DOM非依存の関数）

画面とは独立に、Node.js でも評価できる形で次の関数をグローバルに定義すること。初期化処理は `typeof document !== 'undefined'` の内側でのみ行う。丸めはUI表示側だけが行い、関数はフル精度で返す。

### `calcUSDToJPY(amountUSD, opts)` → object
`opts = { mode: 'simple'|'actual', taxPct, feePct, rate }`
- simple: `{ totalJPY: amountUSD * rate, totalUSD: amountUSD, taxUSD: 0, effectiveRate: rate }`
- actual: `taxUSD = amountUSD * taxPct/100`、`totalUSD = amountUSD + taxUSD`、`effectiveRate = rate * (1 + feePct/100)`、`totalJPY = totalUSD * effectiveRate` を返す（キーは simple と同じ4つ）。

### `calcJPYToUSD(amountJPY, opts)` → object（同じ opts）
- simple: `{ shelfUSD: amountJPY / rate, totalUSD: amountJPY / rate, effectiveRate: rate }`
- actual: 棚価格（税別）`shelfUSD = amountJPY / (effectiveRate * (1 + taxPct/100))`、ただし `effectiveRate = rate * (1 + feePct/100)`。`totalUSD = shelfUSD * (1 + taxPct/100)` も返す。

## 受け入れ例（相対誤差 1e-9 以内で一致すること）

- calcUSDToJPY(100, {mode:'simple', taxPct:4.712, feePct:3.63, rate:150}).totalJPY = 15000
- calcUSDToJPY(100, {mode:'actual', taxPct:4.712, feePct:3.63, rate:150}): taxUSD = 4.712、totalUSD = 104.712、effectiveRate = 155.445、totalJPY ≈ 16276.9568（表示は ¥16,277）
- calcUSDToJPY(0, {mode:'actual', taxPct:4.712, feePct:3.63, rate:150}).totalJPY = 0
- calcJPYToUSD(15000, {mode:'simple', taxPct:4.712, feePct:3.63, rate:150}).shelfUSD = 100
- calcJPYToUSD(16276.9568, {mode:'actual', taxPct:4.712, feePct:3.63, rate:150}).shelfUSD ≈ 100
- calcJPYToUSD(15240, {mode:'actual', taxPct:0, feePct:1.6, rate:150}).shelfUSD = 100
