# Unblocked Chrome for Claude

A Chrome extension + MCP server that gives Claude Code full browser automation — identical to [Claude in Chrome](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn), but without domain restrictions. Also supports Brave Browser.

## Architecture

```
Claude Code ←stdio MCP→ mcp-server.js ←TCP→ native-host.js ←native messaging→ Chrome Extension
```

Three components:
1. **Chrome Extension** — Manifest V3 extension with CDP-based browser automation
2. **MCP Server** — Node.js process started by Claude Code, exposes 18 tools via MCP
3. **Native Messaging Host** — Bridge between the MCP server and the extension

## Prerequisites

- **Node.js** v18+
- **Google Chrome**, **Microsoft Edge**, or **Brave Browser**
- **Claude Code** v2.0.73+

## Installation

### Step 1: Install npm dependencies

```bash
cd host
npm install
cd ..
```

### Step 2: Load the extension in your browser

1. Open your browser and go to `chrome://extensions` (or `brave://extensions` / `edge://extensions`)
2. Enable **Developer mode** (toggle in the top right)
3. Click **Load unpacked**
4. Select the `extension/` directory from this project
5. The extension will appear with a name like "Unblocked Chrome for Claude"
6. **Copy the extension ID** shown under the extension name (a long string like `abcdefghijklmnop...`)

### Step 3: Run the install script

```bash
./install.sh <your-extension-id>
```

This registers the native messaging host for Chrome, Edge, and Brave. It creates:
- A wrapper script at `host/native-host-wrapper.sh`
- Native messaging host manifests in each browser's `NativeMessagingHosts/` directory

### Step 4: Restart your browser

Close **all** browser windows and reopen. Chrome reads native messaging host configs on startup.

### Step 5: Add the MCP server to Claude Code

```bash
claude mcp add unblocked-chrome -- node /absolute/path/to/host/mcp-server.js
```

Use the **absolute path** to `mcp-server.js`. You can find it with:

```bash
echo "node $(pwd)/host/mcp-server.js"
```

## Verification

Start a new Claude Code session and test:

```
Navigate to reddit.com and take a screenshot
```

If everything is working, Claude will:
1. Call `tabs_context_mcp` to get available tabs
2. Call `tabs_create_mcp` to create a new tab
3. Call `navigate` to go to reddit.com
4. Call `computer` with action `screenshot`

Reddit should load (no domain restriction).

## Available Tools

All 18 tools match the Claude in Chrome API surface:

| Tool | Description |
|------|-------------|
| `tabs_context_mcp` | Get tab group context |
| `tabs_create_mcp` | Create new tab |
| `navigate` | Navigate to URL, back, forward |
| `computer` | Mouse, keyboard, screenshots (13 actions) |
| `read_page` | Accessibility tree with element refs |
| `get_page_text` | Extract article/main text |
| `find` | Find elements by text/attributes |
| `form_input` | Set form values by ref |
| `javascript_tool` | Execute JS in page context |
| `read_console_messages` | Console output (filtered) |
| `read_network_requests` | Network activity |
| `resize_window` | Resize browser window |
| `upload_image` | Upload screenshot to file input |
| `gif_creator` | GIF recording (stub) |
| `shortcuts_list` | List shortcuts (stub) |
| `shortcuts_execute` | Run shortcut (stub) |
| `switch_browser` | Switch browser (stub) |
| `update_plan` | Present plan (auto-approved) |

## Differences from Claude in Chrome

1. **No domain blocklist** — navigate to any URL
2. **Brave Browser support** — native messaging registered for Brave
3. **`find` tool** — uses text/attribute matching instead of nested LLM call
4. **`gif_creator`** — stub (not implemented)
5. **`shortcuts_*`** — stub (not implemented)
6. **`update_plan`** — auto-approves (no permission dialog)

## Troubleshooting

### Extension not connecting

1. Verify the extension is loaded and enabled in `chrome://extensions`
2. Check that you ran `./install.sh` with the correct extension ID
3. Restart the browser completely (all windows)
4. Check the native messaging host manifest exists:
   - **Chrome (macOS)**: `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/com.anthropic.unblocked_chrome.json`
   - **Brave (macOS)**: `~/Library/Application Support/BraveSoftware/Brave-Browser/NativeMessagingHosts/com.anthropic.unblocked_chrome.json`
   - **Edge (macOS)**: `~/Library/Application Support/Microsoft Edge/NativeMessagingHosts/com.anthropic.unblocked_chrome.json`

### MCP server not found

Make sure you used an absolute path when adding the MCP:
```bash
claude mcp add unblocked-chrome -- node /absolute/path/to/host/mcp-server.js
```

### "Browser extension is not connected" error

This means the MCP server started but the native host hasn't connected yet. The native host is launched by Chrome when the extension's service worker starts. Try:
1. Open any webpage in the browser (this wakes the service worker)
2. Check the extension's service worker logs in `chrome://extensions` → "Inspect views: service worker"
3. Verify the native host wrapper exists: `host/native-host-wrapper.sh`

### Tools timing out

The default timeout is 60 seconds. For long operations (slow page loads, complex screenshots), this may not be enough. The MCP server will return a timeout error — retry the operation.

### Port conflict

The MCP server and native host communicate on TCP port 18765. If this conflicts:
1. Create `~/.config/unblocked-chrome/config.json`:
   ```json
   { "port": 19000 }
   ```
2. Restart both the browser and Claude Code

## How It Works

1. **Claude Code** starts `mcp-server.js` as a stdio MCP server
2. The MCP server opens a TCP listener on `127.0.0.1:18765`
3. The **Chrome extension**'s service worker calls `chrome.runtime.connectNative()`, which launches `native-host.js`
4. The native host connects to the MCP server's TCP port
5. When Claude sends a tool call, it flows: Claude Code → MCP (stdio) → TCP → native messaging → extension
6. The extension executes the tool using Chrome APIs (CDP for mouse/keyboard/screenshots, content scripts for DOM)
7. Results flow back the same path
