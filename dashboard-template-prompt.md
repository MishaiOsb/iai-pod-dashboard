# IAI Pod Dashboard — Template Prompt for Claude Code

**Copy and paste this entire prompt into Claude Code to generate your own version of the IAI Pod Dashboard.**

---

## PROMPT START

I need you to build a **single-file HTML dashboard** for managing a portfolio of clients. This is a client account management dashboard used by a Solutions Associate at an AI company. Everything — HTML, CSS, and JavaScript — must be in ONE `.html` file. No build tools, no frameworks, no bundler. It should be deployable directly to GitHub Pages.

### DESIGN SYSTEM

**Theme:** Dark mode by default, with a light mode toggle. Use CSS custom properties for all colours so themes can switch cleanly.

**Colour Tokens:**
```
/* Dark theme (default) */
--bg-primary: #1a1a2e
--bg-secondary: #16213e
--bg-elevated: #1f2b47
--bg-card: #1c2940
--text-primary: #e8e8e8
--text-secondary: #a0a0b0
--text-muted: #6b6b80
--border-default: rgba(255,255,255,0.08)

/* Accent colours */
--accent-purple: #6B30FF (primary CTA, links, active states)
--accent-purple-light: #8B50FF
--accent-orange: #FF6629 (secondary)

/* RAG status */
--rag-green: #00c875
--rag-amber: #fdab3d
--rag-red: #df2f4a

/* Spacing & Radius */
--radius-sm: 6px
--radius-md: 8px
--radius-lg: 12px
--transition-fast: 0.15s
--transition-default: 0.25s
```

**Font:** Inter from Google Fonts (400, 500, 600, 700).

**Light theme** should invert backgrounds to white/light grey and text to dark. Use `[data-theme="dark"]` and `[data-theme="light"]` on `<html>`.

### CORE ARCHITECTURE

Build this as a **Single Page Application (SPA)** using hash-based routing. No page reloads.

**Router:**
```javascript
const Router = {
    current: 'overview',
    navigate(route) {
        // Hide all .page-container, show the target one
        // Update nav active states
        // Store in window.location.hash
    },
    init() {
        // Read hash on load, listen to hashchange
    }
};
```

**Pages to create:**
1. **Overview** (`#/overview`) — Main dashboard with KPI cards, smart alerts, client cards grid
2. **Clients** (`#/clients`) — Full client portfolio with filters and search
3. **Planner** (`#/planner`) — Today's calendar, to-do list, email follow-up tracker, weekly view
4. **Targets** (`#/targets`) — Goal tracking with progress bars
5. **Client Detail** (`#/client/:id`) — Deep dive into a single client with notes, editable fields, agent details

**Navigation bar** (sticky top):
- Logo/title on left
- Page nav links
- Search box
- RAG filter pills (All, Green, Amber, Red)
- Theme toggle (light/dark/system)
- Connection status indicator
- Settings button (opens setup overlay)

### DATA LAYER

**Central state:**
```javascript
const State = {
    clients: [],        // Array of client objects
    filtered: [],       // Clients after applying filters
    filters: { rag: 'all', plan: 'all', search: '' },
    sort: 'health-desc',
    dataLoaded: false,
    alerts: [],

    applyFilters() { /* filter + sort State.clients into State.filtered */ },
    calcHealth(client) { /* return 0-100 score based on RAG, contact recency, billing, etc */ },
};
```

**Client object shape:**
```javascript
{
    id: 'string',
    name: 'Client Name',
    rag: 'Green' | 'Amber' | 'Red',
    plan: 'Activate' | 'Scale' | 'Transform' | 'Pilot',
    billing: 'Yes' | 'No' | 'N/A',
    updateStatus: 'Yes' | 'Working on it' | 'Need Contact',
    agentCount: 3,
    agentStage: 'Live' | 'Pilot' | 'Stalled',
    summary: 'Brief description of client status',
    lastContact: '2026-03-20',
    agents: [{ id, name, stage, platform }],
    // ... extensible
}
```

### monday.com INTEGRATION

Use the monday.com API v2 (GraphQL) to fetch client data from a board.

```javascript
const MondayAPI = {
    _token: '',   // stored in localStorage
    _url: 'https://api.monday.com/v2',

    async query(q) { /* POST GraphQL query with auth header */ },
    async testConnection() { /* query { me { name email } } */ },
    async fetchPodClients(podIndex) {
        // Fetch board items with column values
        // Filter by group/pod
        // Return raw items
    }
};
```

**Setup flow:**
1. Show a setup overlay on first visit
2. User enters monday.com API token
3. Test connection → show user name
4. Select which pod/group to manage
5. Fetch and display client data
6. Store token + pod selection in localStorage for return visits

**Column mapping** — Create a CONFIG object mapping your monday.com column IDs:
```javascript
const CONFIG = Object.freeze({
    BOARD_ID: YOUR_BOARD_ID,
    API_URL: 'https://api.monday.com/v2',
    COLUMNS: {
        RAG: 'color_column_id',
        PLAN: 'status_column_id',
        BILLING: 'status2_column_id',
        // ... map your columns
    },
    AUTO_REFRESH_MS: 300000, // 5 minutes
    STALE_WARNING_DAYS: 7,
    STALE_CRITICAL_DAYS: 14,
});
```

### USER SESSION & POD NAMESPACE

All localStorage keys must be namespaced by pod so multiple pods don't collide:

```javascript
const UserSession = {
    podKey(base) { return RUNTIME.podSlug + '_' + (base || ''); },
    load() { /* restore from localStorage */ },
    save() { /* persist to localStorage */ },
};
```

