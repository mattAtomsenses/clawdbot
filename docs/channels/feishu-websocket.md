# Feishu Plugin WebSocket Setup

This document describes how the WebSocket connection is set up in the Feishu/Lark plugin for OpenClaw.

## Overview

The Feishu plugin uses the official **`@larksuiteoapi/node-sdk`** SDK to establish a WebSocket long connection with Feishu/Lark open platform. This enables real-time bidirectional event streaming without requiring a public server.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw Gateway                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Feishu Plugin Extension                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │  │
│  │  │   Monitor    │  │    Bot       │  │    Send    │  │  │
│  │  │  (WebSocket) │  │  (Handler)   │  │  (Client)  │  │  │
│  │  └──────────────┘  └──────────────┘  └────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↕ WebSocket
┌─────────────────────────────────────────────────────────────┐
│                    飞书开放平台 (Lark)                        │
│  - Event push (im.message.receive_v1)                       │
│  - Bot added/removed events                                 │
│  - Message read receipts                                     │
└─────────────────────────────────────────────────────────────┘
```

## Core Files

| File | Purpose |
|------|---------|
| [client.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/client.ts) | WebSocket client creation |
| [monitor.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/monitor.ts) | Connection lifecycle and event handling |
| [bot.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/bot.ts) | Message processing logic |
| [accounts.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/accounts.ts) | Credential resolution |

## WebSocket Client Creation

### 1. Creating the WS Client

Located in **[client.ts:38-50](https://github.com/m1heng/clawdbot-feishu/blob/main/src/client.ts#L38-L50)**:

```typescript
export function createFeishuWSClient(cfg: FeishuConfig): Lark.WSClient {
  const creds = resolveFeishuCredentials(cfg);
  if (!creds) {
    throw new Error("Feishu credentials not configured (appId, appSecret required)");
  }
  return new Lark.WSClient({
    appId: creds.appId,
    appSecret: creds.appSecret,
    domain: resolveDomain(creds.domain),  // "feishu" or "lark"
    loggerLevel: Lark.LoggerLevel.info,
  });
}
```

### 2. Creating the Event Dispatcher

Located in **[client.ts:52-60](https://github.com/m1heng/clawdbot-feishu/blob/main/src/client.ts#L52-L60)**:

```typescript
export function createEventDispatcher(cfg: FeishuConfig): Lark.EventDispatcher {
  const creds = resolveFeishuCredentials(cfg);
  return new Lark.EventDispatcher({
    encryptKey: creds?.encryptKey,
    verificationToken: creds?.verificationToken,
  });
}
```

The `EventDispatcher` handles:
- Event validation (signature verification)
- Event decryption (if encryption is enabled)
- Routing events to registered handlers

## Event Registration

Located in **[monitor.ts:68-103](https://github.com/m1heng/clawdbot-feishu/blob/main/src/monitor.ts#L68-L103)**:

```typescript
eventDispatcher.register({
  "im.message.receive_v1": async (data) => {
    try {
      const event = data as unknown as FeishuMessageEvent;
      await handleFeishuMessage({
        cfg,
        event,
        botOpenId,
        runtime,
        chatHistories,
      });
    } catch (err) {
      error(`feishu: error handling message event: ${String(err)}`);
    }
  },

  "im.message.message_read_v1": async () => {
    // Ignore read receipts
  },

  "im.chat.member.bot.added_v1": async (data) => {
    try {
      const event = data as unknown as FeishuBotAddedEvent;
      log(`feishu: bot added to chat ${event.chat_id}`);
    } catch (err) {
      error(`feishu: error handling bot added event: ${String(err)}`);
    }
  },

  "im.chat.member.bot.deleted_v1": async (data) => {
    try {
      const event = data as unknown as { chat_id: string };
      log(`feishu: bot removed from chat ${event.chat_id}`);
    } catch (err) {
      error(`feishu: error handling bot removed event: ${String(err)}`);
    }
  },
});
```

## Starting the WebSocket Connection

Located in **[monitor.ts:113-143](https://github.com/m1heng/clawdbot-feishu/blob/main/src/monitor.ts#L113-L143)**:

```typescript
return new Promise((resolve, reject) => {
  const cleanup = () => {
    if (currentWsClient === wsClient) {
      currentWsClient = null;
    }
  };

  const handleAbort = () => {
    log("feishu: abort signal received, stopping WebSocket client");
    cleanup();
    resolve();
  };

  if (abortSignal?.aborted) {
    cleanup();
    resolve();
    return;
  }

  abortSignal?.addEventListener("abort", handleAbort, { once: true });

  try {
    wsClient.start({
      eventDispatcher,
    });
    log("feishu: WebSocket client started");
  } catch (err) {
    cleanup();
    abortSignal?.removeEventListener("abort", handleAbort);
    reject(err);
  }
});
```

## Configuration

### Enabling WebSocket Mode

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxx",
      "appSecret": "your_app_secret",
      "domain": "feishu",
      "connectionMode": "websocket"
    }
  }
}
```

