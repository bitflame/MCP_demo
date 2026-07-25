# Weather MCP Server

This is an implementation of an MCP server composed of two tools, `get_alerts()` and
`get_forecast()`, which Claude Desktop consumes to answer a user's questions about
weather forecasts and alerts using data from the National Weather Service (NWS).

## What is MCP?

MCP (Model Context Protocol) is a standard way to connect an AI application to outside
tools and data sources — things like a weather API, a database, or a browser.

Before MCP, if you wanted an AI app to use, say, 5 different tools, you'd need to write
custom integration code for each tool, for each AI app. If 3 different AI apps all
wanted to use those same 5 tools, that's potentially 3 × 5 = 15 separate integrations
to build and maintain. This is sometimes called the "M×N problem" — M apps times N
tools means M×N integrations.

MCP fixes this by defining one standard protocol. A tool author builds one MCP server;
any MCP-compatible app can then use it without custom glue code. This turns the problem
from M×N integrations into roughly M+N: each tool is built once, each app supports the
protocol once, and everything after that just plugs together.

MCP itself is just the *shape* of the conversation — the message format and rules for
how a host, client, and server talk to each other. It doesn't enforce security policy
on its own; that's up to whoever builds the server. What it does provide is a clear
boundary: a server only exposes the specific tools it chooses to expose, so access
control can be built in at that boundary.

## The three roles: host, client, server

- **Host** — the user-facing application. In this project, that's Claude Desktop. The
  host manages the conversation with the AI model and decides when to call a tool
  based on what the user asked.
- **Client** — lives *inside* the host, not as a separate app. There's one client
  instance per connected server, and each client handles the actual protocol-level
  conversation with that one server. If Claude Desktop is connected to 3 servers, it's
  running 3 client instances internally.
- **Server** — the external process that does the actual work and exposes a set of
  **tools** (functions the AI model can call). A server's capabilities are defined by
  the tools it offers. For example, the Playwright MCP server (built by Microsoft)
  exposes a `browser_click` tool that lets an AI model click on links in a browser to
  do things like automated web testing.

MCP was created and open-sourced by Anthropic in November 2024 specifically to solve
the M×N integration problem described above.

## This project

The code here is based on the official MCP quickstart guide
(https://modelcontextprotocol.io/quickstart) and uses the National Weather Service API
to build a working server. The host in this setup is Claude Desktop, configured to
launch and connect to this server. The server announces itself under the name
`weather`, which is what Claude Desktop looks for when deciding to route a
weather-related question here.

### Tools exposed

| Tool | Parameters | Description |
|---|---|---|
| `get_alerts` | `state: str` (two-letter US state code, e.g. `CA`, `NY`) | Returns active NWS weather alerts for a US state |
| `get_forecast` | `latitude: float`, `longitude: float` | Returns a 5-period forecast for a location |

## Running it

This project uses [`uv`](https://astral.sh/uv), a fast Python package and project
manager, instead of the traditional `pip`/`venv` combo.

**1. Install `uv` (Windows PowerShell):**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**2. Set up the project:**
```powershell
uv init weather
cd weather
uv venv
.venv\Scripts\activate
uv add "mcp[cli]" httpx
```

**3. Add `weather.py`** (server code in this repo).

**4. Connect it to Claude Desktop** by adding this to your Claude Desktop config - make sure to save a backup copy
(accessible via Settings → Developer → Edit Config):
```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory",
        "C:\\path\\to\\your\\weather",
        "run",
        "weather.py"
      ]
    }
  }
}
```

Fully quit Claude Desktop (from the system tray, not just closing the window) and
relaunch. Look for the tools/hammer icon in the chat input to confirm it connected.

**5. Test it** — ask something specific enough that Claude can't answer from its own
built-in knowledge, e.g. "Are there any active weather alerts in Texas?" A generic
"what's the weather" question may get answered by Claude Desktop's own built-in
weather widget instead of this server, since both can plausibly answer that question.

## Bugs I hit

The most significant bug: my `main()` function looked like this —

```python
def main():
    mcp.run(transport="stdio")
    if __name__ == "__main__":
        main()
```

The `if __name__ == "__main__":` check was indented *inside* `main()`, so it was dead
code that never executed — nothing at the module level ever called `main()`. The
script would register its tools on import (which is why I'd see a
"Tool already exists" warning on a second load) and then just exit immediately,
without ever starting the actual server loop.

This meant the script "ran" without erroring, which made it non-obvious at first —
running `uv run weather.py` returned to the prompt instantly instead of hanging, which
in retrospect was the tell that the server was never actually starting. It also
explained Claude Desktop's "could not attach to MCP server" error: it launched a
process that exited almost immediately instead of staying alive to listen on stdio.

The fix was moving the `if __name__ == "__main__":` block out to module level:
```python
def main():
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

After the fix, running the script manually correctly hangs (waiting on stdio input),
which is the expected behavior for a long-running MCP server.