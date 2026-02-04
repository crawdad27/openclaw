# Node Delegation Workflow: Adding Calendar Access

## Example: Native Calendar Integration (EventKit)

This guide walks through adding a new native capability to OpenClaw using **Calendar Access** as an example. We'll implement it for macOS/iOS using EventKit APIs.

---

## Step 0: Decision - Why Node Delegation?

**Calendar Access Requirements:**

- ✅ Requires native EventKit framework (Swift only)
- ✅ Needs calendar permissions from OS
- ✅ Platform-specific (macOS/iOS, not Android initially)
- ✅ Real-time calendar event creation/reading
- ❌ No external CLI tool exists that's good enough
- ❌ Can't be done in TypeScript/Node.js

**Decision:** Use **Node Delegation** pattern

---

## Architecture Overview

```
┌──────────────┐
│ Agent calls  │  nodes-tool: calendar_create_event
│  nodes tool  │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ Gateway receives     │  RPC: node.invoke
│   node.invoke        │  { command: "calendar.create", params: {...} }
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ NodeRegistry         │  Finds node with "calendar" cap
│  routes to device    │  Sends WebSocket event: "node.invoke.request"
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ macOS/iOS app        │  GatewayNodeSession receives event
│  receives command    │  Calls handleInvoke()
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ MacNodeCalendar      │  Native EventKit APIs
│  Service executes    │  Request permissions, create event
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Swift returns        │  BridgeInvokeResponse { ok, payloadJSON }
│   result             │  Sends RPC: node.invoke.result
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ NodeRegistry         │  Resolves pending invoke promise
│  returns to gateway  │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Agent receives       │  { eventId: "...", calendarName: "..." }
│   result             │
└────────────────────────┘
```

---

## Step 1: Define Protocol (TypeScript Types)

**File:** `src/gateway/node-protocol/calendar-commands.ts` (new file)

```typescript
// Command identifiers
export const CALENDAR_COMMANDS = ["calendar.create", "calendar.list", "calendar.search"] as const;

export type CalendarCommand = (typeof CALENDAR_COMMANDS)[number];

// calendar.create parameters
export interface CalendarCreateParams {
  title: string;
  notes?: string;
  startDate: string; // ISO8601
  endDate: string; // ISO8601
  calendarName?: string; // Which calendar (default: "Calendar")
  location?: string;
  url?: string;
  alarms?: CalendarAlarm[];
}

export interface CalendarAlarm {
  offsetMinutes: number; // -15 = 15 min before
}

// calendar.list parameters
export interface CalendarListParams {
  startDate?: string; // ISO8601
  endDate?: string; // ISO8601
  calendars?: string[]; // Filter by calendar names
  limit?: number;
}

// calendar.search parameters
export interface CalendarSearchParams {
  query: string;
  startDate?: string;
  endDate?: string;
  limit?: number;
}

// Response types
export interface CalendarEvent {
  eventId: string;
  title: string;
  notes?: string;
  startDate: string;
  endDate: string;
  calendarName: string;
  location?: string;
  url?: string;
}

export interface CalendarCreateResult {
  eventId: string;
  calendarName: string;
}

export interface CalendarListResult {
  events: CalendarEvent[];
}
```

**Update command policy:** `src/gateway/node-command-policy.ts`

```typescript
const CALENDAR_COMMANDS = [
  "calendar.create",
  "calendar.list",
  "calendar.search",
] as const;

const PLATFORM_DEFAULTS = {
  ios: [
    ...CANVAS_COMMANDS,
    ...CAMERA_COMMANDS,
    ...SCREEN_COMMANDS,
    ...LOCATION_COMMANDS,
    ...CALENDAR_COMMANDS,  // ← Add here
  ],
  android: [...],
  macos: [
    ...CANVAS_COMMANDS,
    ...CAMERA_COMMANDS,
    ...SCREEN_COMMANDS,
    ...LOCATION_COMMANDS,
    ...SYSTEM_COMMANDS,
    ...CALENDAR_COMMANDS,  // ← Add here
  ],
  // ...
};
```

---

## Step 2: Define Swift Protocol Types (Shared)

**File:** `apps/shared/OpenClawKit/Sources/OpenClawKit/CalendarCommands.swift` (new file)

