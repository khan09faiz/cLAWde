# UI/UX Patterns & Guidelines

## 1. Design Philosophy

- **Clarity over cleverness** — Legal analysis must be immediately understandable
- **Trust through transparency** — Privacy controls always visible
- **Progressive disclosure** — Show summaries first, details on demand
- **Accessibility first** — WCAG 2.1 AA compliance minimum

---

## 2. Layout Patterns

### 2.1 Split-Pane Analysis View
- **Left:** Original document (PDF viewer with highlights)
- **Right:** Analysis panel (scrollable, interactive)
- **Linked scrolling:** Clicking a clause in analysis highlights in PDF and vice versa

### 2.2 Card-Based Results
- Each clause analysis in its own card
- Cards include: risk badge, summary, expandable details
- Sortable by risk level, clause order, or category

### 2.3 Dashboard Grid
- Contract cards with status and safety score
- Quick filters: by risk level, date, type
- Search across all contracts

---

## 3. Component Patterns

### 3.1 Risk Badges
```
[Low]     → Green background, dark green text
[Medium]  → Amber background, dark amber text
[High]    → Red background, white text
```
- Always include text label, never color-only (accessibility)
- Consistent sizing across the app

### 3.2 Safety Score Gauge
- Circular gauge (0-100)
- Color transitions: red (0-30) → amber (31-60) → green (61-100)
- Number prominently displayed in center
- Animated on first load

### 3.3 Expandable Sections
- Default: collapsed with summary visible
- Click to expand full analysis
- Smooth animation (200ms ease-in-out)
- Clear expand/collapse indicator

### 3.4 Toast Notifications
- Success: green, auto-dismiss after 3s
- Warning: amber, requires manual dismiss
- Error: red, requires manual dismiss
- Always include descriptive text, never just icons

### 3.5 Loading States
- Skeleton screens for initial loads (not spinners)
- Progress bar with stage labels for document processing
- Estimated time remaining for long operations

---

## 4. Interaction Patterns

### 4.1 Upload Flow
1. Clear drop zone with visual cues
2. File type validation before upload (client-side)
3. Upload progress bar
4. Processing stages with labels:
   - "Extracting text..."
   - "Identifying clauses..."
   - "Analyzing risks..."
   - "Generating summary..."
5. Smooth transition to results

### 4.2 Q&A Chat
- Suggested questions shown as clickable chips
- Typing indicator while LLM processes
- Source citations inline (clickable to jump to clause)
- "Was this helpful?" feedback buttons
- Clear separation between user and AI messages

### 4.3 Clause Navigation
- Breadcrumb showing current position in contract
- Next/Previous clause navigation
- Jump to specific clause via table of contents
- "Back to summary" always accessible

### 4.4 Delete Confirmation
- Two-step confirmation for permanent deletion
- Clear language: "This will permanently delete your contract and analysis"
- Countdown in delete dialog (3 seconds) before action is enabled
- No accidental deletion possible

---

## 5. Typography

| Element | Font Size | Weight | Line Height |
|---------|-----------|--------|-------------|
| Page title | 24px / 1.5rem | Bold (700) | 1.3 |
| Section heading | 20px / 1.25rem | Semibold (600) | 1.4 |
| Card title | 16px / 1rem | Semibold (600) | 1.4 |
| Body text | 14px / 0.875rem | Regular (400) | 1.6 |
| Small text | 12px / 0.75rem | Regular (400) | 1.5 |
| Clause text | 14px / 0.875rem | Regular (400) | 1.8 |

- Use system font stack for performance
- Monospace for legal clause display (optional, test readability)

---

## 6. Spacing System

Use a consistent 4px base unit:
- `4px` — tight spacing (within components)
- `8px` — compact spacing
- `16px` — standard spacing
- `24px` — section spacing
- `32px` — major section separation
- `48px` — page-level spacing

---

## 7. Empty States

Every view needs an empty state:

| View | Empty State Message | Action |
|------|-------------------|--------|
| Dashboard | "No contracts analyzed yet" | "Upload your first contract" button |
| Q&A | "Ask a question about your contract" | Suggested question chips |
| Deadlines | "No deadlines found in this contract" | — |
| Search | "No results found" | "Try different keywords" |

---

## 8. Error States

| Error | User Message | Action |
|-------|-------------|--------|
| Upload failed | "We couldn't process this file. Try a different format." | Retry button |
| LLM timeout | "Analysis is taking longer than expected." | Retry / wait |
| Unsupported format | "This file type isn't supported. Try PDF, DOCX, or TXT." | — |
| Network error | "Connection lost. Your data is safe." | Auto-retry |
| Server error | "Something went wrong. Please try again." | Retry button |

---

## 9. Mobile-Specific Patterns

- Bottom sheet for clause details (instead of side panel)
- Swipeable clause cards
- Sticky risk score header
- Floating action button for Q&A
- Collapsible PDF viewer
- Tab navigation instead of side navigation

---

## 10. Animations & Transitions

| Interaction | Animation | Duration |
|-------------|-----------|----------|
| Page transition | Fade in | 200ms |
| Card expand | Height + fade | 200ms |
| Risk score | Counter animation | 500ms |
| Toast appear | Slide in from top | 150ms |
| Modal open | Scale + fade | 200ms |
| Skeleton shimmer | Continuous pulse | 1.5s loop |

**Rules:**
- Never animate during document processing (focus attention on progress)
- Respect `prefers-reduced-motion` system setting
- No animation longer than 500ms
- Use `ease-in-out` timing function consistently