### Connection Modes

| Mode | Description | Pros | Cons |
|------|-------------|------|------|
| `websocket` | Long connection (recommended) | No public server needed, real-time events | - |
| `webhook` | HTTP callback | Traditional approach | Requires public HTTPS endpoint |

### Domain Configuration

| Domain | Region | API Endpoint |
|--------|--------|--------------|
| `feishu` | China | `open.feishu.cn` |
| `lark` | International | `open.larksuite.com` |
| Custom URL | Private deployment | Your specified URL |

## Supported Events

| Event Type | Handler | Description |
|------------|---------|-------------|
| `im.message.receive_v1` | `handleFeishuMessage` | Incoming messages (DM and group) |
| `im.message.message_read_v1` | (ignored) | Message read receipts |
| `im.chat.member.bot.added_v1` | Log only | Bot added to group |
| `im.chat.member.bot.deleted_v1` | Log only | Bot removed from group |

## Feishu Platform Setup

### 1. Enable Long Connection

In Feishu Open Platform → **Events & Callbacks**:

1. Go to **Event configuration**
2. Select **"Use long connection to receive events"**
3. Click **Save**

### 2. Subscribe to Events

Add the following event subscriptions:

| Event | Required |
|-------|----------|
| `im.message.receive_v1` | ✅ Yes |
| `im.message.message_read_v1` | Optional |
| `im.chat.member.bot.added_v1` | Recommended |
| `im.chat.member.bot.deleted_v1` | Recommended |

### 3. Required Permissions

| Permission | Scope | Description |
|------------|-------|-------------|
| `im:message` | Messaging | Send and receive messages |
| `im:message.p2p_msg:readonly` | DM | Read direct messages |
| `im:message.group_at_msg:readonly` | Group | Receive @mentions in groups |
| `im:message:send_as_bot` | Send | Send messages as bot |
| `im:resource` | Media | Upload/download images/files |

### 4. Publish the App

1. Go to **Version Management & Release**
2. Create a new version
3. Publish (at least to test version)

## Troubleshooting

### Bot Cannot Receive Messages

**Check:**
1. ✅ Event subscription configured (long connection enabled)
2. ✅ `im.message.receive_v1` event added
3. ✅ Permissions approved
4. ✅ App published
5. ✅ Gateway running: `moltbot channels status`

### WebSocket Connection Failed

**Log output:**
```
feishu: WebSocket connection failed
```

**Solutions:**
1. Verify `appId` and `appSecret` are correct
2. Check network connectivity to `open.feishu.cn` or `open.larksuite.com`
3. Ensure long connection is enabled in Feishu console
4. Restart gateway: `moltbot gateway restart`

### Event Handlers Not Called

**Check:**
```bash
# View real-time logs
tail -f /tmp/moltbot/moltbot-*.log | grep -i feishu

# Check if events are being received
moltbot config set logLevel debug
moltbot gateway restart
```

## Linux Environment Setup

### System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| OS | Ubuntu 20.04+ / CentOS 8+ / Debian 11+ | Ubuntu 22.04 LTS |
| Node.js | 22+ | Latest 22.x |
| RAM | 512 MB | 1 GB+ |
| Network | Outbound HTTPS (443) | Stable connection |

### Network Configuration

#### Firewall Settings

WebSocket requires **outbound** HTTPS to Feishu servers. No inbound ports needed.

