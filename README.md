# Strategy Event Analyzer (SEA) Format

event sourcing 策略績效檢驗工具.

## 目的

策略產出幾種固定的事件當作 SSoT, 以此計算統一的策略績效.

## 專案類型

- SEA format
  - SEA 可為目錄，或是壓縮的單一檔案(zip compress, .sea or .gsea extension)
  - SEA 的目錄下，必有 strategy.toml
  - SEA strategy 分為 plan & group 兩種，在 strategy.toml 的 [strategy]的 type 指定。 若是type 沒有指定，就是 "plan"
    - plan 為單一策略的 event, 除 events.jsonl 外，還可以有其他 files, 如 metrics.json, strategy.md
    - group 有一個 group.json 與其各策略目錄，若為壓縮格式，則 extension 習慣為 .gsea，但仍可用 .sea，其 plan/group　的判斷，是由 strategy.toml 來判斷。

- group.json format example
  - "group" object 為必要的 entry，為 project list
    - project 中的 "project" 為必要 entry, 它會去找 {project} sea 目錄 或是 {project}.sea 或是 {project}.gsea
    - project_name 通常與 {project} 相同，但確切的 name，要看 sea subproject 下 strategy.toml 的 [strategy] 的 name
    - prject object 的 "count", "weight" 為 optional，default=1
    - metadata 為 optional
  - 除了 "group" 外，其他 entry (i.e. "bonus_setting") 為 optional

```
{
  "group": [
    {
      "project": "aa-1",
      "count": 1,
      "weight": "1",
      "metadata": {
        "color": "Blue"
      }
    },
    {
      "project": "aa-2",
      "count": 3,
      "weight": "1",
      "metadata": {
        "color": "Maroon"
      }
    },
    {
      "project": "aa-3",
      "count": 1,
      "weight": "3.5",
      "metadata": {
        "color": "#FF0000"
      }
    }
  ],
  "bonus_setting": {
    "ref_capital": 840.0,
    "init_capital": 900.0,
    "leverage": 4,
    "fee_rate": 0.05,
    "rebate_rate": 50.0,
    "quarter_share_rate": 30.0,
    "bonus_thresholds": [
      {
        "threshold": 100.0,
        "rate": 10.0
      },
      {
        "threshold": 150.0,
        "rate": 20.0
      },
      {
        "threshold": 200.0,
        "rate": 30.0
      }
    ]
  }
}
```

## 事件類型 (events.jsonl)

- 事件基本結構
  - timestamp (event_time, ISO 8601 format "{YYYY-MM-dd}T{HH:mm:ss}" without timezone if not specified)
  - type
  - data (payload)
- 事件種類 (type)
  - **checkpoint** # should be provided only once
    - start_date: str (yyyy-mm-dd)
    - end_date: str (yyyy-mm-dd)
    - valuation_times: list[str] (optional, hh:mm:ss, ex: 12:00:00, 23:59:00)
  - **fund**
    - currency: str
    - amount: float
    - 注意：base_currency 由第一筆 fund.currency 決定，之後所有 fund.currency 必須一致，否則會報錯
  - **fee**
    - fee_id: str # unique id of a fee model used in mock account
    - type: str # type of the fee model, used for construction
    - ... # model specific args
  - **spec**
    - spec_id: str
    - lot_size: int
    - tick_size: float
    - currency: str
    - multiplier: float (optional = 1)
  - **instrument**
    - instrument_ids: list[str] # all share the same spec_id
    - spec_id: str
  - **fx-rate**
    - from_currency: str
    - to_currency: str
    - rate: float
  - **price**
    - instrument_id: str
    - price: float
  - **trade** (order-filled)
    - instrument_id: str
    - price: float
    - quantity: float
    - metadata: dict (optional)
      - trade_id: str (optional) # 策略邏輯定義的, 一筆交易可能有多筆成交, 用來計算「交易次數、期望值」

**每個 event type 發生時, 對 core mock account 調用哪些方法來更新帳戶資訊:**

**方法命名規則:**

- `record_xxx`: 會產生 derived event（衍生事件）的方法。這些方法會觸發帳戶歷史記錄的更新，例如產生 FundRecord、Trade、Fee、RealizedPnL 等事件記錄。
  - 在 CoreAccount 中，只有 `record_fund` 和 `record_trade` 使用此命名。
- `update_xxx`: 僅調整帳戶內部狀態的方法。這些方法只更新帳戶的配置或狀態，不會產生衍生事件。
  - 例如：`update_price`、`update_instrument_spec`、`update_instrument`、`update_fee_model`、`update_fx_rate` 等。

- checkpoint
  - 本身無須對應, 用來建立 valuation_time
  - 每個 valuation_time 須呼叫 get_account_state 取得帳戶狀況
- fund
  - 第一筆 fund 決定 base_currency
  - record_fund
- fee
  - update_fee_model
- spec
  - update_instrument_spec
- instrument
  - update_instrument
- fx_rate
  - update_fx_rate
- price
  - update_price
- trade
  - record_trade
