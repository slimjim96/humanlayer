# HumanLayer .NET Automation Examples

Cross-platform .NET 10 examples for automating AI CLI tools with human-in-the-loop oversight using the HumanLayer daemon.

## Overview

This project demonstrates how to:

- **Run AI sessions programmatically** via the HumanLayer daemon REST API
- **Process approvals in batches** for periodic human review
- **Schedule automated tasks** that run 24/7 with configurable oversight
- **Monitor events in real-time** using Server-Sent Events (SSE)
- **Run parallel AI tasks** to maximize throughput
- **Use alternative AI providers** via OpenRouter proxy

## Prerequisites

1. **.NET 10 SDK** - Install from https://dotnet.microsoft.com/download
2. **HumanLayer Daemon (hld)** - Running locally or accessible via network
3. **Claude Code** - Required for Claude-based sessions

### Starting the Daemon

```bash
# Install hld if not already installed
# (See main HumanLayer documentation)

# Start the daemon
hld daemon start
```

## Quick Start

```bash
cd examples/dotnet-automation

# Restore dependencies
dotnet restore

# Run the demo to verify connectivity
dotnet run -- demo
```

## Available Modes

### Demo Mode
Quick demonstration of API capabilities - health check, list sessions, list approvals.

```bash
dotnet run -- demo
```

### Batch Processor
Processes pending approvals in batches. Automatically approves safe read-only tools, prompts for manual review on others.

```bash
dotnet run -- batch
```

**Features:**
- Auto-approves safe tools (Read, Glob, Grep, etc.)
- Interactive prompts for dangerous operations
- Continuous polling with configurable interval
- Statistics on processed approvals

### Task Scheduler
Runs AI tasks on a schedule, similar to cron. Ideal for 24/7 background automation.

```bash
dotnet run -- scheduler
```

**Example scheduled tasks:**
- Hourly: Code review summaries (read-only, auto-approved)
- Every 4 hours: Security scans (limited tools, auto-approved)
- Daily: Documentation updates (requires human approval for edits)

### Real-time Monitor
Watches for events via SSE stream. Useful for dashboards or notification systems.

```bash
dotnet run -- monitor
```

### Parallel Tasks
Runs multiple AI tasks concurrently to maximize throughput.

```bash
dotnet run -- parallel
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `HUMANLAYER_URL` | Daemon REST API URL | `http://localhost:7777/api/v1` |
| `WORKING_DIR` | Default working directory for sessions | Current directory |
| `OPENROUTER_API_KEY` | API key for OpenRouter proxy | (none) |

### Using OpenRouter for Alternative AI Providers

You can route requests through OpenRouter to use models from OpenAI, Anthropic, Meta, Mistral, and others:

```csharp
var task = new AutomationTask
{
    Name = "GPT-4 Analysis",
    Query = "Analyze this codebase",
    ProxyBaseUrl = "https://openrouter.ai/api/v1",
    ProxyModel = "openai/gpt-4-turbo",
    ProxyApiKey = Environment.GetEnvironmentVariable("OPENROUTER_API_KEY")
};
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Your .NET Application                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Scheduler  │  │    Batch     │  │   Parallel   │       │
│  │              │  │  Processor   │  │    Runner    │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         └─────────────────┼─────────────────┘               │
│                           ▼                                  │
│                 ┌──────────────────┐                        │
│                 │ HumanLayerClient │                        │
│                 └────────┬─────────┘                        │
└──────────────────────────┼──────────────────────────────────┘
                           │ HTTP/REST
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                   HumanLayer Daemon (hld)                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐ │
│  │  Sessions  │  │  Approvals │  │  OpenRouter Proxy      │ │
│  │  Manager   │  │  Queue     │  │  (alternative models)  │ │
│  └─────┬──────┘  └─────┬──────┘  └───────────┬────────────┘ │
└────────┼───────────────┼─────────────────────┼──────────────┘
         │               │                     │
         ▼               ▼                     ▼
    ┌─────────┐    ┌──────────┐         ┌──────────────┐
    │ Claude  │    │  Human   │         │  OpenRouter  │
    │  Code   │    │ Reviewer │         │   + GPT-4    │
    └─────────┘    └──────────┘         │   + Llama    │
                                        │   + etc.     │
                                        └──────────────┘
```

