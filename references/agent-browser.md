# Browser Automation with `agent-browser`

[agent-browser](https://github.com/vercel-labs/agent-browser) is a CLI tool from Vercel Labs for AI-driven browser automation. It uses a **ref-based interaction model** that saves ~93% of context window compared to traditional DOM/accessibility tree dumps.

## Installation

In a Daytona sandbox:

```bash
# Fix broken yarn repo (Daytona Debian 13 image)
sudo rm -f /etc/apt/sources.list.d/yarn.list

# Install agent-browser + Chromium
npm install -g agent-browser
agent-browser install              # Downloads Chromium binary
agent-browser install --with-deps  # Also installs system libraries (libgbm, etc.)
```

Total install time: ~30 seconds.

## Core Workflow

Every browser automation follows this loop:

1. **Navigate** — `agent-browser open <url>`
2. **Snapshot** — `agent-browser snapshot -i` (get interactive element refs)
3. **Interact** — use refs to click, fill, select

```bash
# Open a page
agent-browser open https://example.com
# ✓ Example Domain

# Get interactive elements with refs
agent-browser snapshot -i
# - link "More information..." [ref=e1]
# - combobox "Search" [ref=e2]
# - button "Submit" [ref=e3]

# Interact using refs
agent-browser click @e1
agent-browser fill @e2 "search query"
agent-browser click @e3
```

Refs like `@e1`, `@e2` are compact, stable identifiers — much cheaper than CSS selectors or XPath.

## Key Commands

### Navigation
```bash
agent-browser open <url>          # Navigate to URL
agent-browser back                # Go back
agent-browser forward             # Go forward
agent-browser reload              # Reload page
```

### Inspection
```bash
agent-browser snapshot            # Full accessibility tree
agent-browser snapshot -i         # Interactive elements only (most useful)
agent-browser snapshot -c         # Compact (remove empty structural elements)
agent-browser snapshot -d 3       # Limit tree depth
agent-browser screenshot          # Take screenshot (PNG)
agent-browser screenshot --full   # Full page screenshot
agent-browser screenshot --annotate  # Labeled screenshot with numbered elements
```

### Interaction
```bash
agent-browser click @e1           # Click element
agent-browser dblclick @e1        # Double-click
agent-browser fill @e1 "text"     # Clear and fill input
agent-browser type @e1 "text"     # Type into element (appends)
agent-browser press Enter         # Press key
agent-browser select @e1 "value"  # Select dropdown option
agent-browser check @e1           # Check checkbox
agent-browser hover @e1           # Hover element
agent-browser scroll down 500     # Scroll
```

### Data Extraction
```bash
agent-browser get text @e1        # Get element text
agent-browser get html @e1        # Get element HTML
agent-browser get value @e1       # Get input value
agent-browser get title           # Get page title
agent-browser get url             # Get current URL
agent-browser eval "document.title"  # Run JavaScript
```

### Waiting
```bash
agent-browser wait @e1            # Wait for element to appear
agent-browser wait 2000           # Wait 2 seconds
agent-browser wait --load networkidle  # Wait for network to settle
```

## Headless Only on Daytona

agent-browser runs in **headless mode** by default, which works perfectly on Daytona sandboxes. The `--headed` flag (visible browser window) does NOT work — there's no X server.

For debugging, use `screenshot --annotate` to see what the browser sees, or `snapshot` to inspect the page structure.

## Sessions

Sessions keep the browser state (cookies, localStorage) between commands:

```bash
# Named session — isolated browser instance
agent-browser --session myapp open https://app.example.com

# Auto-save/restore state across sandbox restarts
agent-browser --session-name myapp open https://app.example.com
```

Multiple sessions can run concurrently for multi-tab or multi-site workflows.

## Authenticating with Miriad Board Apps

Board apps served from the Miriad filesystem require a session cookie for API access. In a headless browser, use the **agent login endpoint** to authenticate:

```
GET /auth/agent-login?token=<MIRIAD_SPACE_TOKEN>&redirect=<path>
```

This endpoint validates the space token, sets a `miriad-agent-session` cookie (space-scoped, HttpOnly, Secure, SameSite=Lax), and 302 redirects to the target path. The `redirect` parameter is **required** (returns 400 without it) and must be a same-origin path.

### Authentication Flow

```bash
# Environment variables are auto-injected in sandboxes:
#   MIRIAD_API_URL, MIRIAD_SPACE_ID, MIRIAD_CHANNEL_ID, MIRIAD_SPACE_TOKEN

# 1. Authenticate and redirect straight to your board app
agent-browser open "$MIRIAD_API_URL/auth/agent-login?token=$MIRIAD_SPACE_TOKEN&redirect=/channels/$MIRIAD_CHANNEL_ID/raw/my-app/index.html"

# 2. Browser now has a session cookie — all subsequent navigation is authenticated
agent-browser snapshot -i
agent-browser click @e1
```

After step 2, the browser has a valid session cookie. All subsequent navigation and API calls from the board app work automatically — the cookie persists for the session.

### Testing Board Apps End-to-End

A typical board app testing workflow:

```bash
# Install agent-browser in sandbox
sudo rm -f /etc/apt/sources.list.d/yarn.list   # Fix broken yarn repo on Daytona
npm install -g agent-browser
agent-browser install --with-deps

# Authenticate and navigate to the app
agent-browser open "$MIRIAD_API_URL/auth/agent-login?token=$MIRIAD_SPACE_TOKEN&redirect=/channels/$MIRIAD_CHANNEL_ID/raw/my-app/index.html"

# Wait for the app to load (SPAs may fetch data async)
agent-browser wait 3000

# Verify the app rendered correctly
agent-browser snapshot -i
agent-browser screenshot /tmp/app.png

# Interact with the app
agent-browser fill @e1 "search query"
agent-browser wait 2000
agent-browser screenshot /tmp/after-interaction.png

```

### Security Notes

- **Token in URL is acceptable** — the browser runs headless in a scrubbed sandbox with no history persistence
- **Session cookie is space-scoped** — matches the space token's permissions (content/work objects, denied config routes)
- **Same-origin redirect only** — the `redirect` parameter is validated to prevent open redirect attacks
- **Cookie lifetime** — session cookie (no explicit expiry), dies when the browser process ends
- **Use `MIRIAD_SPACE_TOKEN`** from the sandbox environment — it's injected automatically

## Practical Examples

### Fill a form
```bash
agent-browser open https://example.com/signup
agent-browser snapshot -i
# - textbox "Email" [ref=e1]
# - textbox "Password" [ref=e2]
# - button "Sign Up" [ref=e3]
agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "securepassword"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i  # Check result
```

### Extract data from a page
```bash
agent-browser open https://news.ycombinator.com
agent-browser snapshot  # Full tree with text content
# Parse the output for story titles, links, scores, etc.
```

### Take an annotated screenshot
```bash
agent-browser open https://example.com
agent-browser screenshot --annotate --full /tmp/annotated.png
# Creates a labeled screenshot with numbered elements and a legend
```

## Tips

- **Always use `snapshot -i`** for interaction — it's the most token-efficient view
- **Chain commands** with `&&` — the browser daemon persists between calls
- **Use `wait --load networkidle`** after navigation for SPAs that load data asynchronously
- **`get text`** is great for extracting specific content without parsing the full snapshot
- **`eval`** lets you run arbitrary JavaScript for complex extraction or interaction
- **Refs reset on each snapshot** — always take a fresh snapshot before interacting