```swift
import Foundation

// MARK: - Command Enum

public enum OpenClawCalendarCommand: String, Codable, Sendable {
    case create = "calendar.create"
    case list = "calendar.list"
    case search = "calendar.search"
}

// MARK: - calendar.create

public struct OpenClawCalendarCreateParams: Codable, Sendable {
    public var title: String
    public var notes: String?
    public var startDate: String  // ISO8601
    public var endDate: String    // ISO8601
    public var calendarName: String?
    public var location: String?
    public var url: String?
    public var alarms: [OpenClawCalendarAlarm]?

    public init(
        title: String,
        notes: String? = nil,
        startDate: String,
        endDate: String,
        calendarName: String? = nil,
        location: String? = nil,
        url: String? = nil,
        alarms: [OpenClawCalendarAlarm]? = nil
    ) {
        self.title = title
        self.notes = notes
        self.startDate = startDate
        self.endDate = endDate
        self.calendarName = calendarName
        self.location = location
        self.url = url
        self.alarms = alarms
    }
}

public struct OpenClawCalendarAlarm: Codable, Sendable {
    public var offsetMinutes: Int  // -15 = 15 min before

    public init(offsetMinutes: Int) {
        self.offsetMinutes = offsetMinutes
    }
}

public struct OpenClawCalendarCreateResult: Codable, Sendable {
    public var eventId: String
    public var calendarName: String

    public init(eventId: String, calendarName: String) {
        self.eventId = eventId
        self.calendarName = calendarName
    }
}

// MARK: - calendar.list

public struct OpenClawCalendarListParams: Codable, Sendable {
    public var startDate: String?  // ISO8601
    public var endDate: String?    // ISO8601
    public var calendars: [String]?
    public var limit: Int?

    public init(
        startDate: String? = nil,
        endDate: String? = nil,
        calendars: [String]? = nil,
        limit: Int? = nil
    ) {
        self.startDate = startDate
        self.endDate = endDate
        self.calendars = calendars
        self.limit = limit
    }
}

public struct OpenClawCalendarEvent: Codable, Sendable {
    public var eventId: String
    public var title: String
    public var notes: String?
    public var startDate: String
    public var endDate: String
    public var calendarName: String
    public var location: String?
    public var url: String?

    public init(
        eventId: String,
        title: String,
        notes: String?,
        startDate: String,
        endDate: String,
        calendarName: String,
        location: String?,
        url: String?
    ) {
        self.eventId = eventId
        self.title = title
        self.notes = notes
        self.startDate = startDate
        self.endDate = endDate
        self.calendarName = calendarName
        self.location = location
        self.url = url
    }
}

public struct OpenClawCalendarListResult: Codable, Sendable {
    public var events: [OpenClawCalendarEvent]

    public init(events: [OpenClawCalendarEvent]) {
        self.events = events
    }
}

// MARK: - calendar.search

public struct OpenClawCalendarSearchParams: Codable, Sendable {
    public var query: String
    public var startDate: String?
    public var endDate: String?
    public var limit: Int?

    public init(
        query: String,
        startDate: String? = nil,
        endDate: String? = nil,
        limit: Int? = nil
    ) {
        self.query = query
        self.startDate = startDate
        self.endDate = endDate
        self.limit = limit
    }
}
```

---

## Step 3: Implement macOS Calendar Service

**File:** `apps/macos/Sources/OpenClaw/NodeMode/MacNodeCalendarService.swift` (new file)