### FEATURES TO BUILD

**1. KPI Cards (Overview page)**
- Total clients, RAG breakdown (Green/Amber/Red counts), billing active %, average health score
- Each card is clickable to filter the client grid
- Subtle hover animation

**2. Smart Alerts Banner**
- Auto-generated alerts based on data rules:
  - Client not contacted in 7+ days (amber) or 14+ days (red)
  - Red RAG clients
  - Billing not active
  - "Need Contact" update status
- Dismissable individually
- Collapsible section

**3. Client Cards Grid**
- Card per client showing: name, RAG badge, plan badge, billing badge, health score bar, last contact, agent count
- Click to navigate to client detail
- Filterable by RAG, plan, search text
- Sortable by health, name, last contact

**4. Client Detail Page**
- Full client profile with all data
- Editable fields (summary, RAG, plan, billing) — saved to localStorage, optionally synced back
- Notes section (add/delete timestamped notes via NotesManager)
- Agent sub-items list
- Health score breakdown (show each factor and its point contribution)

**5. Notes System**
```javascript
const NotesManager = {
    get(clientId) { /* return array of { text, date } */ },
    add(clientId, text) { /* add note with timestamp */ },
    remove(clientId, index) { /* remove by index */ },
    // Stored in State.localData[clientId].notes
};
```

**6. Health Score Algorithm**
Calculate 0-100 from:
- Base: 40 points
- RAG status: Green +25, Amber +10, Red -15
- Last contact recency: ≤3d +15, ≤7d +8, ≤14d +2, >14d -10
- Update status: Yes +8, Working +3, Need Contact -8
- Billing: Yes +8, No -3
- Agent maturity: Live agents +5-8, stalled -5, none -3
- Notes sentiment: positive keywords +5, negative -10, severe (cancel/churn) -20

**7. To-Do System**
```javascript
const DashTodo = {
    _items: [],  // { id, text, priority, dueDate, completed, autoGenerated }
    load() {},
    save() {},
    add(text, priority, dueDate) {},
    toggle(id) {},
    remove(id) {},
};
```
- Manual task entry with priority (high/medium/low) and due date
- Auto-generated tasks from data (e.g., "Contact [client] — 7 days since last touch")
- Filterable by: today, this week, all, completed

**8. Planner Page**
- Today's calendar section (events from calendar data)
- To-do list with filters
- Email follow-up tracker (which clients need email responses)
- Weekly projection view (Mon-Sun with events + tasks per day)
- All sections collapsible with state persisted to localStorage

**9. Targets System**
```javascript
const TargetsManager = {
    _targets: [],  // { id, label, current, goal, type }
    load() {},
    save() {},
    render() {},  // Progress bars with percentages
};
```
- Auto-targets: "All RAG Green", "All Billing Active", "All Updates Confirmed"
- Manual targets with editable goals
- Visual progress bars

**10. Theme System**
```javascript
const ThemeManager = {
    init() { /* detect saved pref or system pref */ },
    set(theme) { /* 'light' | 'dark' | 'system' */ },
};
```

**11. Proactive Notifications**
- Background polling every 10 minutes
- Browser Notification API for desktop alerts (high priority only)
- In-app notification bell with badge count
- Rules: stale contacts, unresponded emails, upcoming meetings, critical health, Red RAG, billing gaps

**12. Connection Mode**
- Online: full API sync with auto-refresh
- Offline: local data only, all features still work

### STYLING GUIDELINES

- Cards: `background: var(--bg-card); border: 1px solid var(--border-default); border-radius: var(--radius-lg);`
- Hover: subtle border colour change or glow
- Badges (RAG, Plan, Billing): small pill-shaped with background colour
- Health bar: horizontal bar with gradient from red → amber → green
- Buttons: primary uses `--accent-purple`, secondary uses transparent with border
- Follow-up action buttons: ghost/outline pill style (border + text colour, no fill)
- Modals: centred overlay with backdrop blur
- Responsive: works on desktop and tablet, sidebar collapses to burger menu on mobile

### WHAT NOT TO INCLUDE

Do NOT include these features (they require complex proxy/auth setup):
- Slack integration
- OpenAI/Claude AI features
- Fireflies.ai meeting transcripts
- Microsoft MSAL authentication (Outlook, Teams, Calendar sync)
- Cloudflare Worker proxy
- Weekly report sharing via Teams

These can be added later by the user if needed.

### FILE STRUCTURE

Everything in ONE file: `dashboard.html`
```html
<!DOCTYPE html>
<html data-theme="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pod Dashboard</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.7"></script>
    <style>
        /* ALL CSS HERE — design tokens, theme vars, layout, components */
    </style>
</head>
<body>
    <!-- NAV, SETUP OVERLAY, PAGES, MODALS — ALL HTML HERE -->
    <script>
        /* ALL JAVASCRIPT HERE — modules, router, API, rendering */
    </script>
</body>
</html>
```

### GETTING STARTED

1. Create the HTML shell with all CSS variables and theme support
2. Build the navigation bar and SPA router
3. Create the setup overlay with monday.com token input
4. Build the MondayAPI module and data fetching
5. Create the State module with health calculation
6. Render KPI cards and client grid on the overview page
7. Build client detail page with notes
8. Add the planner page with to-do system
9. Add smart alerts
10. Add targets system
11. Add proactive notifications
12. Add search, filters, and sorting

Make it professional, polished, and production-ready. Every interaction should feel smooth. Use CSS transitions on hover states, loading skeletons during data fetch, and toast notifications for user feedback.

## PROMPT END
