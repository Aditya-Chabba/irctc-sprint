# Impact vs Effort Matrix

## The Matrix

|                   | Low Effort                                                  | High Effort                                                                 |
|-------------------|---------------------------------------------------------------|------------------------------------------------------------------------------|
| **High Impact**   | Spec 2: Persistent Search Filters<br>Spec 5: Class Change Live Refresh | Spec 1: Tatkal Virtual Queue<br>Spec 3: Seat Selection Persistence<br>AI Feature: Waitlist Confirmation Predictor |
| **Low Impact**    | Spec 4: Logged-Out Search Feedback<br>Spec 6: Session-Aware Results Page | *(none)* |

## How I Scored Each Dimension

### Impact Scoring (1–5)
I scored Impact based on:
- Number of users affected (from Part A frequency analysis)
- Whether the problem is in the core booking flow
- Severity of consequence for the user

| Solution | Users Affected (Part A) | Core Flow? | Consequence | Impact Score |
|----------|--------------------------|------------|--------------|--------------|
| Spec 1: Tatkal Virtual Queue | 20-40 lakh daily | Yes | Trip missed, money/time lost | 5 |
| Spec 2: Persistent Search Filters | 8 crore (all users) | Yes | Time wasted, frustration | 4 |
| Spec 3: Seat Selection Persistence | 30-40% of bookings | Yes | Wrong seat, accessibility harm | 5 |
| Spec 4: Logged-Out Search Feedback | All logged-out visitors | Peripheral (pre-booking) | Frustration, drop-off | 2 |
| Spec 5: Class Change Live Refresh | All users comparing classes | Yes | Misleading data, bad decisions | 4 |
| Spec 6: Session-Aware Results Page | Users w/ multi-tab sessions | Peripheral (edge case) | Page freeze, lost time | 2 |
| AI: Waitlist Confirmation Predictor | Anyone offered WL | Yes | Informed decision-making | 4 |

### Effort Scoring (1–5)
I scored Effort based on:
- Number of system components touched
- Whether new infrastructure is required
- Risk of breaking existing flows
- Railway API dependencies

| Solution | Components Touched | New Infra? | Risk | Backend Dependency | Effort Score |
|----------|---------------------|------------|------|----------------------|--------------|
| Spec 1: Tatkal Virtual Queue | Frontend + Backend + DB + Redis | Yes (queue infra) | Medium-High | High | 5 |
| Spec 2: Persistent Search Filters | Frontend + Backend | No | Low | Medium | 2 |
| Spec 3: Seat Selection Persistence | Frontend + Backend + DB | Minor (hold table) | Medium | Medium-High | 4 |
| Spec 4: Logged-Out Search Feedback | Frontend + Backend (minor) | No | Low | Low | 1 |
| Spec 5: Class Change Live Refresh | Frontend only | No | Low | Low | 1 |
| Spec 6: Session-Aware Results Page | Frontend + Backend (minor) | No | Low | Low | 2 |
| AI: Waitlist Confirmation Predictor | Frontend + Backend + ML pipeline | Yes (ML infra) | Medium | Medium | 4 |

---

## Placement Justifications

### Spec 1: Tatkal Virtual Queue — Major Project (High Impact, High Effort)
This is the single highest-impact fix, directly addressing a daily crash affecting 20-40 lakh users and the platform's most reputationally damaging failure (Part A frequency: daily at 10:00 AM). It requires new queue infrastructure (Redis), new API endpoints, and careful handling of extreme concurrency, making the effort genuinely high per the technical plan. This needs dedicated sprint planning and proper engineering resourcing rather than a quick patch.

### Spec 2: Persistent Search Filters — Quick Win (High Impact, Low Effort)
This affects all 8 crore registered users on nearly every search, one of the highest-reach problems documented in Part A. The fix only requires frontend state management changes and live query parameters per the technical plan — no new infrastructure or database changes. This is a clear do-first item: maximum reach for minimal engineering cost.

### Spec 3: Seat Selection Persistence — Major Project (High Impact, High Effort)
This affects 30-40% of all booking attempts with a severe consequence for affected users — disabled or elderly passengers ending up in inaccessible seats, per Part A's documentation. It requires backend seat-hold infrastructure, new database records, and atomic concurrency handling, making the effort non-trivial per the technical plan. The accessibility implications justify the investment despite the complexity.

### Spec 4: Logged-Out Search Feedback — Fill-In (Low Impact, Low Effort)
This only affects users browsing before logging in, a narrower and less business-critical segment than logged-in bookers actively trying to complete a purchase. The fix is small — an auth-state check before form submission — with no new infrastructure required. This is cheap enough to slot into spare sprint capacity as a polish item rather than a core-metric mover.

### Spec 5: Class Change Live Refresh — Quick Win (High Impact, Low Effort)
Every user comparing classes hits this bug, and it directly causes booking decisions based on stale, misleading fare and availability data (observed consistently in Part A testing). The fix is frontend-only — wiring an existing API call to a dropdown's onChange handler — with zero new infrastructure needed. This is essentially a one-sprint fix with outsized impact on user trust in the platform's data.

### Spec 6: Session-Aware Results Page — Fill-In (Low Impact, Low Effort)
This addresses an edge case (logout while on results page, often via multi-tab usage) affecting a narrower slice of users than the other five problems. The fix is a small frontend auth-listener addition with no new infrastructure required. It improves polish and avoids a confusing dead-end but doesn't move core booking metrics significantly.

### AI Feature: Waitlist Confirmation Predictor — Major Project (High Impact, High Effort)
This improves decision-making for a large share of bookings (anyone offered a WL seat) and directly reduces the blind-gamble repeat-booking behavior that worsens Tatkal server load (connecting back to Problem 1's impact). Building and training the ML model, sourcing historical PNR/cancellation data, and integrating a new prediction pipeline is genuinely high effort compared to a pure UI fix. This should be planned as a dedicated project with data science involvement, not bundled into a quick sprint.

---

## Recommended Sprint Order
1. **Spec 2: Persistent Search Filters** — Highest reach (all 8 crore users), lowest effort; ships fastest and builds team momentum.
2. **Spec 5: Class Change Live Refresh** — Equally low effort, high impact; can be done in parallel with Spec 2 since it only touches frontend.
3. **Spec 1: Tatkal Virtual Queue** — Highest single-problem impact; needs its own dedicated sprint given the new infrastructure required.
4. **Spec 3: Seat Selection Persistence** — High impact on accessibility-critical users; scheduled after the queue work since both touch booking-flow infrastructure and can share learnings.
5. **AI: Waitlist Confirmation Predictor** — High value but depends on having clean historical data pipelines; best sequenced after the core booking flow fixes (1 and 3) stabilize, since reliable booking data improves the model's training set.
6. **Spec 4: Logged-Out Search Feedback** and **Spec 6: Session-Aware Results Page** — Low-effort fill-ins, slotted in whenever spare capacity exists between the above priorities.