```swift
import EventKit
import Foundation
import OpenClawKit

/// Service for managing calendar events via EventKit
final class MacNodeCalendarService {
    enum Error: Swift.Error, LocalizedError {
        case permissionDenied
        case calendarNotFound(String)
        case eventNotFound(String)
        case invalidDateFormat(String)

        var errorDescription: String? {
            switch self {
            case .permissionDenied:
                return "Calendar permission denied"
            case .calendarNotFound(let name):
                return "Calendar '\(name)' not found"
            case .eventNotFound(let id):
                return "Event '\(id)' not found"
            case .invalidDateFormat(let date):
                return "Invalid date format: \(date)"
            }
        }
    }

    private let eventStore = EKEventStore()
    private let iso8601Formatter = ISO8601DateFormatter()

    init() {
        self.iso8601Formatter.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
    }

    // MARK: - Permissions

    /// Check current authorization status
    var authorizationStatus: EKAuthorizationStatus {
        if #available(macOS 14.0, *) {
            return EKEventStore.authorizationStatus(for: .event)
        } else {
            return EKEventStore.authorizationStatus(for: .event)
        }
    }

    /// Request calendar access
    func requestAccess() async throws {
        let granted: Bool
        if #available(macOS 14.0, *) {
            granted = try await self.eventStore.requestFullAccessToEvents()
        } else {
            granted = try await self.eventStore.requestAccess(to: .event)
        }

        guard granted else {
            throw Error.permissionDenied
        }
    }

    // MARK: - Create Event

    func createEvent(params: OpenClawCalendarCreateParams) async throws -> OpenClawCalendarCreateResult {
        // Ensure permissions
        if self.authorizationStatus != .fullAccess && self.authorizationStatus != .authorized {
            try await self.requestAccess()
        }

        // Parse dates
        guard let startDate = self.iso8601Formatter.date(from: params.startDate) else {
            throw Error.invalidDateFormat(params.startDate)
        }
        guard let endDate = self.iso8601Formatter.date(from: params.endDate) else {
            throw Error.invalidDateFormat(params.endDate)
        }

        // Find calendar
        let calendar = try self.findCalendar(name: params.calendarName ?? "Calendar")

        // Create event
        let event = EKEvent(eventStore: self.eventStore)
        event.calendar = calendar
        event.title = params.title
        event.notes = params.notes
        event.startDate = startDate
        event.endDate = endDate
        event.location = params.location

        // Add URL if provided
        if let urlString = params.url, let url = URL(string: urlString) {
            event.url = url
        }

        // Add alarms
        if let alarms = params.alarms {
            event.alarms = alarms.map { alarmParam in
                EKAlarm(relativeOffset: TimeInterval(alarmParam.offsetMinutes * 60))
            }
        }

        // Save to store
        try self.eventStore.save(event, span: .thisEvent)

        return OpenClawCalendarCreateResult(
            eventId: event.eventIdentifier,
            calendarName: calendar.title
        )
    }

    // MARK: - List Events

    func listEvents(params: OpenClawCalendarListParams) async throws -> OpenClawCalendarListResult {
        // Ensure permissions
        if self.authorizationStatus != .fullAccess && self.authorizationStatus != .authorized {
            try await self.requestAccess()
        }

        // Parse date range
        let startDate = params.startDate.flatMap { self.iso8601Formatter.date(from: $0) } ?? Date()
        let endDate = params.endDate.flatMap { self.iso8601Formatter.date(from: $0) }
            ?? Calendar.current.date(byAdding: .month, value: 1, to: startDate)!

        // Get calendars to search
        var calendars: [EKCalendar] = []
        if let calendarNames = params.calendars {
            for name in calendarNames {
                if let cal = try? self.findCalendar(name: name) {
                    calendars.append(cal)
                }
            }
        } else {
            calendars = self.eventStore.calendars(for: .event)
        }

        // Fetch events
        let predicate = self.eventStore.predicateForEvents(
            withStart: startDate,
            end: endDate,
            calendars: calendars
        )

        var events = self.eventStore.events(matching: predicate)

        // Apply limit
        if let limit = params.limit {
            events = Array(events.prefix(limit))
        }

        // Convert to result
        let resultEvents = events.map { event in
            OpenClawCalendarEvent(
                eventId: event.eventIdentifier,
                title: event.title ?? "",
                notes: event.notes,
                startDate: self.iso8601Formatter.string(from: event.startDate),
                endDate: self.iso8601Formatter.string(from: event.endDate),
                calendarName: event.calendar.title,
                location: event.location,
                url: event.url?.absoluteString
            )
        }

        return OpenClawCalendarListResult(events: resultEvents)
    }

    // MARK: - Search Events

    func searchEvents(params: OpenClawCalendarSearchParams) async throws -> OpenClawCalendarListResult {
        // Fetch all events in range
        let listParams = OpenClawCalendarListParams(
            startDate: params.startDate,
            endDate: params.endDate,
            calendars: nil,
            limit: nil
        )
        let allEvents = try await self.listEvents(params: listParams)

        // Filter by query
        let query = params.query.lowercased()
        var filtered = allEvents.events.filter { event in
            event.title.lowercased().contains(query)
                || (event.notes?.lowercased().contains(query) ?? false)
                || (event.location?.lowercased().contains(query) ?? false)
        }

        // Apply limit
        if let limit = params.limit {
            filtered = Array(filtered.prefix(limit))
        }

        return OpenClawCalendarListResult(events: filtered)
    }

    // MARK: - Helpers

    private func findCalendar(name: String) throws -> EKCalendar {
        let calendars = self.eventStore.calendars(for: .event)
        guard let calendar = calendars.first(where: { $0.title == name }) else {
            // If not found by exact match, try default calendar
            if let defaultCalendar = self.eventStore.defaultCalendarForNewEvents {
                return defaultCalendar
            }
            throw Error.calendarNotFound(name)
        }
        return calendar
    }
}
```