**For Feishu (China):**
```bash
# Allow outbound HTTPS to Feishu
sudo iptables -A OUTPUT -p tcp --dport 443 -d open.feishu.cn -j ACCEPT
sudo iptables -A OUTPUT -p tcp --dport 443 -d feishu.cn -j ACCEPT

# For firewalld
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" destination address="open.feishu.cn" service name="https" accept'
sudo firewall-cmd --reload
```

**For Lark (International):**
```bash
# Allow outbound HTTPS to Lark
sudo iptables -A OUTPUT -p tcp --dport 443 -d open.larksuite.com -j ACCEPT
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" destination address="open.larksuite.com" service name="https" accept'
```

#### Proxy Configuration

If behind a corporate proxy, configure environment variables:

```bash
# /etc/systemd/system/moltbot-gateway.service
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:8080"
Environment="HTTPS_PROXY=http://proxy.example.com:8080"
Environment="NO_PROXY=localhost,127.0.0.1"
```

Or for user session:
```bash
export HTTPS_PROXY=http://proxy.example.com:8080
moltbot gateway start
```

### Installation on Linux

#### 1. Install Node.js 22+

```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# CentOS/RHEL
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo yum install -y nodejs

# Verify
node --version  # Should be v22.x.x
```

#### 2. Install OpenClaw

```bash
# Global npm install
sudo npm install -g openclaw

# Or use pnpm (recommended for development)
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
pnpm link --global
```

#### 3. Install Feishu Plugin

```bash
# Install from npm
openclaw plugins install @m1heng-clawd/feishu

# Or from local file (if npm install fails)
curl -O https://registry.npmjs.org/@m1heng-clawd/feishu/-/feishu-0.1.3.tgz
openclaw plugins install ./feishu-0.1.3.tgz
```

### systemd Service Configuration

Create a systemd service for automatic startup:

**`/etc/systemd/system/moltbot-gateway.service`:**

```ini
[Unit]
Description=OpenClaw Gateway with Feishu Plugin
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=moltbot
Group=moltbot
WorkingDirectory=/opt/moltbot
Environment="NODE_ENV=production"
ExecStart=/usr/bin/moltbot gateway run --bind 0.0.0.0 --port 18789
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=moltbot-gateway

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/moltbot /tmp/moltbot

[Install]
WantedBy=multi-user.target
```

**Enable and start the service:**

```bash
# Create dedicated user (optional but recommended)
sudo useradd -r -s /bin/false -d /var/lib/moltbot moltbot

# Create directories
sudo mkdir -p /var/lib/moltbot /tmp/moltbot
sudo chown -R moltbot:moltbot /var/lib/moltbot /tmp/moltbot

# Reload systemd and enable service
sudo systemctl daemon-reload
sudo systemctl enable moltbot-gateway
sudo systemctl start moltbot-gateway

# Check status
sudo systemctl status moltbot-gateway
```

### Log Management

#### Log Locations

| Log Type | Location |
|----------|----------|
| Gateway logs | `/tmp/moltbot/moltbot-*.log` |
| journald logs | `journalctl -u moltbot-gateway` |
| Plugin logs | Included in gateway logs |

#### Viewing Logs

```bash
# Real-time gateway logs
tail -f /tmp/moltbot/moltbot-*.log

# Feishu-specific logs
tail -f /tmp/moltbot/moltbot-*.log | grep -i feishu

# systemd journal logs
sudo journalctl -u moltbot-gateway -f

# Last 100 lines with errors
sudo journalctl -u moltbot-gateway -n 100 --priority=err

# Search for WebSocket connection events
sudo journalctl -u moltbot-gateway -g "WebSocket"
```

#### Log Rotation

Create `/etc/logrotate.d/moltbot-gateway`:

```conf
/tmp/moltbot/moltbot-*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 moltbot moltbot
    sharedscripts
    postrotate
        systemctl reload moltbot-gateway > /dev/null 2>&1 || true
    endscript
}
```

### Configuration Files

#### Config Location

| Platform | Config Path |
|----------|-------------|
| Linux (systemd) | `/var/lib/moltbot/.openclaw/config.json` |
| Linux (user) | `~/.openclaw/config.json` |

#### Setting Configuration

