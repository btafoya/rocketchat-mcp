# rocketchat-mcp — Project Details

An [MCP](https://modelcontextprotocol.io) server for [Rocket.Chat](https://rocket.chat).
It exposes a stdio-based MCP server that lets MCP clients send/read messages,
manage DMs and channels, search history, and upload/download files against a
Rocket.Chat workspace.

This repository is an actively maintained fork of
[elieworkspace/rocketchat-mcp](https://github.com/elieworkspace/rocketchat-mcp),
adding Personal Access Token auth, more tools, reliability improvements, and a
PyPI/MCP Registry publishing pipeline. See [NOTICE](NOTICE) for attribution.

## Repository layout

| File | Purpose |
| --- | --- |
| `rocketchat.py` | Entire MCP server implementation (client, tools, lifecycle) |
| `pyproject.toml` | Package metadata, dependencies, build config, console entry point |
| `uv.lock` | Locked dependency tree for reproducible installs |
| `server.json` | MCP Registry manifest (stdio transport, env vars, version) |
| `glama.json` | Glama MCP server listing metadata |
| `docker-compose.yaml` | Local Rocket.Chat + MongoDB stack for development/testing |
| `.github/workflows/publish.yml` | Release build, PyPI publish, and MCP Registry publish |
| `README.md` | User-facing quick-start and tool reference |
| `NOTICE` / `LICENSE` | Fork attribution and MIT license for modifications |

## Architecture

The project is deliberately single-file: all server behavior lives in
`rocketchat.py`. The design layers are:

1. **HTTP client layer** — `RocketChatAPI` wraps `httpx.AsyncClient` and
   provides async JSON requests to `/api/v1/*` endpoints.
2. **Room resolution layer** — helpers resolve room names to IDs and discover
   whether a room uses `channels.messages`, `groups.messages`, or `im.messages`,
   caching results across calls.
3. **MCP server layer** — a `FastMCP("rocketchat")` instance registers async
   tool functions with the `@mcp.tool()` decorator.
4. **Lifecycle layer** — `main()` parses CLI/env arguments, initializes the
   client, and starts the stdio transport. An orphan watchdog exits the
   process if its parent (or grandparent) becomes PID 1.

```
┌─────────────────────────────────────────────┐
│  MCP client (Claude Desktop, Cursor, etc.)   │
│  stdio transport                             │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│  FastMCP server (`mcp` in rocketchat.py)     │
│  ── @mcp.tool() handlers                     │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│  RocketChatAPI                               │
│  ── async_request, resolve_room_id,        │
│     fetch_room_messages, _get_client         │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│  Rocket.Chat REST API v1                   │
└─────────────────────────────────────────────┘
```

## Key components

### `RocketChatAPI` (`rocketchat.py:44`)

The low-level API client. It supports two authentication modes:

- **Personal Access Token** (preferred): pass `user_id` + `auth_token`.
- **Username / password**: performs a `POST /api/v1/login` at startup.

State:

- `_client` — lazily created persistent `httpx.AsyncClient` with connection
  pooling and a 30 s default timeout.
- `_room_endpoint` — `roomId -> endpoint` cache so repeated reads use the
  already-discovered endpoint (`channels.messages`, `groups.messages`, or
  `im.messages`).
- `_room_id_cache` — `roomId/name -> roomId` cache; many read/search endpoints
  only accept a raw room ID while post endpoints accept names.

Methods:

- `async_request(method, endpoint, json_data=None, params=None)` — generic
  JSON API call with `X-Auth-Token` / `X-User-Id` headers.
- `resolve_room_id(room)` — probes `rooms.info?roomId=` and `roomName=`; falls
  back to returning the input unchanged so downstream endpoints report the
  real error.
- `fetch_room_messages(room, params)` — resolves the room, tries cached then
  candidate endpoints, and caches the working endpoint.
- `_headers()` / `_auth_headers()` — JSON vs. multipart auth headers.
- `_get_client()` — returns a reusable async HTTP client.

### Tool handlers

All handlers are async functions decorated with `@mcp.tool()`. They guard on
`rocket_client` being initialized, log calls, and return plain-string results.

| Tool | Rocket.Chat endpoint(s) | Notes |
| --- | --- | --- |
| `send_message_in_channel` | `chat.postMessage` | Accepts channel name or ID |
| `send_direct_message` | `chat.postMessage` | Prefixes recipient with `@` |
| `delete_message` | `chat.delete` | Requires `room_id` + `msg_id` |
| `get_channel_messages` | `*.messages` via `fetch_room_messages` | `count` capped at 100; supports `offset` |
| `search_messages` | `chat.search` | Resolves room ID first |
| `get_unread` | `subscriptions.get` + `*.messages` | Lists rooms with unread counts and latest messages |
| `send_file` | `im.create` (for `@user`) + `rooms.upload/{room_id}` | Uses multipart upload; 60 s timeout |
| `download_attachment` | `chat.getMessage` + `/file-upload/{file_id}/{file_name}` | Writes to system temp dir |
| `list_all_rooms` | `channels.list`, `groups.list`, `im.list` | Requires corresponding `view-*` permission |
| `list_users` | `users.list` | Requires admin or `view-*` permission |
| `get_user_info` | `users.info` | |
| `create_channel` | `channels.create` | |

### Message formatting

`format_message_line(msg, indent="")` (`rocketchat.py:184`) renders a single
message as:

```
[timestamp] (id: <msg_id>) <username>: <text>
  💬 <attachment description>
  📎 <filename> (<type>) [use download_attachment with id: <msg_id>]
```

It defensively handles missing or `None` fields so one malformed message does
not crash a page of history.

## Authentication & configuration

### Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `ROCKETCHAT_SERVER_URL` | yes | Workspace URL, e.g. `https://chat.example.com` |
| `ROCKETCHAT_USER_ID` | yes* | Rocket.Chat user ID |
| `ROCKETCHAT_AUTH_TOKEN` | yes* | Personal Access Token |
| `ROCKETCHAT_WATCHDOG_INTERVAL` | no | Orphan-watchdog interval in seconds; default `60` |

*Required for token auth; alternatively use `--username` / `--password`.

### CLI arguments

```
--server-url      Server URL (also env var)
--username        Username for password auth
--password        Password for password auth
--auth-token      Personal access token (also env var)
--user-id         User ID (also env var)
--no-verify-ssl   Disable SSL verification for self-signed certs
```

Recommended invocation (keeps secrets out of process arguments):

```bash
uvx rocketchat-mcp --server-url https://chat.example.com
```

## Lifecycle & reliability

- **Persistent HTTP client**: `_get_client()` reuses an `httpx.AsyncClient`
  across requests for connection pooling.
- **Room caching**: avoids re-probing endpoints and re-resolving names on every
  read.
- **Log rotation**: `RotatingFileHandler(maxBytes=1_000_000, backupCount=1)`
  writes to `rocketchat_mcp.log` in the package directory, plus stderr.
- **Orphan watchdog**: `start_orphan_watchdog()` runs a daemon thread that
  self-exits when `ppid == 1` or when the parent’s parent became PID 1.
  Prevents zombie MCP servers when the client process dies abnormally (POSIX
  only).

## Dependencies

Declared in `pyproject.toml`:

- `httpx>=0.28.1` — async HTTP client
- `mcp[cli]>=1.12.3` — FastMCP server framework

Requires Python `>=3.10`. Package manager: `uv` (lock file present).

## Distribution

- **Console script**: `rocketchat-mcp = "rocketchat:main"`.
- **Build backend**: `hatchling`.
- **Wheel contents**: only `rocketchat.py` is included.
- **Publish workflow** (`.github/workflows/publish.yml`):
  1. `uv build` produces sdist + wheel.
  2. `pypa/gh-action-pypi-publish` publishes to PyPI via Trusted Publishing.
  3. `mcp-publisher` publishes `server.json` to the MCP Registry using GitHub
     OIDC.

## Development stack

`docker-compose.yaml` spins up a local Rocket.Chat instance for manual
integration testing:

- `rocketchat` service on `127.0.0.1:3000`
- `mongodb` service as a single-node replica set `rs0`

## MCP Registry manifest

`server.json` registers the package under the identifier
`io.github.millerchou/rocketchat-mcp` with stdio transport and the three
required environment variables `ROCKETCHAT_SERVER_URL`,
`ROCKETCHAT_USER_ID`, and `ROCKETCHAT_AUTH_TOKEN` (the latter two marked as
secrets).

## License

Modifications in this fork are released under the MIT License (see `LICENSE`).
Upstream code is used under the terms described in `NOTICE`.