---

## Step 4: Integrate into macOS Node Runtime

**File:** `apps/macos/Sources/OpenClaw/NodeMode/MacNodeRuntime.swift`

Add calendar service property:

```swift
final class MacNodeRuntime {
    // ... existing services
    private let cameraController: MacNodeCameraController
    private let locationService: MacNodeLocationService
    private let screenRecordManager: MacNodeScreenRecordManager

    // Add calendar service
    private let calendarService = MacNodeCalendarService()  // ← Add

    // ... rest of implementation
}
```

Update capabilities in connection:

```swift
// In connectToGateway() method
let caps: [String] = [
    "canvas",
    "camera",
    "screen",
    "location",
    "voiceWake",
    "calendar",  // ← Add
]
```

Update commands declaration:

```swift
private func buildNodeCommands() -> [String] {
    var commands: [String] = []

    // ... existing commands

    // Calendar commands
    if self.calendarService.authorizationStatus == .authorized ||
       self.calendarService.authorizationStatus == .fullAccess {
        commands.append(contentsOf: [
            OpenClawCalendarCommand.create.rawValue,
            OpenClawCalendarCommand.list.rawValue,
            OpenClawCalendarCommand.search.rawValue,
        ])
    }

    return commands
}
```

Add invoke handler:

```swift
// In handleInvoke() method
private func handleInvoke(_ req: BridgeInvokeRequest) async -> BridgeInvokeResponse {
    let command = req.command

    switch command {
    // ... existing cases

    case OpenClawCalendarCommand.create.rawValue,
         OpenClawCalendarCommand.list.rawValue,
         OpenClawCalendarCommand.search.rawValue:
        return try await self.handleCalendarInvoke(req)

    default:
        return BridgeInvokeResponse(error: .unsupportedCommand)
    }
}

private func handleCalendarInvoke(_ req: BridgeInvokeRequest) async throws -> BridgeInvokeResponse {
    let command = req.command

    // Check permissions
    let status = self.calendarService.authorizationStatus
    guard status == .authorized || status == .fullAccess else {
        return BridgeInvokeResponse(error: .calendarPermissionRequired)
    }

    switch command {
    case OpenClawCalendarCommand.create.rawValue:
        let params = try Self.decodeParams(OpenClawCalendarCreateParams.self, from: req.paramsJSON)
        let result = try await self.calendarService.createEvent(params: params)
        let payload = try Self.encodePayload(result)
        return BridgeInvokeResponse(id: req.id, ok: true, payloadJSON: payload)

    case OpenClawCalendarCommand.list.rawValue:
        let params = try Self.decodeParams(OpenClawCalendarListParams.self, from: req.paramsJSON)
        let result = try await self.calendarService.listEvents(params: params)
        let payload = try Self.encodePayload(result)
        return BridgeInvokeResponse(id: req.id, ok: true, payloadJSON: payload)

    case OpenClawCalendarCommand.search.rawValue:
        let params = try Self.decodeParams(OpenClawCalendarSearchParams.self, from: req.paramsJSON)
        let result = try await self.calendarService.searchEvents(params: params)
        let payload = try Self.encodePayload(result)
        return BridgeInvokeResponse(id: req.id, ok: true, payloadJSON: payload)

    default:
        return BridgeInvokeResponse(error: .unsupportedCommand)
    }
}
```

Update error types in `BridgeFrames.swift`:

```swift
public enum OpenClawNodeErrorCode: String, Codable, Sendable {
    // ... existing codes
    case calendarPermissionRequired = "calendar_permission_required"
    case calendarNotFound = "calendar_not_found"
    case invalidDateFormat = "invalid_date_format"
}
```

---

## Step 5: Implement iOS Calendar Service

**File:** `apps/ios/Sources/Model/NodeAppModel.swift`

Follow the same pattern as macOS (iOS uses the same EventKit framework):

