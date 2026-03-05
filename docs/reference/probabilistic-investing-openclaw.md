# OpenClaw cho Swing Trader Việt Nam với tư duy xác suất

## Mục tiêu của tài liệu

Tài liệu này tối ưu cho **một Swing Trader cá nhân** (giữ lệnh vài ngày đến vài tuần), không phải hệ thống quỹ lớn. Mục tiêu là giúp bạn vận hành OpenClaw theo quy trình gọn, kỷ luật, và có thể kiểm chứng:

1. Chọn cơ hội theo **xác suất + kịch bản** thay vì đoán cảm tính.
2. Quản trị vốn theo **R, drawdown, portfolio heat**.
3. Theo dõi lệnh theo **state machine** rõ ràng (WATCH → IN_POSITION → EXITED).
4. Tạo **audit trail** để biết vì sao vào lệnh, vì sao thoát, và vì sao sai.

## Phạm vi tối giản cho nhà đầu tư cá nhân

Dùng kiến trúc 5 lớp là đủ mạnh:

- **Data**: Vnstock Gold/Golden + nguồn chính thức (HOSE/HNX/VSDC) cho sự kiện quan trọng.
- **Feature**: EMA, ATR, RSI, MACD, Rolling VWAP, CVD (nếu có intraday đủ tốt).
- **Signal**: kịch bản Bull/Base/Bear cho D1/H4/H1.
- **Risk**: sizing theo risk%, giới hạn drawdown, giới hạn heat.
- **Ops**: checklist ngày/tuần, log quyết định, review sau lệnh.

Không cần hạ tầng “mini-fund” nếu bạn chỉ giao dịch một danh mục nhỏ.

## Khung quyết định xác suất cho Swing

Mỗi ý tưởng giao dịch bắt buộc có:

- 3 kịch bản: **Bull / Base / Bear**.
- Xác suất tổng = 100%.
- Trigger + invalidation + time window.
- Payoff theo R và EV.

```text
EV_gross = Σ(p_i * R_i)
EV_net   = EV_gross - costs
```

Trong đó `costs` gồm phí + trượt giá ước tính.

### Quy tắc vào lệnh tối thiểu

- Chỉ xem xét lệnh khi `EV_net > 0`.
- Nếu dữ liệu thiếu hoặc mâu thuẫn: không vào lệnh.
- Không dùng ngôn ngữ “chắc chắn”; chỉ dùng phát biểu có điều kiện.

## Bộ chỉ báo và công thức dùng thực chiến

Các chỉ báo đủ dùng cho Swing Việt Nam:

- EMA: 34 / 89 / 200 (xác nhận cấu trúc xu hướng).
- ATR(14): chuẩn hóa stop theo biến động.
- RSI(14): đọc động lượng và trạng thái quá mua/quá bán theo ngữ cảnh trend.
- MACD(12,26,9): xác nhận động lượng và chất lượng nhịp.
- Rolling VWAP(21): vùng giá trị động cho pullback.
- CVD: chỉ bật khi dữ liệu intraday đủ ổn định.

Công thức sizing cơ bản:

```text
risk_amount   = NAV * risk%
position_size = risk_amount / stop_distance
```

`stop_distance` phải đến từ cấu trúc giá hoặc ATR, không đặt tùy hứng.

## Multi-timeframe rule cho D1/H4/H1

### D1 (bối cảnh)

- Ưu tiên long khi `Close > EMA200` và `EMA34 > EMA89`.
- Loại mã ATR% quá thấp (không có biên) hoặc quá cao (rủi ro gap lớn).

### H4 (setup)

- Pullback về EMA34 hoặc Rolling VWAP(21) trong bối cảnh D1 thuận.
- Hoặc breakout khỏi vùng tích lũy có volume xác nhận.

### H1 (trigger)

- Chỉ kích hoạt khi có tín hiệu entry rõ (reclaim, break cấu trúc nhỏ, hoặc momentum xác nhận).
- Nếu dùng CVD thì chỉ coi là tín hiệu phụ, không thay thế cấu trúc giá.

## Tích hợp Vnstock Gold cho OpenClaw

### Nguồn dữ liệu ưu tiên

1. Vnstock cho OHLCV/intraday/news/pipeline.
2. HOSE/HNX/VSDC để xác minh corporate actions và công bố quan trọng.

### Quy tắc vận hành dữ liệu

- Lưu song song `raw` và `adjusted`.
- Gắn `data_quality_flag` cho missing bar, outlier, schema mismatch.
- Nếu feed intraday lỗi: hạ hệ thống về chế độ D1/H4 thay vì cố trade H1.

### Rate limit và ổn định pipeline

- Dùng cache + retry + exponential backoff cho job ingest.
- Tách job theo lớp: EOD, intraday, corporate actions.
- Không gọi API quá dày chỉ để “refresh cảm giác”.

## Risk engine gọn cho Swing Trader

Thiết lập tham số runtime:

- `NAV={NAV}`
- `risk%={risk%}`
- `portfolio_heat_cap={X}%`
- `max_drawdown_trigger={Y}%`

