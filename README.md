# pin-mcp

Write a [Model Context Protocol](https://modelcontextprotocol.io) server in [Pin](https://github.com/pin-language/pin):
one small native binary that Claude Code, Claude Desktop or any MCP client starts and talks to over
standard input/output.

```bash
pin get github.com/pin-language/mcp@v0.1.0
```

```php
import github.com/pin-language/mcp/mcp;

function tools(): list<mcp::Tool> {
    $sq = new mcp::Tool("squares", "The squares of 1 to n");
    $sq->param("n", "integer", "how many squares (1 to 1000)", true);
    return [$sq];
}

while (true) {
    $r = try mcp::next() catch ($e) { break; };                 // the client closed the pipe
    if (mcp::basic($r, "squares", "1.0.0")) { continue; }      // initialize, ping, notifications
    if ($r->method == "tools/list") {
        mcp::reply_tools($r, tools());
    } elseif ($r->method == "tools/call" && mcp::tool_name($r) == "squares") {
        $n = json_int(mcp::arguments($r), "n");
        list<int> $out = [];
        for ($i = 1; $i <= $n; $i++) { $out[] = $i * $i; }
        mcp::reply_text($r, json_encode($out));
    } else {
        mcp::reply_error($r, -32601, "unknown method " . $r->method);
    }
}
```

```bash
pin build server.pn -o squares-mcp
claude mcp add squares /full/path/to/squares-mcp     # Claude Code
```

`example.pn` is a complete server with two tools and argument checks.

## What it gives you

| | |
|---|---|
| `mcp::next()` | the next message (`$r->method`, `$r->id`, `$r->params`); fails at the end of input |
| `mcp::basic($r, $name, $version)` | answers `initialize`, `ping` and notifications; `true` when it did |
| `mcp::Tool` + `->param($name, $type, $description, $required)` | a tool and its input schema |
| `mcp::reply_tools($r, $tools)` | the answer to `tools/list` |
| `mcp::tool_name($r)`, `mcp::arguments($r)` | what a `tools/call` asks for; read the arguments with `json_int`, `json_string`, `json_raw`, … |
| `mcp::reply_text($r, $text)` / `mcp::reply_tool_error($r, $text)` | a tool's result, or a failure the model can see and fix |
| `mcp::reply_error($r, $code, $message)` | a protocol error (`-32601` unknown method, `-32602` bad params) |

Ids are sent back exactly as they came (numbers or strings). Tested with the official MCP Python SDK
(initialize, tools/list, tools/call, errors, ping).

Not yet: resources, prompts, the HTTP transport, sending notifications to the client. Needs Pin 0.8 or later
(`read_line`, `json_raw`).

MIT licence, © PinaTo Co., Ltd.