```swift
// Add calendar service
private let calendarService = IOSNodeCalendarService()  // Create similar service for iOS

// Update capabilities
let caps = ["canvas", "camera", "screen", "location", "calendar"]

// Update commands
private func buildNodeCommands() -> [String] {
    var commands: [String] = []
    // ... existing
    if self.calendarService.authorizationStatus == .authorized {
        commands.append(contentsOf: OpenClawCalendarCommand.allCases.map(\.rawValue))
    }
    return commands
}

// Add invoke handler (same as macOS)
private func handleCalendarInvoke(_ req: BridgeInvokeRequest) async throws -> BridgeInvokeResponse {
    // Same implementation as macOS
}
```

---

## Step 6: Add to nodes-tool (TypeScript)

**File:** `src/agents/tools/nodes-tool.ts`

Add calendar actions to schema:

```typescript
const NodesToolSchema = Type.Object({
  action: Type.Unsafe<
    | "list"
    | "camera_snap"
    | "camera_clip"
    | "location_get"
    | "screen_record"
    | "notify"
    | "calendar_create" // ← Add
    | "calendar_list" // ← Add
    | "calendar_search" // ← Add
  >({
    type: "string",
    enum: [
      "list",
      "camera_snap",
      "camera_clip",
      "location_get",
      "screen_record",
      "notify",
      "calendar_create",
      "calendar_list",
      "calendar_search",
    ],
  }),

  // ... existing params

  // Calendar params
  calendar_title: Type.Optional(Type.String()),
  calendar_notes: Type.Optional(Type.String()),
  calendar_start_date: Type.Optional(Type.String()), // ISO8601
  calendar_end_date: Type.Optional(Type.String()), // ISO8601
  calendar_name: Type.Optional(Type.String()),
  calendar_location: Type.Optional(Type.String()),
  calendar_query: Type.Optional(Type.String()),
  calendar_limit: Type.Optional(Type.Number()),
});
```

Add action implementations:

```typescript
export function createNodesTool(options?: { ... }): AnyAgentTool {
  return {
    name: "nodes",
    description: `Discover and control paired nodes (camera/location/notify/screen/calendar).

Actions:
  ...
  calendar_create - Create calendar event (params: calendar_title, calendar_start_date, calendar_end_date, calendar_notes?, calendar_location?, calendar_name?)
  calendar_list - List upcoming events (params: calendar_start_date?, calendar_end_date?, calendar_limit?)
  calendar_search - Search events by query (params: calendar_query, calendar_start_date?, calendar_end_date?, calendar_limit?)
`,
    parameters: NodesToolSchema,
    execute: async (_toolCallId, args) => {
      const action = args.action;

      switch (action) {
        // ... existing cases

        case "calendar_create": {
          const nodeId = await resolveNodeId(gatewayOpts, args.node);

          const raw = await callGatewayTool("node.invoke", gatewayOpts, {
            nodeId,
            command: "calendar.create",
            params: {
              title: args.calendar_title,
              notes: args.calendar_notes,
              startDate: args.calendar_start_date,
              endDate: args.calendar_end_date,
              calendarName: args.calendar_name,
              location: args.calendar_location,
            },
            idempotencyKey: crypto.randomUUID(),
          });

          const result = raw?.payload as { eventId: string; calendarName: string };

          return jsonResult({
            eventId: result.eventId,
            calendarName: result.calendarName,
            message: `Created event "${args.calendar_title}" in ${result.calendarName}`,
          });
        }

        case "calendar_list": {
          const nodeId = await resolveNodeId(gatewayOpts, args.node);

          const raw = await callGatewayTool("node.invoke", gatewayOpts, {
            nodeId,
            command: "calendar.list",
            params: {
              startDate: args.calendar_start_date,
              endDate: args.calendar_end_date,
              limit: args.calendar_limit,
            },
            idempotencyKey: crypto.randomUUID(),
          });

          const result = raw?.payload as { events: CalendarEvent[] };

          return jsonResult({
            count: result.events.length,
            events: result.events,
          });
        }

        case "calendar_search": {
          const nodeId = await resolveNodeId(gatewayOpts, args.node);

          const raw = await callGatewayTool("node.invoke", gatewayOpts, {
            nodeId,
            command: "calendar.search",
            params: {
              query: args.calendar_query,
              startDate: args.calendar_start_date,
              endDate: args.calendar_end_date,
              limit: args.calendar_limit,
            },
            idempotencyKey: crypto.randomUUID(),
          });

          const result = raw?.payload as { events: CalendarEvent[] };

          return jsonResult({
            query: args.calendar_query,
            count: result.events.length,
            events: result.events,
          });
        }
      }
    },
  };
}
```

