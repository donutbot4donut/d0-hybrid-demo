# D0 Hybrid Demo — Rebuild Brief

## Context

We're building a high-fidelity interactive HTML demo for D0, our proactive AI trading agent product.

There's an existing demo at https://d0-hybrid-proto.vercel.app/ that has the right CORE IDEAS but wrong UX implementation. We need to rebuild it to match our actual production architecture.

## Production Architecture (What We're Matching)

### Two Frontends
1. **dount-HIL-frontend** (app.donutbrowser.ai) — The main trading interface
   - React 19 + Vite + TypeScript + Tailwind + Radix/shadcn
   - Layout: Left sidebar (SidebarTabs: Chat history | Strategy) + Center (TradingView chart + ChartAnalysisOverlay) + Overlapping AI Chat Terminal
   - Key components: TradeDialog (Swap/LimitOrder), TokenSearchDialog, TradingViewChart, SignalPanel
   
2. **donut-d0** (d0.donutbrowser.ai) — D0 management console
   - Next.js 15 + React 19 + Tailwind
   - Dashboard: Bot status, Environment control, Assets Overview, Open Positions

### Core Architecture (from Blog)
- **Dual Environment Separation**: Research Path (LLM + MCP, degradable) vs Execution Path (deterministic, verified)
- **P-E-V Model**: Planner (LLM reasoning → structured plan) → Executor (deterministic, policy-validated) → Verifier (independent check)
- **5-Layer Risk Control**: L1 Stop-Loss → L2 Position Caps → L3 Cooling Periods → L4 AI Review Gate → L5 GRPS (Global Risk Protection System)
- **9-Step Execution Loop**: Set Goals → 24/7 Monitor → Condition Trigger → AI Review → Strategy Execution → Settlement → TP/SL Activation → Push Notification → Capital Continues
- **Approval Inbox**: Async approval pattern — agent prepares executable drafts in background, user confirms/rejects with full context
- **Trust Gradient** (future M3): Progressive autonomy levels based on agent track record

## Problems with Current Demo

1. **Layout mismatch** — Demo: left panel (heatmap + signal feed) + right (chat). Production: left sidebar + center chart + overlapping chat terminal
2. **"Playbook" concept doesn't exist** — We don't have Playbooks in our codebase. The Strategy Hub is a separate future feature
3. **Signal Feed panel doesn't exist** — Signals come through AI chat conversation, not a dedicated panel
4. **Trust Score / Permission Layers are M3 scope** — Not in production yet. Fine to show as concept but shouldn't be the primary UI
5. **Market Heatmap doesn't exist** — Not a production component
6. **Console "3-column" is invented** — No such mode exists
7. **Chat flow is too scripted** — Real D0 uses SSE streaming AI responses, freeform conversation
8. **Missing TradeDialog** — Production has Swap/LimitOrder trade dialog, completely absent
9. **Portfolio view doesn't match** — Production dashboard has Bot Status, Environment Control, not just P&L
10. **The flow conflates research and execution** — Typing "上" in chat shouldn't deploy a strategy. P-E-V separation must be visible

## What to Build

### Single HTML file: `index.html`

A dark-themed, high-fidelity interactive demo that:

### Layout (Terminal View)
```
┌─────────────────────────────────────────────────────────┐
│  D0 logo   [Terminal] [Dashboard]      SOL +$1,240  🟢  │
├────────┬──────────────────────────┬─────────────────────┤
│Sidebar │  TradingView Chart Area  │  AI Chat Terminal   │
│        │  (with candlestick sim)  │                     │
│ Chats  │                          │  [streaming msgs]   │
│ ────── │  Entry/SL/TP overlays    │                     │
│ Signal │                          │  [structured cards] │
│ alerts │  Position info overlay   │                     │
│        │                          │  [input box]        │
├────────┴──────────────────────────┴─────────────────────┤
│ Status bar: Active positions + Risk status + Trust       │
└─────────────────────────────────────────────────────────┘
```

### Layout (Dashboard View)
```
┌─────────────────────────────────────────────────────────┐
│  D0 logo   [Terminal] [Dashboard]      SOL +$1,240  🟢  │
├─────────────────────┬───────────────────────────────────┤
│  Bot Status         │  Approval Inbox                   │
│  Environment Health │  (pending actions with context)   │
├─────────────────────┼───────────────────────────────────┤
│  Active Positions   │  Risk Controls (GRPS)             │
│  (from perps engine)│  - Max Exposure                   │
│                     │  - Position Caps                  │
│                     │  - Circuit Breaker                │
├─────────────────────┼───────────────────────────────────┤
│  P&L Overview       │  Activity Log                     │
│  NAV / Win Rate     │  (decision trace with reasoning)  │
└─────────────────────┴───────────────────────────────────┘
```