Rule khuyến nghị:

- Nếu DD vượt ngưỡng tuần: giảm risk/trade 50%.
- Nếu DD vượt ngưỡng tháng: chuyển paper/shadow, dừng lệnh thật.
- Nếu tổng heat vượt `X%`: không mở thêm vị thế cùng hướng.

## Luồng vận hành hằng ngày

### Trước phiên

- Cập nhật dữ liệu D1/H4/H1 và kiểm tra data quality.
- Đối chiếu corporate actions/tin công bố trọng yếu.
- Tạo danh sách Top 10 và phát assignment chấm điểm.

### Trong phiên

- Chỉ xử lý mã đã pass hard filter.
- Chỉ gửi đề xuất khi có trigger và risk gate pass.

### Sau phiên

- Cập nhật trạng thái vị thế.
- Ghi log: quyết định, giá vào/ra, lý do, sai lệch so với plan.
- Tạo review ngắn cho lệnh đóng.

## Template chấm điểm Top 10

Mỗi mã cần một `evidence pack` gồm:

- chart D1/H4/H1 có đánh dấu vùng setup,
- score breakdown,
- risk plan (entry/stop/size/heat impact),
- link tin/công bố liên quan,
- kết luận hành động: WATCH / ENTER / NO-TRADE.

Gợi ý trọng số:

- Trend D1: 30%
- Setup H4: 25%
- Trigger H1: 15%
- Liquidity/slippage: 10%
- Risk quality (R:R, stop logic): 15%
- Event risk (tin/corporate actions): 5%

## Prompt mẫu cho OpenClaw

### System prompt

```text
Bạn là OpenClaw hỗ trợ Swing Trading cho thị trường Việt Nam.

Luật bắt buộc:
1) Luôn xuất Bull/Base/Bear, tổng xác suất 100%.
2) Luôn có trigger, invalidation, time window.
3) Luôn tính EV_gross, EV_net.
4) Dùng NAV={NAV}, risk%={risk%}, heat cap={X}%, DD trigger={Y}%.
5) Nếu dữ liệu thiếu/mâu thuẫn thì NO-TRADE hoặc hỏi tối đa 3 câu.
6) Không dùng ngôn ngữ chắc chắn.
```

### Top 10 scoring prompt

```text
Đầu vào: danh sách Top 10 mã cổ phiếu.

Với mỗi mã {TICKER}, xuất:
- Điểm theo 6 nhóm: Trend D1, Setup H4, Trigger H1, Liquidity, Risk quality, Event risk.
- Kịch bản Bull/Base/Bear + xác suất.
- Plan: entry, stop, size theo NAV={NAV}, risk%={risk%}.
- Tác động heat danh mục (cap={X}%).
- Kết luận: WATCH / ENTER / NO-TRADE.
```

### Position monitoring prompt

```text
Theo dõi vị thế {TICKER} sau khi đã vào lệnh.

Yêu cầu:
1) Cập nhật trạng thái: IN_POSITION / RISK_OFF / EXIT_SIGNAL.
2) So sánh giá hiện tại với stop/trailing/target.
3) Nếu có NEW_INFO={NEW_INFO}, cập nhật xác suất và EV_net.
4) Đề xuất hành động: HOLD / REDUCE / EXIT.
```

## Logging và audit bắt buộc

Mỗi quyết định phải lưu:

- timestamp + dữ liệu nguồn,
- snapshot feature,
- kịch bản + xác suất + EV,
- risk check kết quả,
- hành động cuối cùng,
- strategy version/prompt version,
- override reason (nếu có).

## Checklist kiểm thử tối thiểu

1. Data integrity: thiếu bar, outlier, timestamp lệch.
2. Backtest hygiene: walk-forward, không leakage.
3. Cost realism: có phí + slippage trong EV_net.
4. Risk control: test drawdown switch và heat gate.
5. Ops resilience: test retry/backoff và chế độ degraded.

## Roadmap triển khai gọn trong 4 giai đoạn

- **Giai đoạn 1 (1-2 tuần)**: chuẩn hóa ingest + lưu raw/adjusted + báo cáo data quality.
- **Giai đoạn 2 (2-4 tuần)**: feature/signal D1-H4-H1 + Top10 scoring report.
- **Giai đoạn 3 (1-2 tuần)**: risk gate (size/heat/DD) + journal + post-trade review.
- **Giai đoạn 4 (2-3 tuần)**: monitor/alerts + semi-auto execution checklist.

## Kết luận thực dụng

Nếu bạn là Swing Trader nhỏ lẻ, hệ thống tốt nhất không phải hệ phức tạp nhất; đó là hệ thống mà bạn chạy đều mỗi ngày, kỷ luật rủi ro nhất quán, và có log để học từ sai lầm.

OpenClaw nên đóng vai trò **orchestrator + trợ lý phân tích có kiểm soát**, còn quyết định vốn thật vẫn nằm ở bạn theo mô hình human-in-the-loop.