---

## Step 7: Add CLI Command (Optional)

**File:** `src/cli/nodes-cli/register.calendar.ts` (new file)

```typescript
import type { Command } from "commander";
import { getGatewayClient } from "../client.js";
import { renderTable } from "../../terminal/table.js";

export function registerCalendarCommands(program: Command) {
  const calendar = program.command("calendar").description("Calendar commands for paired nodes");

  // openclaw nodes calendar create
  calendar
    .command("create")
    .description("Create a calendar event")
    .requiredOption("--title <title>", "Event title")
    .requiredOption("--start <iso8601>", "Start date (ISO8601)")
    .requiredOption("--end <iso8601>", "End date (ISO8601)")
    .option("--notes <notes>", "Event notes")
    .option("--location <location>", "Event location")
    .option("--calendar <name>", "Calendar name", "Calendar")
    .option("--node <id>", "Node ID (default: first with calendar cap)")
    .action(async (opts) => {
      const client = await getGatewayClient();

      const result = await client.send("node.invoke", {
        nodeId: opts.node || "auto:calendar",
        command: "calendar.create",
        params: {
          title: opts.title,
          notes: opts.notes,
          startDate: opts.start,
          endDate: opts.end,
          calendarName: opts.calendar,
          location: opts.location,
        },
      });

      if (result.ok) {
        console.log(`✓ Created event: ${result.payload.eventId}`);
      } else {
        console.error(`✗ Error: ${result.error?.message}`);
      }
    });

  // openclaw nodes calendar list
  calendar
    .command("list")
    .description("List upcoming calendar events")
    .option("--start <iso8601>", "Start date")
    .option("--end <iso8601>", "End date")
    .option("--limit <n>", "Max events to return", "20")
    .option("--node <id>", "Node ID")
    .action(async (opts) => {
      const client = await getGatewayClient();

      const result = await client.send("node.invoke", {
        nodeId: opts.node || "auto:calendar",
        command: "calendar.list",
        params: {
          startDate: opts.start,
          endDate: opts.end,
          limit: parseInt(opts.limit, 10),
        },
      });

      if (result.ok) {
        const events = result.payload.events;
        renderTable({
          headers: ["Title", "Start", "End", "Calendar"],
          rows: events.map((e: any) => [
            e.title,
            new Date(e.startDate).toLocaleString(),
            new Date(e.endDate).toLocaleString(),
            e.calendarName,
          ]),
        });
      } else {
        console.error(`✗ Error: ${result.error?.message}`);
      }
    });

  // openclaw nodes calendar search
  calendar
    .command("search <query>")
    .description("Search calendar events")
    .option("--start <iso8601>", "Start date")
    .option("--end <iso8601>", "End date")
    .option("--limit <n>", "Max results", "20")
    .option("--node <id>", "Node ID")
    .action(async (query, opts) => {
      const client = await getGatewayClient();

      const result = await client.send("node.invoke", {
        nodeId: opts.node || "auto:calendar",
        command: "calendar.search",
        params: {
          query,
          startDate: opts.start,
          endDate: opts.end,
          limit: parseInt(opts.limit, 10),
        },
      });

      if (result.ok) {
        const events = result.payload.events;
        console.log(`Found ${events.length} events matching "${query}"`);
        renderTable({
          headers: ["Title", "Start", "Calendar"],
          rows: events.map((e: any) => [
            e.title,
            new Date(e.startDate).toLocaleString(),
            e.calendarName,
          ]),
        });
      } else {
        console.error(`✗ Error: ${result.error?.message}`);
      }
    });
}
```

**Update:** `src/cli/nodes-cli/index.ts`

```typescript
import { registerCalendarCommands } from "./register.calendar.js";

export function registerNodesCommands(program: Command) {
  const nodes = program.command("nodes").description("Node management");

  // ... existing registrations
  registerCameraCommands(nodes);
  registerLocationCommands(nodes);
  registerScreenCommands(nodes);
  registerCalendarCommands(nodes); // ← Add
}
```

---

## Step 8: Update Permissions (Info.plist)

**macOS:** `apps/macos/Sources/OpenClaw/Resources/Info.plist`

```xml
<key>NSCalendarsUsageDescription</key>
<string>OpenClaw needs calendar access to create and read events on your behalf.</string>
<key>NSCalendarsFullAccessUsageDescription</key>
<string>OpenClaw needs full calendar access to manage events across all calendars.</string>
```

