# Ứng dụng tư duy xác suất trong hệ thống phân tích đầu tư và thiết lập OpenClaw

## Mục tiêu

Tài liệu này mô tả cách vận hành OpenClaw theo hướng ra quyết định dựa trên phân phối kết quả (probabilistic thinking), thay vì dự đoán một kịch bản duy nhất.

Khung áp dụng tập trung vào bốn trụ cột:

1. **Xác suất và kỳ vọng**: mọi quyết định phải có kịch bản, xác suất, payoff theo R, và EV.
2. **Kỷ luật rủi ro**: kích thước vị thế, portfolio heat, drawdown guardrail, risk-off.
3. **Tâm lý giao dịch**: chống overconfidence, recency bias, gambler's fallacy, disposition effect.
4. **Vận hành hệ thống**: logging, audit, kiểm định calibration, kiểm soát execution và safety cho LLM.

## Nguyên tắc vận hành bắt buộc

- Mọi ý tưởng giao dịch phải có ít nhất 3 kịch bản: **Bull / Base / Bear**.
- Tổng xác suất các kịch bản phải bằng 100%.
- Mỗi kịch bản cần có:
  - trigger,
  - invalidation,
  - target theo R,
  - time window.
- Bắt buộc tính:
  - `EV_gross = Σ(p_i * R_i)`,
  - `EV_net = EV_gross - transaction_costs`.
- Nếu không đủ dữ liệu để định lượng xác suất hoặc payoff: phải ghi rõ giả định và độ nhạy của kết luận.

## Công thức cốt lõi

### 1) Position sizing theo fixed risk

```text
risk_amount = NAV * risk%
position_size = risk_amount / stop_distance
```

Trong đó `stop_distance` phải là stop có cấu trúc (structure/ATR), không dùng stop tùy ý.

### 2) Bayesian update

```text
P(H1|E) = P(H1)P(E|H1) / [P(H1)P(E|H1) + P(H0)P(E|H0)]
```

Khi có `NEW_INFO`, phải cập nhật xác suất, EV và hành động tương ứng.

### 3) Utility-adjusted EV (cho hồ sơ loss-averse)

```text
v(R) = R^α              nếu R >= 0
v(R) = -λ|R|^α          nếu R < 0
EU   = Σ(p_i * v(R_i))
```

Luật khuyến nghị:

- `EU < 0` => **NO-TRADE** hoặc yêu cầu cấu trúc payoff tốt hơn.
- `EU >= 0` nhưng vi phạm heat/drawdown => **ALLOW-REDUCED** hoặc **BLOCK**.

## Rule engine rủi ro

Thiết lập tham số runtime:

- `NAV={NAV}`
- `risk%={risk%}`
- `portfolio_heat_cap={X}%`
- `max_drawdown_trigger={Y}%`

Policy gợi ý:

- **Soft stop**: DD tuần vượt ngưỡng => giảm risk/trade xuống 50%.
- **Hard stop**: DD tháng vượt ngưỡng => dừng execution thật, chuyển paper/shadow.
- **Cluster/correlation control**: nếu heat theo cụm đã chạm ngưỡng => chặn hoặc giảm size lệnh mới trong cùng cụm.

## Bộ prompt mẫu cho OpenClaw

### System prompt

```text
Bạn là Openclaw, một hệ thống phân tích và hỗ trợ giao dịch theo Tư duy Xác suất.

Luật bắt buộc:
1) Luôn xuất Bull/Base/Bear, tổng xác suất 100%.
2) Luôn có trigger, invalidation, target (R), time window.
3) Luôn tính EV_gross, EV_net và ghi rõ giả định.
4) Dùng NAV={NAV}, risk%={risk%}, heat cap={X}%, DD trigger={Y}%.
5) Khi có {NEW_INFO}, phải cập nhật xác suất theo Bayes hoặc heuristic có log.
6) Cấm ngôn ngữ chắc chắn; chỉ dùng phát biểu có điều kiện.
7) Nếu thiếu dữ liệu hoặc dữ liệu mâu thuẫn, hỏi tối đa 3 câu hỏi làm rõ.
```

### Pre-trade checklist prompt

```text
Pre-trade checklist cho {TICKER}:

1) Kịch bản Bull/Base/Bear đầy đủ và tổng xác suất = 100%?
2) EV_net > 0 sau phí + trượt giá giả định?
3) Stop logic hợp lệ (structure/ATR)?
4) Lệnh mới có làm heat vượt {X}% hoặc vi phạm drawdown rule {Y}% không?
5) Có dấu hiệu bias hành vi (overconfidence/recency/gambler/disposition) không?

Kết luận: ENTER / ENTER-REDUCED / NO-TRADE.
```

### Bayesian update prompt

```text
Cập nhật cho {TICKER} với NEW_INFO={NEW_INFO}:

1) Nhắc lại xác suất cũ Bull/Base/Bear.
2) Trích xuất evidence E (price/volume/flow/news/structure).
3) Cập nhật xác suất (Bayes nếu có likelihood, nếu không dùng heuristic có log giả định).
4) Nếu invalidation xuất hiện: đặt p kịch bản đó = 0.
5) Cập nhật EV_net và hành động: HOLD / ENTER / REDUCE / EXIT / NO-TRADE.
```

## Logging và audit tối thiểu

Mỗi quyết định phải lưu:

- input dữ liệu (timestamp, nguồn, độ trễ),
- bản tóm tắt feature,
- kịch bản + xác suất + EV/EU,
- risk checks (heat, DD, limits),
- hành động cuối,
- phiên bản model/prompt/config,
- lý do override (nếu có).

## Checklist kiểm thử trước khi chạy thật

1. **Data integrity**: missing/outlier/timestamp drift/corporate actions.
2. **Probability quality**: Brier score, calibration theo rolling window.
3. **Backtest robustness**: walk-forward, giới hạn số lần tuning.
4. **Execution safety**: pre-trade controls, throttle, kill-switch.
5. **LLM safety**: test prompt injection trực tiếp/gián tiếp, kiểm soát output handling.

## Roadmap triển khai gợi ý

- **Giai đoạn 1 (2-4 tuần)**: data pipeline, feature store, logging, paper mode.
- **Giai đoạn 2 (3-6 tuần)**: scenario engine, calibration, Brier dashboard.
- **Giai đoạn 3 (2-4 tuần)**: risk engine (heat/correlation/drawdown), risk override.
- **Giai đoạn 4 (2-6 tuần)**: security hardening, red-team LLM, governance và rollout có gate.

## Ghi chú triển khai

- Nếu chưa có API execution, vận hành ở paper/shadow trước.
- Không tự động tăng rủi ro sau chuỗi thắng/thua.
- Chỉ cho phép override khi có bằng chứng dữ liệu mới và phải audit được.
