# IRCTC 2×2 Impact vs Effort Matrix — Part B

## Pre-Placement Scoring Table

| # | Solution | Users Affected | Severity | Core Flow? | Consequence | Components Touched | New Infra Needed | Risk | Backend Dependency |
|---|----------|----------------|----------|------------|--------------|---------------------|-------------------|------|---------------------|
| 1 | Tatkal Virtual Queue | 20-40 lakh daily | Critical | Yes (core) | Trip missed, money/time lost | Frontend + Backend + DB + Redis | Yes (queue infra) | Medium-High | High |
| 2 | Persistent Search Filters | 8 crore (all users) | Medium | Yes (core) | Time wasted, frustration | Frontend + Backend | No | Low | Medium |
| 3 | Seat Selection Persistence | 30-40% of bookings | High | Yes (core) | Wrong seat, accessibility harm | Frontend + Backend + DB | Minor (hold table) | Medium | Medium-High |
| 4 | Logged-Out Search Feedback | All logged-out visitors | Medium | Peripheral (pre-booking) | Frustration, drop-off | Frontend + Backend (minor) | No | Low | Low |
| 5 | Class Change Live Refresh | All users comparing classes | Medium | Yes (core) | Misleading data, bad decisions | Frontend only | No | Low | Low |
| 6 | Session-Aware Results Page | Users w/ multi-tab sessions | Medium | Peripheral (edge case) | Page freeze, lost time | Frontend + Backend (minor) | No | Low | Low |
| AI | Waitlist Confirmation Predictor | Anyone offered WL | High | Yes (core) | Informed decision-making | Frontend + Backend + ML model | Yes (ML pipeline) | Medium | Medium |

---

## Quadrant Placements

### 🚀 Quick Wins (High Impact, Low Effort)

**Solution 2 — Persistent Search Filters**
Affects all 8 crore registered users on nearly every search, making it one of the highest-reach fixes in this set. The fix only requires frontend state management changes and passing filters as live query parameters — no new infrastructure or database changes. Low risk, low backend dependency, and immediately measurable via reduced filter-mismatch rate, making this a clear do-first item.

**Solution 5 — Class Change Live Refresh**
Every user comparing classes hits this bug, and it directly causes users to make booking decisions on stale, misleading data. The fix is frontend-only — wiring an existing API call to a dropdown's onChange handler — with zero new infrastructure or backend changes needed. This is essentially a one-sprint fix with outsized impact on trust in the platform's data.

### 🏗 Major Projects (High Impact, High Effort)

**Solution 1 — Tatkal Virtual Queue**
This is the single highest-impact fix in the set, directly addressing a daily crash affecting 20-40 lakh users and the platform's most reputationally damaging failure. It requires new infrastructure (Redis-backed queue), new API endpoints, and careful handling of extreme concurrency, making it genuinely high-effort. This needs proper sprint planning and dedicated engineering resourcing rather than a quick patch.

**Solution 3 — Seat Selection Persistence**
This affects 30-40% of all booking attempts and has a severe consequence for affected users (disabled or elderly passengers ending up in inaccessible seats), justifying high impact. It requires backend seat-hold infrastructure, new database records, and careful handling of concurrent seat-hold conflicts, making the effort non-trivial. The accessibility implications make this worth the investment despite the complexity.

**AI — Waitlist Confirmation Predictor**
This feature directly improves decision-making for a huge share of bookings (anyone offered a WL seat) and reduces blind-gamble repeat-booking behavior that worsens Tatkal server load. Building and training an ML model, sourcing historical data, and integrating a new prediction pipeline is genuinely high effort compared to a pure UI fix. This should be planned as a dedicated project with its own data science involvement, not bundled into a quick sprint.

### 🧩 Fill-Ins (Low Impact, Low Effort)

**Solution 4 — Logged-Out Search Feedback**
This only affects users browsing before logging in, a smaller and less business-critical segment than logged-in bookers, so impact is moderate-to-low relative to the others. The fix is small (an auth-state check before form submission) with no new infrastructure, making it cheap enough to slot into spare sprint capacity. It's a nice-to-have polish item rather than a core-metric mover.

**Solution 6 — Session-Aware Results Page**
This addresses an edge case (logout while on results page, often via multi-tab usage) that affects a narrower slice of users than the other problems. The fix is a small frontend auth-listener addition with no new infrastructure, making it low-cost to implement whenever capacity allows. It improves polish and avoids a confusing dead-end but doesn't move core booking metrics significantly.

### ❌ Time Sinks (Low Impact, High Effort)

*No solutions placed in this quadrant.* All 6 Part A problems and the AI feature were chosen specifically because they touch real, frequent pain points with a defensible value proposition — none of them represent low-value, high-cost work. This is intentional: low-value-high-cost ideas were filtered out at the discovery stage rather than carried into Part B.

---