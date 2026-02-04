# OpenClaw Gateway Architecture

## High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL CLIENTS                                │
├──────────────┬──────────────┬──────────────┬──────────────┬─────────────────┤
│   CLI Tool   │  macOS App   │   Web UI     │ iOS/Android  │  Remote Clients │
│  (operator)  │  (operator   │  (operator)  │    (node)    │   (via tunnel)  │
│              │   + node)    │              │              │                 │
└──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┴─────────┬───────┘
       │              │              │              │                 │
       │              │              │              │                 │
       └──────────────┴──────────────┴──────────────┴─────────────────┘
                                     │
                            WebSocket Protocol
                       (role: operator | node)
                    (auth: token | password | device)
                                     │
╔═══════════════════════════════════════════════════════════════════════════════╗
║                          GATEWAY SERVER (Port 18789)                          ║
║                         src/gateway/server.impl.ts                            ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                               ║
║  ┌─────────────────────────────────────────────────────────────────────┐    ║
║  │                    HTTP/HTTPS + WebSocket Server                     │    ║
║  │                      server-http.ts, server-ws-runtime.ts            │    ║
║  ├─────────────────────────────────────────────────────────────────────┤    ║
║  │  • WebSocket Upgrade (primary control plane)                        │    ║
║  │  • HTTP Endpoints: /hooks, /control, /v1/chat/completions,          │    ║
║  │    /v1/responses, /tools/invoke, /canvas/*                          │    ║
║  │  • TLS Support (optional)                                           │    ║
║  └─────────────────────────────────────────────────────────────────────┘    ║
║                                     │                                        ║
║  ┌──────────────────────────────────┴──────────────────────────────────┐    ║
║  │                     Protocol Layer & Auth                            │    ║
║  │                  protocol/schema/*.ts, auth.ts                       │    ║
║  ├──────────────────────────────────────────────────────────────────────┤    ║
║  │  • TypeBox schemas + JSON Schema validation                         │    ║
║  │  • Request/Response/Event frames                                    │    ║
║  │  • Role-based authorization (operator scopes, node caps)            │    ║
║  │  • Device pairing + token management                                │    ║
║  └──────────────────────────────────────────────────────────────────────┘    ║
║                                     │                                        ║
║  ┌──────────────────────────────────┴──────────────────────────────────┐    ║
║  │                  RPC Method Router (27+ methods)                     │    ║
║  │                    server-methods.ts, server-methods/*               │    ║
║  └──┬───────┬────────┬────────┬────────┬────────┬────────┬──────────┬──┘    ║
║     │       │        │        │        │        │        │          │       ║
║  ┌──▼──┐ ┌─▼───┐ ┌──▼───┐ ┌──▼───┐ ┌──▼───┐ ┌──▼───┐ ┌──▼────┐ ┌───▼───┐  ║
║  │Agent│ │Chat │ │Channels│ │Config│ │Cron │ │Devices│ │Nodes │ │Health │  ║
║  │     │ │     │ │       │ │      │ │     │ │      │ │      │ │       │  ║
║  └──┬──┘ └─┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬────┘ └───┬───┘  ║
║     │      │        │        │        │        │        │          │       ║
║  ┌──▼──────▼────────▼────────▼────────▼────────▼────────▼──────────▼────┐  ║
║  │                      Runtime State & Services                         │  ║
║  ├────────────────────────────────────────────────────────────────────────┤  ║
║  │                                                                        │  ║
║  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │  ║
║  │  │  Agent Runtime   │   │ Channel Manager  │   │  Node Registry   │  │  ║
║  │  │  server-chat.ts  │   │server-channels.ts│   │node-registry.ts  │  │  ║
║  │  ├──────────────────┤   ├──────────────────┤   ├──────────────────┤  │  ║
║  │  │• Agent SDK exec  │   │• Channel lifecycle│   │• Device pairing │  │  ║
║  │  │• Tool streaming  │   │• Multi-account   │   │• Capability route│  │  ║
║  │  │• Chat run state  │   │• Plugin docking  │   │• Command invoke │  │  ║
║  │  │• Abort control   │   │• Auto-restart    │   │• Permissions    │  │  ║
║  │  └──────────────────┘   └────────┬─────────┘   └──────────────────┘  │  ║
║  │                                   │                                    │  ║
║  │  ┌──────────────────┐   ┌─────────▼─────────┐   ┌──────────────────┐  │  ║
║  │  │ Cron Scheduler   │   │  Channel Plugins  │   │  Discovery Svc   │  │  ║
║  │  │ server-cron.ts   │   │ (WhatsApp, Tele-  │   │server-discovery  │  │  ║
║  │  ├──────────────────┤   │  gram, Discord,   │   │  -runtime.ts     │  │  ║
║  │  │• Scheduled jobs  │   │  Slack, Signal,   │   ├──────────────────┤  │  ║
║  │  │• Run history     │   │  iMessage, etc.)  │   │• Bonjour/mDNS   │  │  ║
║  │  │• Wake on schedule│   │                   │   │• DNS-SD         │  │  ║
║  │  └──────────────────┘   └───────────────────┘   │• Tailscale Serve│  │  ║
║  │                                                  └──────────────────┘  │  ║
║  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │  ║
║  │  │  Hooks Manager   │   │  Config Reloader │   │  Broadcast Mgr   │  │  ║
║  │  │    hooks.ts      │   │config-reload.ts  │   │server-broadcast  │  │  ║
║  │  ├──────────────────┤   ├──────────────────┤   │     .ts          │  │  ║
║  │  │• Wake hooks      │   │• File watcher    │   ├──────────────────┤  │  ║
║  │  │• Agent hooks     │   │• Hot-reload      │   │• Event streaming │  │  ║
║  │  │• HTTP→agent map  │   │• SIGUSR1 restart │   │• Client fanout   │  │  ║
║  │  └──────────────────┘   └──────────────────┘   │• Seq numbering   │  │  ║
║  │                                                  └──────────────────┘  │  ║
║  │  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐  │  ║
║  │  │  Session Store   │   │  Plugin Loader   │   │  Lane Controller │  │  ║
║  │  │ (agent sessions) │   │server-plugins.ts │   │ server-lanes.ts  │  │  ║
║  │  ├──────────────────┤   ├──────────────────┤   ├──────────────────┤  │  ║
║  │  │• Session keys    │   │• Plugin discovery│   │• Concurrency lim │  │  ║
║  │  │• Message history │   │• Method registry │   │• Queue per lane  │  │  ║
║  │  │• Subagent links  │   │• HTTP handlers   │   │• Throttling      │  │  ║
║  │  └──────────────────┘   └──────────────────┘   └──────────────────┘  │  ║
║  │                                                                        │  ║
║  └────────────────────────────────────────────────────────────────────────┘  ║
║                                                                               ║
╚═══════════════════════════════════════════════════════════════════════════════╝
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  File Storage     │     │  Messaging Platforms │     │  External APIs   │
├───────────────────┤     ├──────────────────────┤     ├──────────────────┤
│ ~/.openclaw/      │     │ • WhatsApp (Baileys) │     │ • Anthropic API  │
│                   │     │ • Telegram (grammY)  │     │ • OpenAI API     │
│ • openclaw.json   │     │ • Discord            │     │ • Provider APIs  │
│ • credentials/    │     │ • Slack              │     │                  │
│ • agents/*/       │     │ • Signal             │     │                  │
│   sessions/*.jsonl│     │ • iMessage           │     │                  │
│ • devices/        │     │ • BlueBubbles        │     │                  │
│ • pairing/        │     │ • Matrix (plugin)    │     │                  │
│ • sessions.json   │     │ • MS Teams (plugin)  │     │                  │
└───────────────────┘     └──────────────────────┘     └──────────────────┘
```

## Message Flow Example

```
┌─────────────┐
│ WhatsApp    │  User sends message
│   User      │
└──────┬──────┘
       │
       ▼
┌──────────────┐
│ Baileys      │  Channel plugin receives message
│  (Channel)   │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ Channel Manager      │  Routes to gateway
│ server-channels.ts   │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Routing Layer        │  Determines session key
│ src/routing/         │  (agent:agentId:sessionId)
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Agent Runtime        │  Executes agent turn
│ server-chat.ts       │  • Tool calls
└──────┬───────────────┘  • Streaming deltas
       │                  • Result generation
       ▼
┌──────────────────────┐
│ Broadcast Manager    │  Streams events to clients
│ server-broadcast.ts  │
└──────┬───────────────┘
       │
       ├────────────────────┬────────────────────┬─────────────────┐
       ▼                    ▼                    ▼                 ▼
  ┌─────────┐         ┌──────────┐        ┌──────────┐      ┌──────────┐
  │ macOS   │         │ Web UI   │        │ CLI Tool │      │ Channel  │
  │  App    │         │          │        │          │      │ (reply)  │
  └─────────┘         └──────────┘        └──────────┘      └────┬─────┘
                                                                  │
                                                                  ▼
                                                            ┌──────────────┐
                                                            │ WhatsApp     │
                                                            │   User       │
                                                            └──────────────┘
```

## Node Command Flow

```
┌──────────────┐
│ CLI/Web UI   │  operator requests: "take a screenshot"
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ Agent Runtime        │  agent.tool_use → screen.record
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Node Registry        │  finds node with screen capability
│ node-registry.ts     │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ node.invoke RPC      │  sends command over WebSocket
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ iOS/Android/macOS    │  executes screen capture
│   Node Client        │  (native capability)
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ node.invoke.result   │  returns image/video bytes
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Agent Runtime        │  processes result, continues turn
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Response to User     │
└────────────────────────┘
```

## Configuration & Lifecycle

```
┌─────────────────────┐
│  Config File Watch  │  ~/.openclaw/openclaw.json changes
│  config-reload.ts   │
└──────┬──────────────┘
       │
       ▼
┌──────────────────────┐
│  Classify Change     │  safe vs critical
└──────┬───────────────┘
       │
       ├───────────────────────┬────────────────────────┐
       ▼                       ▼                        ▼
┌──────────────┐      ┌─────────────────┐     ┌────────────────┐
│ Hot-Apply    │      │ SIGUSR1 Restart │     │ Exit & Restart │
│ (model cfg,  │      │ (channel cfg,   │     │ (port, bind,   │
│  aliases)    │      │  auth changes)  │     │  TLS)          │
└──────────────┘      └─────────────────┘     └────────────────┘
```

## Key Architectural Principles

1. **Single Port, Multiple Services**: All HTTP/WS on one port → simplifies tunneling
2. **Role-Based Protocol**: Operator vs Node roles with distinct capabilities
3. **Event Streaming**: Sequence-numbered events, no replay (clients re-fetch on gap)
4. **Idempotency**: Dedupe cache ensures safe retries for side-effects
5. **Plugin Extensibility**: Channels, methods, HTTP handlers all pluggable
6. **Graceful Degradation**: Hot-reload, channel auto-restart, coordinated shutdown
7. **Device Trust Model**: Pairing + device tokens + local auto-approve
8. **Concurrency Control**: Lane-based throttling prevents resource exhaustion
9. **Type Safety**: TypeBox schemas → JSON Schema → validated at runtime
10. **Single Instance**: One gateway per ~/.openclaw directory (state ownership)
