# AI Feature Spec — Waitlist Confirmation Probability Predictor
*Addresses Part A Problem 1: Tatkal Booking Crashes at 10:00 AM (and complements Problem 3: Seat Selection)*

## 1. Problem It Solves
When a user is offered a waitlisted (WL) ticket during Tatkal or general booking, they currently have zero information about whether that ticket is likely to confirm before the journey date. This forces a blind gamble — book and hope, or skip the train entirely and miss travel. This directly compounds Problem 1 (Tatkal crashes): users repeatedly retry booking attempts because they don't trust that a WL outcome is worth accepting, adding to peak-time server load.

## 2. Model or API Choice
A custom lightweight gradient-boosted classifier (e.g. XGBoost or LightGBM), not a large language model. This is a structured, tabular prediction problem (will WL#34 confirm by journey date — yes/no, or a probability), which gradient-boosted trees handle efficiently and explainably, with low compute cost and fast inference — critical since predictions need to return in under a second on a high-traffic page. An LLM-based approach (e.g. GPT-4) is unnecessary and far too slow/expensive for this structured prediction task; this is explicitly the kind of "AI for AI's sake" the assignment warns against.

## 3. Training or Input Data
Historical booking and cancellation data: for each train, route, class, quota, and WL position, the historical confirmation rate (whether that exact WL position confirmed by departure, sourced from IRCTC's own booking/PNR history). Additional features: day of week, season/festival proximity, route popularity, typical cancellation rate for that specific train. This data already exists within IRCTC's internal booking systems (PNR status history, cancellation records) — no new external data source is needed, only access to historical records IRCTC already holds.

## 4. How Output Is Shown to the User
On the booking confirmation screen, next to the WL position (e.g. "WL 14"), a simple label appears: "Likely to confirm — 78% based on past trends" or "Unlikely to confirm — 22% based on past trends," color-coded (green/amber/red) but never shown as a guarantee. No raw probability decimals, no jargon — just a plain-language confidence label, since this needs to work for users on low-end phones with limited data literacy, not just power users.

## 5. Fallback When AI Fails or Is Uncertain
If the model has insufficient historical data for a specific train/route combination (e.g. a newly launched train with no booking history), the system shows a neutral message: "Confirmation chance: data not yet available" rather than guessing or hiding the feature entirely. If the prediction API call itself fails or times out (e.g. over 500ms), the booking flow proceeds normally without the prediction label — the core booking action is never blocked by the AI feature failing. This ensures the feature degrades gracefully and never becomes a single point of failure for the booking flow itself, consistent with IRCTC's need to work reliably on 2G connections.

---