**iOS:** `apps/ios/Sources/Info.plist`

```xml
<key>NSCalendarsUsageDescription</key>
<string>OpenClaw needs calendar access to create and read events.</string>
<key>NSCalendarsFullAccessUsageDescription</key>
<string>OpenClaw needs full calendar access to manage all your calendar events.</string>
```

---

## Step 9: Testing

### Test 1: Manual Swift Testing

```bash
# Build macOS app
cd apps/macos
xcodebuild -scheme OpenClaw -configuration Debug

# Run app with node mode
# Enable calendar permissions in System Settings → Privacy
```

### Test 2: CLI Testing

```bash
# Start gateway + macOS node app

# Create event via CLI
openclaw nodes calendar create \
  --title "Team Meeting" \
  --start "2024-02-15T14:00:00Z" \
  --end "2024-02-15T15:00:00Z" \
  --notes "Discuss Q1 roadmap" \
  --location "Conference Room A"

# List events
openclaw nodes calendar list --limit 10

# Search events
openclaw nodes calendar search "roadmap"
```

### Test 3: Agent Testing

```bash
openclaw agent

> "Create a calendar event tomorrow at 2pm for a dentist appointment"

# Agent should use:
# nodes tool, action: calendar_create
# params: { title: "Dentist Appointment", startDate: "...", endDate: "..." }

> "What meetings do I have this week?"

# Agent should use:
# nodes tool, action: calendar_list
# params: { startDate: "...", endDate: "..." }

> "Search my calendar for any events about project alpha"

# Agent should use:
# nodes tool, action: calendar_search
# params: { query: "project alpha" }
```

---

## Complete File Checklist

| File                                                                 | Purpose                            | Status  |
| -------------------------------------------------------------------- | ---------------------------------- | ------- |
| `src/gateway/node-protocol/calendar-commands.ts`                     | TypeScript types                   | ✅ New  |
| `src/gateway/node-command-policy.ts`                                 | Add CALENDAR_COMMANDS to allowlist | ✅ Edit |
| `apps/shared/OpenClawKit/Sources/OpenClawKit/CalendarCommands.swift` | Swift protocol types               | ✅ New  |
| `apps/macos/Sources/OpenClaw/NodeMode/MacNodeCalendarService.swift`  | EventKit service (macOS)           | ✅ New  |
| `apps/macos/Sources/OpenClaw/NodeMode/MacNodeRuntime.swift`          | Integrate calendar service         | ✅ Edit |
| `apps/ios/Sources/Services/IOSNodeCalendarService.swift`             | EventKit service (iOS)             | ✅ New  |
| `apps/ios/Sources/Model/NodeAppModel.swift`                          | Integrate calendar service         | ✅ Edit |
| `apps/shared/OpenClawKit/Sources/OpenClawKit/BridgeFrames.swift`     | Add error codes                    | ✅ Edit |
| `src/agents/tools/nodes-tool.ts`                                     | Add calendar actions               | ✅ Edit |
| `src/cli/nodes-cli/register.calendar.ts`                             | CLI commands                       | ✅ New  |
| `src/cli/nodes-cli/index.ts`                                         | Register calendar commands         | ✅ Edit |
| `apps/macos/Sources/OpenClaw/Resources/Info.plist`                   | Calendar permissions               | ✅ Edit |
| `apps/ios/Sources/Info.plist`                                        | Calendar permissions               | ✅ Edit |

---

## Summary: Node Delegation Pattern

The node delegation pattern follows this workflow:

1. **Define protocol** (TypeScript types + Swift structs)
2. **Implement native service** (Swift/Kotlin using platform APIs)
3. **Integrate into node runtime** (handle invoke, update caps/commands)
4. **Update command policy** (add to platform allowlist)
5. **Wrap in nodes-tool** (TypeScript agent tool)
6. **Add CLI commands** (optional user-facing interface)
7. **Update permissions** (Info.plist / AndroidManifest)
8. **Test** (manual → CLI → agent)

This pattern works for **any platform-specific capability**:

- Calendar (EventKit)
- Contacts (Contacts framework)
- Photos (PhotoKit)
- Files (DocumentPicker)
- Health (HealthKit)
- HomeKit devices
- Siri Shortcuts
- Push notifications
- Bluetooth
- NFC
- Etc.

The key is: **TypeScript defines the protocol, native apps implement it, gateway routes it.**