```bash
# Using CLI
moltbot config set channels.feishu.enabled true
moltbot config set channels.feishu.appId "cli_xxxxx"
moltbot config set channels.feishu.appSecret "your_secret"
moltbot config set channels.feishu.connectionMode "websocket"

# Or edit directly
nano ~/.openclaw/config.json
```

### Process Management

#### Starting/Stopping

```bash
# Start gateway
moltbot gateway start
# or
sudo systemctl start moltbot-gateway

# Stop gateway
moltbot gateway stop
# or
sudo systemctl stop moltbot-gateway

# Restart gateway
moltbot gateway restart
# or
sudo systemctl restart moltbot-gateway

# Check status
moltbot gateway status
# or
sudo systemctl status moltbot-gateway
```

#### Auto-restart on Failure

The systemd service above includes `Restart=always` for automatic recovery. For additional monitoring:

```bash
# Check service health
sudo systemctl is-active moltbot-gateway

# Enable monitoring alerts (requires monitoring setup)
sudo apt install monitoring-tools  # Example
```

### Permission Issues

#### File Permissions

```bash
# Fix permission denied errors
sudo chown -R moltbot:moltbot /var/lib/moltbot
sudo chmod 755 /var/lib/moltbot

# Fix log file permissions
sudo chown moltbot:moltbot /tmp/moltbot -R
sudo chmod 755 /tmp/moltbot
```

#### SELinux (if enabled)

```bash
# Check if SELinux is blocking
sudo ausearch -m avc -ts recent | grep moltbot

# Create custom policy if needed
sudo audit2allow -w -a
```

### Performance Tuning

#### Increase File Limits

For high-traffic scenarios:

```bash
# /etc/systemd/system/moltbot-gateway.service
[Service]
LimitNOFILE=65536
LimitNPROC=4096
```

#### Memory Limits

```bash
[Service]
MemoryMax=1G
MemorySwapMax=512M
```

### Docker Deployment (Alternative)

**`Dockerfile`:**

```dockerfile
FROM node:22-alpine
RUN npm install -g openclaw
RUN openclaw plugins install @m1heng-clawd/feishu
COPY config.json /root/.openclaw/
EXPOSE 18789
CMD ["moltbot", "gateway", "run", "--bind", "0.0.0.0", "--port", "18789"]
```

**`docker-compose.yml`:**

```yaml
version: '3.8'
services:
  moltbot:
    build: .
    ports:
      - "18789:18789"
    volumes:
      - ./config:/root/.openclaw
      - moltbot-data:/var/lib/moltbot
    restart: unless-stopped
```

### Testing WebSocket Connection

```bash
# Test connectivity to Feishu
curl -v https://open.feishu.cn

# Test DNS resolution
nslookup open.feishu.cn

# Test WebSocket endpoint (requires wscat)
npm install -g wscat
wscat -c "wss://push.feishu.cn/ws"
```

## Security Considerations

### Encryption Keys

If your Feishu app uses event encryption, configure:

```json
{
  "channels": {
    "feishu": {
      "encryptKey": "your_encrypt_key",
      "verificationToken": "your_verification_token"
    }
  }
}
```

### Access Control

Use appropriate policies for DM and group access:

```json
{
  "channels": {
    "feishu": {
      "dmPolicy": "pairing",      // "open" | "pairing" | "allowlist"
      "groupPolicy": "allowlist", // "open" | "allowlist" | "disabled"
      "requireMention": true
    }
  }
}
```

## References

- [Feishu Plugin GitHub](https://github.com/m1heng/clawdbot-feishu)
- [@larksuiteoapi/node-sdk](https://www.npmjs.com/package/@larksuiteoapi/node-sdk)
- [Feishu WebSocket Documentation](https://open.feishu.cn/document/common-capabilities/event-subscription/subscribe-to-events/push-methods/persistent-connection)
- [Local Setup Guide](docs/channels/feishu-setup.md)

## Related Files

- [feishu-setup.md](feishu-setup.md) - Complete setup guide
- [src/client.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/client.ts) - Client implementation
- [src/monitor.ts](https://github.com/m1heng/clawdbot-feishu/blob/main/src/monitor.ts) - Monitor implementation