### Interactive Flow (Terminal)

The chat interaction should follow this sequence with keyword triggers:

1. **Initial state**: Welcome message + chart showing JUP/USDT. Sidebar shows recent conversations + a signal alert badge.

2. **Click signal in sidebar** → Signal alert appears in chat as a structured message (type: "signal"). Chart highlights the token. AI says: "JUP 刚宣布 $150M Q2 回购，我来拉一下数据..."

3. **AI auto-follows with Research Card** (after 1.5s delay, simulating MCP tool calls):
   - Shows "🔬 Research" card with: signal strength, historical event comparison, on-chain data, confidence level
   - Card clearly labeled as "Research Path" output
   - AI asks: "信号强度不错，要我做个交易计划吗？"

4. **User types "做个计划" or "可以做"** → AI generates P-E-V Plan Card:
   - Labeled "📐 Plan — Planner Output"
   - Shows: Entry zone, Invalidation condition, SL/TP, Position size (% NAV), R:R ratio
   - Below: "⚠️ This plan needs verification before execution"
   - AI says: "计划已生成。需要跑验证吗？"

5. **User types "验证" or "verify"** → Verification Card:
   - Labeled "✅ Verification — Verifier Output"
   - Checklist: IS Sharpe, OOS decay, sample count, max drawdown, R:R
   - Risk check: GRPS constraints pass/fail
   - AI says: "验证通过 (4/5)。要提交到审批收件箱吗？"

6. **User types "提交" or "执行"** → Approval Inbox Card:
   - Labeled "📬 Approval Inbox — Pending"
   - Shows full context: Plan summary + Verification results + Risk assessment
   - Two buttons: "批准执行" + "拒绝"
   - NOT instant execution — makes it clear this goes to async approval
   - AI says: "已提交到审批收件箱。确认后 Agent 会在入场条件达成时自动执行。"

7. **User clicks "批准执行"** → Execution Confirmation:
   - Shows "🚀 Execution Authorized"
   - Details: execution mode (SL auto + entry needs trigger), GRPS constraints active
   - Decision trace: shows the full chain (Signal → Research → Plan → Verify → Approve → Execute)

### Design Requirements

- **Dark theme**: Use the D0 color palette — deep dark bg (#0a0b0f), purple accent (#a855f7), green for gains, red for losses
- **Font**: Inter or system-ui sans-serif, monospace for numbers
- **Cards**: Should have type labels (Research Path / Planner Output / Verifier Output / Approval Inbox) with distinct visual markers
- **Chart**: Simulated candlestick chart using canvas or SVG (doesn't need real data, just realistic-looking)
- **Animations**: Smooth transitions, typing indicators, card slide-in effects
- **Chat**: Messages should appear with typing animation (dots → message). Cards should slide in.
- **Responsive**: Should work well at 1280px+ width

### Chinese UI
- All UI labels in Chinese (same as demo)
- Technical terms can stay English (GRPS, P&L, NAV, R:R, Sharpe)
- Agent messages mix Chinese and English naturally

### Status Bar
Bottom bar showing:
- Active positions with P&L (● JUP +$447  ● SOL +$407)
- Risk status indicator (🟢 GRPS Normal)
- Agent trust score (if space permits)

### Dashboard View
When switching to Dashboard tab:
- **Bot Status**: D0-001, Online, Environment healthy, uptime
- **Approval Inbox**: Show the pending JUP trade + 1-2 historical approved items
- **Active Positions**: JUP LONG +$447 (18.7%), SOL LONG +$407 (8.1%)  
- **Risk Controls**: GRPS parameters — max daily loss, max position size, max concurrent, circuit breaker status
- **P&L Overview**: Total NAV, monthly P&L, win rate, max drawdown
- **Activity Log**: Timestamped entries showing P-E-V decisions (clearly showing the separation)

### Critical UX Rules
1. **P-E-V separation must be visible** — Every card/step must be labeled with which component produced it
2. **Research ≠ Execution** — Chat is research path. Approval Inbox is the bridge to execution path.
3. **No instant execution from chat** — Always goes through Approval Inbox first
4. **Risk controls always visible** — GRPS status in status bar
5. **Decision chain is traceable** — Each step shows what preceded it

## Tech
- Single index.html file, no build tools
- CSS in `<style>` tag
- JS in `<script>` tag
- Canvas or SVG for the chart (no external library — simulate data)
- All self-contained, deployable to Vercel as static

## Reference Colors
```
--bg: #0a0b0f
--bg2: #12131a  
--bg3: #1a1b26
--bg4: #252736
--border: #2a2d3e
--text: #e4e4e7
--text2: #9ca3af
--text3: #6b7280
--purple: #a855f7
--purple2: #7c3aed
--green: #22c55e
--red: #ef4444
--orange: #f59e0b
```
