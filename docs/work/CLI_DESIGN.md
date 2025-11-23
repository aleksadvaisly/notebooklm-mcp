# NotebookLM CLI - Golang Implementation Design

## Overview

Golang CLI implementation of NotebookLM MCP, providing direct access to NotebookLM's AI-powered knowledge bases from the command line. Uses Cobra for CLI framework and chromedp for browser automation.

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| CLI Framework | [cobra](https://github.com/spf13/cobra) | Commands, flags, help generation |
| Configuration | [viper](https://github.com/spf13/viper) | Config files, env vars, flags |
| Browser | [chromedp](https://github.com/chromedp/chromedp) | Chrome DevTools Protocol automation |
| Storage | JSON files / SQLite | Library persistence |
| Logging | [zerolog](https://github.com/rs/zerolog) | Structured logging |

## Project Structure

```
notebooklm-cli/
├── cmd/
│   └── notebooklm/
│       └── main.go                 # Entry point
│
├── internal/
│   ├── cli/
│   │   ├── root.go                 # Root command
│   │   ├── ask.go                  # ask command
│   │   ├── notebook.go             # notebook subcommands
│   │   ├── session.go              # session subcommands
│   │   ├── auth.go                 # auth subcommands
│   │   └── config.go               # config subcommands
│   │
│   ├── browser/
│   │   ├── context.go              # SharedContextManager
│   │   ├── session.go              # BrowserSession
│   │   ├── stealth.go              # Human-like behavior
│   │   └── selectors.go            # CSS selectors
│   │
│   ├── auth/
│   │   ├── manager.go              # AuthManager
│   │   └── state.go                # Cookie/state persistence
│   │
│   ├── library/
│   │   ├── notebook.go             # NotebookEntry struct
│   │   ├── manager.go              # NotebookLibrary
│   │   └── storage.go              # JSON/SQLite storage
│   │
│   ├── mcp/
│   │   ├── server.go               # MCP server mode
│   │   ├── tools.go                # Tool definitions
│   │   └── handlers.go             # Tool handlers
│   │
│   └── config/
│       ├── config.go               # Configuration management
│       └── paths.go                # Cross-platform paths
│
├── pkg/
│   └── pageutils/
│       ├── extract.go              # DOM extraction
│       └── wait.go                 # Wait for answer
│
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

## CLI Commands

### Root Command

```bash
notebooklm [command] [flags]

Flags:
  --config string     Config file (default: ~/.config/notebooklm-cli/config.yaml)
  --headless          Run browser in headless mode (default: true)
  --verbose           Verbose output
  --timeout duration  Browser timeout (default: 30s)
```

### Ask Command

```bash
notebooklm ask "What are React hooks?" [flags]

Flags:
  --session string     Session ID (reuse conversation)
  --notebook string    Notebook ID from library
  --url string         Direct NotebookLM URL
  --show-browser       Show browser window
  --format string      Output format: text|json|markdown (default: text)
  --new-session        Force new session (ignore active)

Examples:
  notebooklm ask "Explain useState"
  notebooklm ask "How does useEffect work?" --session react-hooks
  notebooklm ask "Compare to Vue" --notebook my-react-docs

Resolution order:
  1. --session flag (explicit)
  2. Active session (from `session select`)
  3. New auto-generated session

  1. --notebook flag (explicit)
  2. Active notebook (from `notebook select`)
  3. Error if none set
```

### Notebook Commands

```bash
notebooklm notebook [command]

Commands:
  add         Add notebook to library
  list        List all notebooks
  get         Get notebook details
  select      Set active notebook
  update      Update notebook metadata
  remove      Remove notebook from library
  search      Search notebooks
  stats       Show library statistics

Examples:
  notebooklm notebook add --url "https://notebooklm.google.com/..." \
    --name "React Docs" \
    --topics "React,Hooks,Components"

  notebooklm notebook list
  notebooklm notebook select react-docs
  notebooklm notebook search "typescript"
```

### Session Commands

```bash
notebooklm session [command]

Commands:
  list        List active sessions
  select      Set active session (sticky)
  close       Close a session
  reset       Reset session (clear history)
  cleanup     Clean up all inactive sessions
  current     Show current active session

Examples:
  notebooklm session list
  notebooklm session select react-session   # set as default
  notebooklm session close react-session
  notebooklm session reset react-session
  notebooklm session current
```

### Auth Commands

```bash
notebooklm auth [command]

Commands:
  setup       Interactive Google login
  status      Check authentication status
  logout      Clear saved authentication
  reauth      Re-authenticate (switch account)

Examples:
  notebooklm auth setup --show-browser
  notebooklm auth status
  notebooklm auth reauth
```

### Config Commands

```bash
notebooklm config [command]

Commands:
  get         Get configuration value
  set         Set configuration value
  list        List all configuration
  reset       Reset to defaults

Examples:
  notebooklm config set headless false
  notebooklm config set session.timeout 20m
  notebooklm config get profile
  notebooklm config list
```

### MCP Server Mode

```bash
notebooklm serve [flags]

Flags:
  --profile string    Tool profile: minimal|standard|full (default: standard)
  --stdio             Use stdio transport (default)
  --port int          HTTP port for SSE transport

Examples:
  notebooklm serve --profile full
  notebooklm serve --stdio
```

## Core Components

### 1. Browser Context Manager (chromedp)

```go
package browser

import (
    "context"
    "github.com/chromedp/chromedp"
)

type ContextManager struct {
    allocCtx    context.Context
    browserCtx  context.Context
    userDataDir string
    headless    bool
}

func NewContextManager(cfg *config.Config) (*ContextManager, error) {
    opts := append(chromedp.DefaultExecAllocatorOptions[:],
        chromedp.Flag("headless", cfg.Headless),
        chromedp.UserDataDir(cfg.UserDataDir),
        chromedp.Flag("disable-blink-features", "AutomationControlled"),
        chromedp.UserAgent("Mozilla/5.0 ..."),
    )

    allocCtx, _ := chromedp.NewExecAllocator(context.Background(), opts...)
    browserCtx, _ := chromedp.NewContext(allocCtx)

    return &ContextManager{
        allocCtx:    allocCtx,
        browserCtx:  browserCtx,
        userDataDir: cfg.UserDataDir,
        headless:    cfg.Headless,
    }, nil
}

func (cm *ContextManager) NewTab() (context.Context, context.CancelFunc) {
    return chromedp.NewContext(cm.browserCtx)
}
```

### 2. Browser Session

```go
package browser

type Session struct {
    ID          string
    NotebookURL string
    ctx         context.Context
    cancel      context.CancelFunc
    msgCount    int
    lastActive  time.Time
}

func (s *Session) Ask(question string) (string, error) {
    var answer string

    err := chromedp.Run(s.ctx,
        // Click input field
        chromedp.Click(InputSelector, chromedp.NodeVisible),

        // Type with human-like delays
        humanType(question),

        // Submit
        chromedp.KeyEvent("\r"),

        // Wait for answer
        waitForAnswer(&answer),
    )

    s.msgCount++
    s.lastActive = time.Now()

    return answer, err
}
```

### 3. Stealth Utils (Human-like Behavior)

```go
package browser

import (
    "math/rand"
    "time"
    "github.com/chromedp/chromedp"
)

// Human typing at 160-240 WPM
func humanType(text string) chromedp.ActionFunc {
    return func(ctx context.Context) error {
        for _, char := range text {
            // Random delay between keystrokes (50-150ms)
            delay := time.Duration(50+rand.Intn(100)) * time.Millisecond
            time.Sleep(delay)

            if err := chromedp.KeyEvent(string(char)).Do(ctx); err != nil {
                return err
            }
        }
        return nil
    }
}

// Random delay between actions
func randomDelay(min, max time.Duration) chromedp.ActionFunc {
    return func(ctx context.Context) error {
        delay := min + time.Duration(rand.Int63n(int64(max-min)))
        time.Sleep(delay)
        return nil
    }
}
```

### 4. Page Utils (Answer Extraction)

```go
package pageutils

const (
    // Primary selectors for NotebookLM UI
    ResponseSelector = ".to-user-container .message-text-content"
    ThinkingSelector = "div.thinking-message"
    InputSelector    = "textarea[placeholder*='Ask']"
)

func waitForAnswer(answer *string) chromedp.ActionFunc {
    return func(ctx context.Context) error {
        // Wait for thinking indicator to disappear
        chromedp.WaitNotPresent(ThinkingSelector).Do(ctx)

        // Poll until answer stabilizes
        var lastHash string
        for {
            var text string
            chromedp.Text(ResponseSelector, &text, chromedp.NodeVisible).Do(ctx)

            hash := hashText(text)
            if hash == lastHash {
                *answer = text
                return nil
            }
            lastHash = hash
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

### 5. Auth Manager

```go
package auth

type Manager struct {
    statePath string
    cookies   []*network.Cookie
}

func (m *Manager) LoadState() error {
    data, err := os.ReadFile(m.statePath)
    if err != nil {
        return err
    }
    return json.Unmarshal(data, &m.cookies)
}

func (m *Manager) SaveState(ctx context.Context) error {
    cookies, err := network.GetCookies().Do(ctx)
    if err != nil {
        return err
    }

    data, _ := json.Marshal(cookies)
    return os.WriteFile(m.statePath, data, 0600)
}

func (m *Manager) IsValid() bool {
    for _, c := range m.cookies {
        if c.Name == "SAPISID" && c.Expires > float64(time.Now().Unix()) {
            return true
        }
    }
    return false
}
```

### 6. Notebook Library

```go
package library

type NotebookEntry struct {
    ID           string    `json:"id"`
    URL          string    `json:"url"`
    Name         string    `json:"name"`
    Description  string    `json:"description"`
    Topics       []string  `json:"topics"`
    Tags         []string  `json:"tags"`
    UseCases     []string  `json:"use_cases"`
    QueryCount   int       `json:"query_count"`
    LastAccessed time.Time `json:"last_accessed"`
    CreatedAt    time.Time `json:"created_at"`
}

type Library struct {
    Notebooks         map[string]*NotebookEntry `json:"notebooks"`
    ActiveNotebookID  string                    `json:"active_notebook_id"`
    ActiveSessionID   string                    `json:"active_session_id"`
    storagePath       string
}

func (l *Library) SetActiveSession(sessionID string) error {
    l.ActiveSessionID = sessionID
    return l.Save()
}

func (l *Library) ClearActiveSession() error {
    l.ActiveSessionID = ""
    return l.Save()
}

func (l *Library) Add(entry *NotebookEntry) error {
    entry.ID = generateID(entry.Name)
    entry.CreatedAt = time.Now()
    l.Notebooks[entry.ID] = entry
    return l.Save()
}

func (l *Library) Search(query string) []*NotebookEntry {
    var results []*NotebookEntry
    query = strings.ToLower(query)

    for _, nb := range l.Notebooks {
        if strings.Contains(strings.ToLower(nb.Name), query) ||
           strings.Contains(strings.ToLower(nb.Description), query) ||
           containsAny(nb.Topics, query) ||
           containsAny(nb.Tags, query) {
            results = append(results, nb)
        }
    }
    return results
}
```

## Configuration

### Config File (~/.config/notebooklm-cli/config.yaml)

```yaml
# Browser settings
browser:
  headless: true
  timeout: 30s
  user_data_dir: ~/.local/share/notebooklm-cli/browser

# Session settings
session:
  max_concurrent: 10
  timeout: 15m
  cleanup_interval: 60s

# Authentication
auth:
  auto_login: false
  email: ""
  password: ""

# Stealth settings
stealth:
  enabled: true
  typing_wpm_min: 160
  typing_wpm_max: 240
  delay_min: 100ms
  delay_max: 400ms

# Default notebook
notebook:
  url: ""
  description: ""
  topics: []

# MCP settings
mcp:
  profile: standard  # minimal|standard|full
  disabled_tools: []
```

### Environment Variables

```bash
NOTEBOOKLM_HEADLESS=true
NOTEBOOKLM_TIMEOUT=30s
NOTEBOOKLM_MAX_SESSIONS=10
NOTEBOOKLM_SESSION_TIMEOUT=15m
NOTEBOOKLM_AUTO_LOGIN=false
NOTEBOOKLM_EMAIL=user@example.com
NOTEBOOKLM_PASSWORD=secret
NOTEBOOKLM_NOTEBOOK_URL=https://notebooklm.google.com/...
```

## MCP Server Implementation

### Tool Definitions

```go
package mcp

var Tools = []Tool{
    {
        Name:        "ask_question",
        Description: "Ask a question to NotebookLM",
        InputSchema: JSONSchema{
            Type: "object",
            Properties: map[string]Property{
                "question":    {Type: "string", Description: "Question to ask"},
                "session_id":  {Type: "string", Description: "Session ID"},
                "notebook_id": {Type: "string", Description: "Notebook ID"},
            },
            Required: []string{"question"},
        },
    },
    // ... more tools
}

// Tool profiles
var Profiles = map[string][]string{
    "minimal": {
        "ask_question",
        "list_notebooks",
        "select_notebook",
        "get_health",
        "setup_auth",
    },
    "standard": {
        // minimal + notebook management
        "add_notebook",
        "update_notebook",
        "remove_notebook",
        "search_notebooks",
        "get_library_stats",
    },
    "full": {
        // standard + session + system
        "list_sessions",
        "close_session",
        "reset_session",
        "re_auth",
        "cleanup_data",
    },
}
```

### Server Implementation

```go
package mcp

type Server struct {
    browser  *browser.ContextManager
    auth     *auth.Manager
    library  *library.Library
    sessions *session.Manager
    tools    map[string]ToolHandler
}

func (s *Server) Start() error {
    // Read from stdin, write to stdout (stdio transport)
    scanner := bufio.NewScanner(os.Stdin)

    for scanner.Scan() {
        var request Request
        json.Unmarshal(scanner.Bytes(), &request)

        response := s.handleRequest(request)
        json.NewEncoder(os.Stdout).Encode(response)
    }

    return nil
}

func (s *Server) handleToolCall(name string, args map[string]any) (any, error) {
    handler, ok := s.tools[name]
    if !ok {
        return nil, fmt.Errorf("unknown tool: %s", name)
    }
    return handler(args)
}
```

## Build & Distribution

### Makefile

```makefile
BINARY=notebooklm
VERSION=$(shell git describe --tags --always)
LDFLAGS=-ldflags "-X main.Version=${VERSION}"

.PHONY: build install test clean

build:
	go build ${LDFLAGS} -o bin/${BINARY} ./cmd/notebooklm

install:
	go install ${LDFLAGS} ./cmd/notebooklm

test:
	go test -v ./...

# Cross-compilation
build-all:
	GOOS=linux GOARCH=amd64 go build ${LDFLAGS} -o bin/${BINARY}-linux-amd64 ./cmd/notebooklm
	GOOS=darwin GOARCH=amd64 go build ${LDFLAGS} -o bin/${BINARY}-darwin-amd64 ./cmd/notebooklm
	GOOS=darwin GOARCH=arm64 go build ${LDFLAGS} -o bin/${BINARY}-darwin-arm64 ./cmd/notebooklm
	GOOS=windows GOARCH=amd64 go build ${LDFLAGS} -o bin/${BINARY}-windows-amd64.exe ./cmd/notebooklm

clean:
	rm -rf bin/
```

### Dependencies (go.mod)

```go
module github.com/user/notebooklm-cli

go 1.21

require (
    github.com/spf13/cobra v1.8.0
    github.com/spf13/viper v1.18.0
    github.com/chromedp/chromedp v0.9.5
    github.com/chromedp/cdproto v0.0.0-20240202021202-6d0b6a386732
    github.com/rs/zerolog v1.32.0
    github.com/google/uuid v1.6.0
)
```

## Usage Examples

### Interactive Session (Sticky Context)

```bash
# Setup authentication
$ notebooklm auth setup --show-browser
Opening browser for Google login...
Login successful! Authentication saved.

# Add a notebook
$ notebooklm notebook add \
  --url "https://notebooklm.google.com/notebook/abc123" \
  --name "React Documentation" \
  --topics "React,Hooks,Components,State"
Added notebook: react-documentation (id: react-doc-abc)

# Set active notebook (sticky)
$ notebooklm notebook select react-doc-abc
Active notebook set to: React Documentation

# First question - creates new session
$ notebooklm ask "What is useState and how do I use it?"
useState is a React Hook that lets you add state to functional components...
[Session: auto-7f3k2m]

# Set this session as active (sticky)
$ notebooklm session select auto-7f3k2m
Active session set to: auto-7f3k2m

# Now all questions use active notebook + active session automatically
$ notebooklm ask "How does it compare to useReducer?"
While useState is simpler for individual state values, useReducer is better for...
[Session: auto-7f3k2m, Messages: 2]

$ notebooklm ask "Show me an example with both"
Here's an example combining useState and useReducer...
[Session: auto-7f3k2m, Messages: 3]

# Check current context
$ notebooklm session current
Active notebook: React Documentation (react-doc-abc)
Active session: auto-7f3k2m (3 messages)

# Force new session (ignore sticky)
$ notebooklm ask "What is useContext?" --new-session
useContext is a Hook that lets you subscribe to React context...
[Session: auto-9x8y1z]

# Switch to different notebook temporarily
$ notebooklm ask "How does Vue handle state?" --notebook vue-docs
Vue uses reactive data properties and the Composition API...
[Session: auto-abc123]

# Output as JSON
$ notebooklm ask "List all hooks" --format json
{
  "status": "success",
  "question": "List all hooks",
  "answer": "React provides several built-in hooks...",
  "session_id": "auto-7f3k2m",
  "notebook_id": "react-doc-abc",
  "message_count": 4
}
```

### MCP Server Mode

```bash
# Start as MCP server (for Claude Desktop, etc.)
$ notebooklm serve --profile full

# In claude_desktop_config.json:
{
  "mcpServers": {
    "notebooklm": {
      "command": "notebooklm",
      "args": ["serve", "--profile", "standard"]
    }
  }
}
```

### Scripting / Automation

```bash
#!/bin/bash
# Batch query script

NOTEBOOK_ID="api-docs"
QUESTIONS=(
    "What authentication methods are supported?"
    "How do I handle rate limiting?"
    "What are the error response formats?"
)

for q in "${QUESTIONS[@]}"; do
    echo "Q: $q"
    notebooklm ask "$q" --notebook $NOTEBOOK_ID --format markdown
    echo "---"
done
```

## Implementation Phases

### Phase 1: Core CLI (MVP)
- [ ] Project setup with Cobra
- [ ] Basic chromedp integration
- [ ] `ask` command with direct URL
- [ ] Auth setup/status commands
- [ ] Config management

### Phase 2: Library Management
- [ ] Notebook library (JSON storage)
- [ ] Add/list/select/remove notebooks
- [ ] Search functionality
- [ ] Session management

### Phase 3: MCP Server
- [ ] MCP protocol implementation
- [ ] Tool definitions
- [ ] Tool handlers
- [ ] Tool profiles

### Phase 4: Polish
- [ ] Stealth improvements
- [ ] Error handling
- [ ] Logging
- [ ] Cross-platform testing
- [ ] Documentation

## Key Differences from Node.js Version

| Aspect | Node.js (Patchright) | Golang (chromedp) |
|--------|---------------------|-------------------|
| Browser | Patchright (Playwright fork) | chromedp (CDP) |
| Async | Promises/async-await | Goroutines/channels |
| Distribution | npm package | Single binary |
| Config | dotenv + env-paths | Viper + XDG |
| Stealth | patchright-stealth | Manual implementation |

## Considerations

### Stealth / Bot Detection
chromedp lacks built-in stealth features. Implement:
- Custom user agent
- Disable automation flags
- Human-like typing delays
- Random mouse movements
- Persistent browser profile

### Error Recovery
- Browser crash: Recreate context
- Auth expiry: Auto-relogin
- Network errors: Retry with backoff
- Rate limits: Detect and warn user

### Resource Management
- Single browser instance
- Shared context across sessions
- Graceful shutdown (close all tabs)
- Memory monitoring
