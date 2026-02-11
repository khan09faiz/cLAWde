# UI Design & Frontend Architecture

## 1. Tech Stack

| Technology | Purpose |
|------------|---------|
| Next.js | React framework with SSR/SSG |
| React | Component library |
| Tailwind CSS | Utility-first CSS framework |
| PDF.js | In-browser PDF rendering |
| Custom overlay | Clause highlighting system |

---

## 2. Core UI Components

### 2.1 Upload Screen
- Drag-and-drop zone for PDF/DOCX/TXT
- Text paste area for manual clause input
- Privacy mode selector (visible)
- File type indicator
- Upload progress bar

### 2.2 Analysis Dashboard
- **Left panel:** PDF viewer with clause highlighting
- **Right panel:** Analysis results
- **Top bar:** Overall safety score (0-100), privacy mode badge
- **Bottom bar:** Action buttons (export, delete, save)

### 2.3 Clause Analysis View
- Clause list (scrollable, clickable)
- Per-clause detail panel:
  - Purpose explanation
  - Plain English summary
  - Who benefits indicator
  - Risk severity badge (color-coded)
  - "What could go wrong" section

### 2.4 Risk Radar
- Visual radar chart showing 5 risk dimensions
- Color-coded risk levels
- Overall score prominently displayed
- Trend indicators (if comparing versions)

### 2.5 Q&A Interface
- Chat-style interface
- Suggested questions
- Source references (clause citations)
- Legal context citations (when applicable)

### 2.6 "Should I Sign?" Panel
- Large verdict display (Safe / Caution / Do Not Sign)
- Deal breakers list
- Negotiable clauses list
- Acceptable risks list
- Explanation text

### 2.7 Deadline Tracker
- Calendar view of key dates
- Countdown timers
- Auto-renewal warnings
- Notice period alerts

### 2.8 Entity Extraction Panel
- Parties identified
- Jurisdiction & governing law
- Monetary values highlighted
- Contract type classification

---

## 3. Trust UX Elements (Critical)

These must be **always visible**, not buried in settings:

| Element | Placement | Design |
|---------|-----------|--------|
| Auto-delete timer | Top bar | Countdown badge |
| "We do not train on your data" | Footer / upload screen | Text + lock icon |
| Delete button | Every analysis page | Red, prominent |
| Privacy mode indicator | Top bar | Badge with mode name |
| Encryption status | Settings / info panel | Green shield icon |

---

## 4. Color System

| Purpose | Color | Usage |
|---------|-------|-------|
| Safe / Low Risk | Green (#22C55E) | Risk badges, safe verdict |
| Caution / Medium Risk | Amber (#F59E0B) | Warning badges, caution verdict |
| Danger / High Risk | Red (#EF4444) | Alert badges, do-not-sign verdict |
| Primary Action | Blue (#3B82F6) | Buttons, links |
| Background | Slate (#F8FAFC) | Page background |
| Text | Dark (#1E293B) | Body text |

---

## 5. Page Flow

```
Landing Page
     ↓
Upload Contract  ←→  Paste Clause
     ↓
Processing Screen (progress indicator)
     ↓
Analysis Dashboard
  ├── Clause List (left panel)
  ├── Risk Radar (summary view)
  ├── "Should I Sign?" (verdict)
  ├── Q&A Chat (interactive)
  ├── Deadlines (calendar)
  └── Export / Delete
```

---

## 6. Responsive Design

| Breakpoint | Layout |
|------------|--------|
| Desktop (≥1024px) | Side-by-side PDF + analysis |
| Tablet (768-1023px) | Stacked with collapsible panels |
| Mobile (<768px) | Single column, tab navigation |

---

## 7. Key Interactions

### Clause Highlighting
- Click clause in analysis panel → highlight in PDF viewer
- Click text in PDF viewer → show clause analysis
- Color-code highlights by risk level

### Risk Score Animation
- Animated gauge for overall score
- Smooth transitions on score updates
- Pulse animation on high-risk items

### Q&A Flow
- Typewriter effect for AI responses
- Inline source citations (clickable)
- Suggested follow-up questions

---

## 8. Accessibility Requirements

- WCAG 2.1 AA compliance
- Keyboard navigation support
- Screen reader compatible
- Sufficient color contrast ratios
- Focus indicators on interactive elements
- Alt text for all visual indicators
