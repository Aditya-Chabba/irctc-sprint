# AI Feature Specification: Waitlist Confirmation Probability Predictor

## Problem It Solves
This addresses Part A Problem 1 (Tatkal Booking Crashes at 10:00 AM) and connects to Problem 3 (Seat Selection Resets). When a user is offered a waitlisted (WL) ticket, they have zero information about whether it will confirm before their journey date. This forces a blind gamble, and many users respond by repeatedly retrying Tatkal booking attempts instead of accepting a WL ticket, which directly adds to the peak-time server load documented in Problem 1.

## Proposed Feature — User Perspective
On the booking confirmation screen, right next to the WL position (e.g. "WL 14"), the user sees a plain-language confidence label: "Likely to confirm — 78%" shown in green, or "Unlikely to confirm — 22%" shown in red, based on historical patterns for that exact train, route, class, and WL position. The user can tap the label to see a one-line explanation ("Based on the last 90 days of bookings for this train"). This appears the moment WL availability is shown — no extra click or page needed — and the user can decide whether to book the WL ticket or look for an alternative train with this information in hand.

## Model or API Choice
A custom lightweight gradient-boosted classifier (XGBoost or LightGBM), not a large language model or external API. This is a structured, tabular prediction problem (will WL position #N confirm by journey date), which gradient-boosted trees handle efficiently, explainably, and cheaply — critical since predictions must return in under a second on a high-traffic page. An LLM-based approach (e.g. GPT-4) would be unnecessarily slow and expensive for this structured prediction task, and Vertex AI / Hugging Face general-purpose models would add integration complexity without any benefit over a purpose-built classifier trained on IRCTC's own data.

## Training or Input Data
Historical booking and cancellation data already held internally by IRCTC: for each train, route, class, quota, and WL position, the historical confirmation rate (whether that exact WL position confirmed by departure, sourced from PNR status history and cancellation records). Additional features: day of week, proximity to festivals/holidays, route popularity, and the train's typical cancellation rate. This data already exists within IRCTC's internal booking systems — no new external data source is needed, only internal access to historical PNR and cancellation records that the platform already generates.

## How Output Is Shown to the User
A small color-coded label appears directly beside the WL position on the booking screen:
```
WL 14   [●  Likely to confirm — 78%]
```
Green (≥65% confidence) = "Likely to confirm," Amber (35-64%) = "Uncertain," Red (<35%) = "Unlikely to confirm." No raw decimal probabilities beyond the single percentage, no technical jargon — this needs to work for users with limited data literacy on low-end phones, not just power users. Tapping the label shows a one-line plain-language explanation of what it's based on.

## Confidence Threshold and Fallback
The prediction is only shown if the model has at least 30 historical bookings for that specific train/class/quota/WL-position combination — below this threshold, the system shows a neutral message instead: "Confirmation chance: not enough data yet" rather than guessing with low-confidence data. If the prediction API call itself fails or times out (over 500ms), the booking flow proceeds normally without the prediction label entirely — the core booking action is never blocked or delayed by this feature failing. This ensures the AI feature degrades gracefully and is never a single point of failure for the booking flow.

## Success Metrics
- User satisfaction with WL booking decisions improves, measured via reduced "I shouldn't have booked this WL ticket" support contacts or post-journey survey responses
- Repeat Tatkal booking attempts (re-clicking "Book Now" multiple times in one session) decrease, indicating users trust the WL prediction enough to make a single informed decision instead of gambling via retries
- Prediction accuracy (model's predicted confirmation rate vs actual observed confirmation rate) stays within 10 percentage points when validated against held-out historical data

## Limitations and Risks
The model is trained on historical patterns and cannot account for unprecedented events (e.g. a sudden surge in cancellations due to a festival schedule change, or a new train route with no track record) — this is mitigated by the "not enough data yet" fallback for low-history cases, but the model can still be wrong even with sufficient data if patterns shift suddenly. There is a risk of user over-reliance: if a user treats "78% likely to confirm" as a guarantee and it doesn't confirm, this could create distrust in the platform — the UI must be carefully worded ("based on past trends," never "guaranteed") to set correct expectations. The model may also reflect historical biases (e.g. if a particular train has historically had erratic cancellation patterns due to operational issues now resolved), so the model should be retrained on a rolling basis (e.g. monthly) rather than treated as static, to stay accurate as real-world patterns evolve.