## Automation Patterns

### Pattern 1: Fully Automated (Trusted Tasks)

For safe, read-only tasks that don't need human oversight:

```csharp
var task = new AutomationTask
{
    Name = "Code Analysis",
    Query = "Analyze code quality",
    AutoApprove = true,
    AutoApproveTimeout = TimeSpan.FromMinutes(5),
    AllowedTools = ["Read", "Glob", "Grep"] // Whitelist safe tools
};
```

### Pattern 2: Human-in-the-Loop (Sensitive Tasks)

For tasks that modify files or execute commands:

```csharp
var task = new AutomationTask
{
    Name = "Code Refactor",
    Query = "Refactor the authentication module",
    AutoApprove = false // Every tool use requires approval
};

// Run task in background, review approvals periodically
var sessionData = await client.CreateSessionAsync(/* ... */);

// Later, in your review loop:
var approvals = await client.GetPendingApprovalsAsync();
foreach (var approval in approvals)
{
    // Review and decide
    if (IsAcceptable(approval))
        await client.ApproveAsync(approval.Id);
    else
        await client.DenyAsync(approval.Id, "Reason for denial");
}
```

### Pattern 3: Hybrid (Auto-approve Safe, Review Dangerous)

```csharp
var safeTools = new HashSet<string> { "Read", "Glob", "Grep" };

var approvals = await client.GetPendingApprovalsAsync();
foreach (var approval in approvals)
{
    if (safeTools.Contains(approval.ToolName))
    {
        await client.ApproveAsync(approval.Id, "Auto-approved");
    }
    else
    {
        // Queue for human review or prompt interactively
    }
}
```

## API Reference

### HumanLayerClient

```csharp
// Create client
var client = new HumanLayerClient("http://localhost:7777/api/v1");

// Health check
var health = await client.HealthAsync();

// Sessions
var session = await client.CreateSessionAsync(request);
var sessions = await client.ListSessionsAsync();
var details = await client.GetSessionAsync(sessionId);
var result = await client.WaitForSessionAsync(sessionId, timeout);
await client.InterruptSessionAsync(sessionId);

// Approvals
var approvals = await client.GetPendingApprovalsAsync();
await client.ApproveAsync(approvalId, comment);
await client.DenyAsync(approvalId, reason);

// Events (SSE)
await client.SubscribeToEventsAsync(onEvent, eventTypes, sessionId, ct);

// High-level automation
var result = await client.RunTaskAsync(automationTask);
```

## Extending the Examples

### Adding Custom Approval Policies

Create a policy-based approval system:

```csharp
public interface IApprovalPolicy
{
    ApprovalDecision Evaluate(Approval approval);
}

public record ApprovalDecision(bool AutoApprove, string? Reason);

public class TimeBasedPolicy : IApprovalPolicy
{
    public ApprovalDecision Evaluate(Approval approval)
    {
        // Auto-approve during business hours only
        var hour = DateTime.Now.Hour;
        if (hour >= 9 && hour < 17)
            return new(true, "Business hours auto-approve");
        return new(false, null);
    }
}
```

### Integrating with Notification Systems

```csharp
await client.SubscribeToEventsAsync(
    onEvent: async evt =>
    {
        if (evt.Type == "new_approval")
        {
            await SendSlackNotification($"New approval needed: {evt.Data}");
        }
    },
    eventTypes: ["new_approval"]
);
```

## Troubleshooting

### Cannot connect to daemon
- Ensure `hld daemon start` is running
- Check the daemon URL matches your configuration
- Verify firewall allows connections to port 7777

### Sessions stuck in "running"
- Check Claude Code is installed and accessible
- Review daemon logs: `hld logs`
- Verify working directory exists and is accessible

### Approvals not appearing
- Ensure session is using permission prompt tool
- Check MCP configuration is correct

## License

MIT License - See main HumanLayer repository for details.
