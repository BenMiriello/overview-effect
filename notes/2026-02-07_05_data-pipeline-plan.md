# Data Pipeline Plan

## Executive Summary

This document outlines improvements to the lightning data pipeline from Blitzortung.org to the client visualization. The primary recommendation is to replace the Puppeteer-based scraping approach with a direct WebSocket connection, which will reduce resource usage by ~95% while providing access to richer strike data including altitude, polarity, and signal quality metrics.

---

## 1. Blitzortung Data Source Assessment

### 1.1 Current WebSocket Protocol Status

Blitzortung's WebSocket servers remain active and use a consistent protocol:

**Server Pool:**
```
wss://ws1.blitzortung.org:3000/
wss://ws5.blitzortung.org:3000/
wss://ws6.blitzortung.org:3000/
wss://ws7.blitzortung.org:3000/
```

**Connection Protocol:**
1. Connect to any server via secure WebSocket (wss)
2. Send initial subscription message: `{"time": 0}`
3. Receive LZW-compressed JSON messages for each strike

**Observation:** The LZW decode function in `server/utils/index.js` remains compatible with the current protocol.

### 1.2 Available Data Fields

Each lightning strike message contains significantly more data than we currently extract:

| Field | Type | Description | Current Usage |
|-------|------|-------------|---------------|
| `time` | number | Unix timestamp in nanoseconds | Converted to ms |
| `lat` | number | Latitude (decimal degrees) | Extracted |
| `lon` | number | Longitude (decimal degrees) | Extracted |
| `alt` | number | Altitude above sea level (meters) | **Not captured** |
| `pol` | number | Polarity (-1 negative, +1 positive) | **Not captured** |
| `mds` | number | Max deviation range (nanoseconds) | **Not captured** |
| `mcg` | number | Max circular gap (degrees) | **Not captured** |
| `delay` | number | Processing delay (seconds) | **Not captured** |
| `region` | number | Geographic region code (1-5) | **Not captured** |
| `sig` | array | Participating detector signals | **Not captured** |

**Signal Array (`sig`) Fields:**
| Field | Description |
|-------|-------------|
| `sta` | Station identifier |
| `time` | Nanosecond offset from strike time |
| `lat/lon` | Detector coordinates |
| `alt` | Detector elevation |
| `status` | Signal quality + polarity flags |

**Region Codes:**
- 1: Europe
- 2: Oceania
- 3: North America
- 4: Asia
- 5: South America

### 1.3 Data Usage Policy

Per Blitzortung's terms, third-party applications should relay data through their own servers rather than having clients connect directly. The current architecture (server relays to clients) complies with this requirement.

### 1.4 Alternative Endpoints

Blitzortung also provides authenticated REST API access:
```
https://username:password@data.blitzortung.org/Data/Protected/last_strikes.php
```
Parameters: `west`, `east`, `north`, `south` (geographic bounds), temporal filters.

This is useful for backfilling/historical data but not for real-time streaming.

---

## 2. Server Architecture Improvements

### 2.1 Replace Puppeteer with Direct WebSocket

**Current Approach (Puppeteer):**
- Launches full Chromium instance (~200-400MB RAM)
- Navigates to map.blitzortung.org
- Intercepts WebSocket frames via CDP
- Decodes and forwards to clients

**Proposed Approach (Direct WebSocket):**
- Single WebSocket connection (~5-10MB RAM)
- No browser overhead
- Direct protocol implementation
- Faster startup, lower latency

**Implementation:**

```typescript
// server/blitzortung_client.ts
import WebSocket from 'ws';
import { decode } from './utils';

const SERVERS = [
  'wss://ws1.blitzortung.org:3000/',
  'wss://ws5.blitzortung.org:3000/',
  'wss://ws6.blitzortung.org:3000/',
  'wss://ws7.blitzortung.org:3000/',
];

interface BlitzortungConfig {
  onStrike: (strike: RawStrike) => void;
  onConnect: () => void;
  onDisconnect: () => void;
  onError: (error: Error) => void;
}

interface RawStrike {
  time: number;      // nanoseconds since epoch
  lat: number;
  lon: number;
  alt: number;       // meters
  pol: number;       // -1 or +1
  mds: number;       // deviation (ns)
  mcg: number;       // circular gap (degrees)
  delay: number;     // processing delay (s)
  region: number;    // 1-5
  sig: StationSignal[];
}

interface StationSignal {
  sta: string;
  time: number;
  lat: number;
  lon: number;
  alt: number;
  status: number;
}

export function createBlitzortungClient(config: BlitzortungConfig) {
  let ws: WebSocket | null = null;
  let reconnectAttempts = 0;
  let isShuttingDown = false;

  const MAX_RECONNECT_ATTEMPTS = 10;
  const BASE_RECONNECT_DELAY = 1000;
  const MAX_RECONNECT_DELAY = 60000;

  function getRandomServer(): string {
    return SERVERS[Math.floor(Math.random() * SERVERS.length)];
  }

  function getReconnectDelay(): number {
    const delay = Math.min(
      BASE_RECONNECT_DELAY * Math.pow(2, reconnectAttempts),
      MAX_RECONNECT_DELAY
    );
    // Add jitter (0-25%)
    return delay + Math.random() * delay * 0.25;
  }

  function connect() {
    if (isShuttingDown) return;

    const serverUrl = getRandomServer();
    console.log(`Connecting to ${serverUrl}...`);

    ws = new WebSocket(serverUrl);

    ws.on('open', () => {
      console.log('Connected to Blitzortung');
      reconnectAttempts = 0;

      // Send subscription message
      ws?.send(JSON.stringify({ time: 0 }));
      config.onConnect();
    });

    ws.on('message', (data: Buffer) => {
      try {
        const decoded = decode(data.toString());
        const strike = JSON.parse(decoded) as RawStrike;

        if (strike.lat !== undefined && strike.lon !== undefined) {
          config.onStrike(strike);
        }
      } catch (err) {
        // Some messages may not be strike data
      }
    });

    ws.on('close', () => {
      console.log('Disconnected from Blitzortung');
      config.onDisconnect();
      scheduleReconnect();
    });

    ws.on('error', (err) => {
      console.error('WebSocket error:', err.message);
      config.onError(err);
    });
  }

  function scheduleReconnect() {
    if (isShuttingDown) return;
    if (reconnectAttempts >= MAX_RECONNECT_ATTEMPTS) {
      console.error('Max reconnection attempts reached');
      return;
    }

    reconnectAttempts++;
    const delay = getReconnectDelay();
    console.log(`Reconnecting in ${Math.round(delay)}ms (attempt ${reconnectAttempts})`);

    setTimeout(connect, delay);
  }

  function disconnect() {
    isShuttingDown = true;
    ws?.close();
  }

  return { connect, disconnect };
}
```

### 2.2 Enhanced Strike Processing

Transform raw Blitzortung data into a richer internal format:

```typescript
// server/strike_processor.ts
interface ProcessedStrike {
  id: string;
  timestamp: number;          // ms since epoch
  lat: number;
  lng: number;
  altitude: number;           // meters (0 = unknown/cloud-to-ground assumed)
  polarity: 'positive' | 'negative' | 'unknown';
  intensity: number;          // 0-1 normalized from station count + signal quality
  quality: number;            // 0-1 based on mds and mcg
  region: string;             // human-readable
  stationCount: number;       // number of detecting stations
  delay: number;              // processing delay in seconds
}

const REGIONS: Record<number, string> = {
  1: 'europe',
  2: 'oceania',
  3: 'north_america',
  4: 'asia',
  5: 'south_america',
};

export function processStrike(raw: RawStrike): ProcessedStrike {
  const stationCount = raw.sig?.length ?? 0;

  // Normalize intensity: more stations = higher confidence = brighter bolt
  // Typical range is 5-50 stations; clamp to 0-1
  const intensity = Math.min(1, Math.max(0, (stationCount - 3) / 30));

  // Quality based on deviation and gap
  // Lower mds (nanoseconds) = more precise timing = higher quality
  // Lower mcg (degrees) = better station distribution = higher quality
  const mdsQuality = Math.max(0, 1 - (raw.mds / 1000000000)); // 1s = 0 quality
  const mcgQuality = Math.max(0, 1 - (raw.mcg / 180));        // 180deg = 0 quality
  const quality = (mdsQuality + mcgQuality) / 2;

  return {
    id: `strike-${raw.time}-${Math.random().toString(36).substr(2, 6)}`,
    timestamp: Math.floor(raw.time / 1000000), // ns -> ms
    lat: raw.lat,
    lng: raw.lon,
    altitude: raw.alt ?? 0,
    polarity: raw.pol === 1 ? 'positive' : raw.pol === -1 ? 'negative' : 'unknown',
    intensity,
    quality,
    region: REGIONS[raw.region] ?? 'unknown',
    stationCount,
    delay: raw.delay,
  };
}
```

### 2.3 Rate Limiting & Validation

```typescript
// server/rate_limiter.ts
interface RateLimiterConfig {
  maxStrikesPerSecond: number;
  maxClientsPerIP: number;
  burstAllowance: number;
}

export function createRateLimiter(config: RateLimiterConfig) {
  const clientTokens = new Map<string, { tokens: number; lastRefill: number }>();
  const connectionsByIP = new Map<string, number>();

  function canAcceptConnection(ip: string): boolean {
    const count = connectionsByIP.get(ip) ?? 0;
    return count < config.maxClientsPerIP;
  }

  function registerConnection(ip: string) {
    connectionsByIP.set(ip, (connectionsByIP.get(ip) ?? 0) + 1);
  }

  function unregisterConnection(ip: string) {
    const count = connectionsByIP.get(ip) ?? 1;
    if (count <= 1) {
      connectionsByIP.delete(ip);
    } else {
      connectionsByIP.set(ip, count - 1);
    }
  }

  // Token bucket for strike throughput
  function shouldThrottle(): boolean {
    // Implementation for global strike throughput limiting
    // Useful if Blitzortung sends bursts during active storms
    return false;
  }

  return {
    canAcceptConnection,
    registerConnection,
    unregisterConnection,
    shouldThrottle,
  };
}
```

### 2.4 Health Check Endpoint

```typescript
// Add to server.js
let lastStrikeTime: number | null = null;
let strikeCount = 0;
let connectionState = 'disconnected';

const healthServer = http.createServer((req, res) => {
  if (req.url === '/health') {
    const health = {
      status: connectionState === 'connected' ? 'healthy' : 'degraded',
      uptime: process.uptime(),
      blitzortung: {
        connected: connectionState === 'connected',
        lastStrike: lastStrikeTime ? new Date(lastStrikeTime).toISOString() : null,
        strikesReceived: strikeCount,
      },
      clients: {
        connected: wss.clients.size,
      },
      memory: process.memoryUsage(),
    };

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(health, null, 2));
  } else {
    res.writeHead(404);
    res.end('Not Found');
  }
});
```

### 2.5 Updated Dependencies

```json
{
  "dependencies": {
    "dotenv": "^16.5.0",
    "ws": "^8.13.0"
  },
  "devDependencies": {
    "@types/ws": "^8.5.10",
    "typescript": "^5.3.0"
  }
}
```

Remove Puppeteer dependency entirely (~400MB saved).

---

## 3. Enhanced Strike Data Model

### 3.1 Updated Interface

```typescript
// shared/types/LightningStrike.ts
export interface LightningStrike {
  id: string;
  lat: number;
  lng: number;
  timestamp: number;        // When strike occurred (ms)
  createdAt: number;        // When client received it (ms)

  // New fields
  altitude: number;         // meters above sea level (0 = ground level assumed)
  polarity: 'positive' | 'negative' | 'unknown';
  intensity: number;        // 0-1, affects visual brightness/size
  quality: number;          // 0-1, data confidence
  region: string;           // geographic region name
  stationCount: number;     // detecting stations (for debugging/display)
}

export interface StrikeVisualParams {
  intensity: number;        // Base brightness
  boltType: 'cg' | 'cc';    // Cloud-to-ground vs cloud-to-cloud
  polarity: 'positive' | 'negative';
}

export function deriveVisualParams(strike: LightningStrike): StrikeVisualParams {
  return {
    intensity: strike.intensity * strike.quality,
    boltType: strike.altitude > 0 ? 'cc' : 'cg',
    polarity: strike.polarity === 'unknown' ? 'negative' : strike.polarity,
  };
}
```

### 3.2 Visual Mapping

| Data Field | Visual Effect |
|------------|---------------|
| `intensity` | Bolt brightness, glow radius |
| `polarity: positive` | Warmer color tint (orange-white), thicker bolt |
| `polarity: negative` | Cooler color tint (blue-white), standard bolt |
| `altitude > 0` | Cloud-to-cloud: no ground termination, more horizontal spread |
| `altitude = 0` | Cloud-to-ground: vertical bolt to surface |
| `quality` | Modulates overall opacity/visibility |

---

## 4. Client-Side Data Handling

### 4.1 Enhanced WebSocket Hook

```typescript
// client/src/services/dataStreams/hooks/useWebSocket.ts
export interface WebSocketConfig {
  url: string;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  onConnectionChange?: (connected: boolean) => void;
}

export interface UseWebSocketResult {
  connected: boolean;
  connecting: boolean;
  lastUpdate: string;
  reconnectAttempt: number;
  maxReconnectAttempts: number;
  send: (data: any) => void;
  subscribe: (callback: (data: any) => void) => () => void;
  manualReconnect: () => void;
}

export function useWebSocket(config: WebSocketConfig): UseWebSocketResult {
  // Exponential backoff with jitter
  const getBackoffDelay = (attempt: number) => {
    const base = config.reconnectInterval ?? 1000;
    const max = 30000;
    const delay = Math.min(base * Math.pow(2, attempt), max);
    return delay + Math.random() * delay * 0.25;
  };

  // ... existing implementation with these additions:
  // - Track connecting state
  // - Track reconnectAttempt count
  // - Expose manualReconnect for user-triggered retry
  // - Call onConnectionChange callback
}
```

### 4.2 Connection Status UI

```typescript
// client/src/components/ui/ConnectionStatus.tsx
interface ConnectionStatusProps {
  connected: boolean;
  connecting: boolean;
  reconnectAttempt: number;
  maxAttempts: number;
  onReconnect: () => void;
}

export function ConnectionStatus({
  connected,
  connecting,
  reconnectAttempt,
  maxAttempts,
  onReconnect,
}: ConnectionStatusProps) {
  if (connected) {
    return <StatusIndicator status="connected" label="Live" />;
  }

  if (connecting) {
    return <StatusIndicator status="connecting" label="Connecting..." />;
  }

  if (reconnectAttempt < maxAttempts) {
    return (
      <StatusIndicator
        status="reconnecting"
        label={`Reconnecting (${reconnectAttempt}/${maxAttempts})...`}
      />
    );
  }

  return (
    <StatusIndicator
      status="disconnected"
      label="Disconnected"
      action={<button onClick={onReconnect}>Retry</button>}
    />
  );
}
```

### 4.3 Demo/Offline Mode

```typescript
// client/src/services/dataStreams/demoDataStream.ts
import { DataStream, LightningStrike } from '../interfaces';

// Storm activity regions with realistic coordinates
const STORM_CENTERS = [
  { lat: 35.5, lng: -95.0, name: 'Central US' },
  { lat: -23.5, lng: -46.5, name: 'Sao Paulo' },
  { lat: 51.5, lng: 10.0, name: 'Central Europe' },
  { lat: 1.0, lng: 110.0, name: 'Borneo' },
  { lat: -12.0, lng: 130.0, name: 'Northern Australia' },
];

export function createDemoDataStream(): DataStream<LightningStrike> {
  let subscribers = new Set<(data: LightningStrike) => void>();
  let intervalId: number | null = null;
  let connected = false;

  function generateStrike(): LightningStrike {
    const center = STORM_CENTERS[Math.floor(Math.random() * STORM_CENTERS.length)];
    const spread = 5 + Math.random() * 10;

    return {
      id: `demo-${Date.now()}-${Math.random().toString(36).substr(2, 6)}`,
      lat: center.lat + (Math.random() - 0.5) * spread,
      lng: center.lng + (Math.random() - 0.5) * spread,
      timestamp: Date.now(),
      createdAt: Date.now(),
      altitude: Math.random() < 0.1 ? 5000 + Math.random() * 5000 : 0,
      polarity: Math.random() < 0.1 ? 'positive' : 'negative',
      intensity: 0.3 + Math.random() * 0.7,
      quality: 0.7 + Math.random() * 0.3,
      region: center.name,
      stationCount: 5 + Math.floor(Math.random() * 25),
    };
  }

  function start() {
    if (intervalId) return;

    // Variable rate: 1-5 strikes per second
    const emit = () => {
      const strike = generateStrike();
      subscribers.forEach(cb => cb(strike));

      const delay = 200 + Math.random() * 800;
      intervalId = window.setTimeout(emit, delay);
    };

    emit();
    connected = true;
  }

  function stop() {
    if (intervalId) {
      clearTimeout(intervalId);
      intervalId = null;
    }
    connected = false;
  }

  return {
    subscribe(callback) {
      subscribers.add(callback);
      if (subscribers.size === 1) start();
      return () => {
        subscribers.delete(callback);
        if (subscribers.size === 0) stop();
      };
    },
    connect: start,
    disconnect: stop,
    isConnected: () => connected,
    getLastUpdate: () => 'Demo Mode',
  };
}
```

### 4.4 Automatic Fallback

```typescript
// client/src/services/dataStreams/hooks/useLightningData.ts
export function useLightningData({ url, enableDemoFallback = true }: Config) {
  const wsResult = useWebSocket({ url });
  const [usingDemo, setUsingDemo] = useState(false);
  const demoStreamRef = useRef<DataStream<LightningStrike> | null>(null);

  useEffect(() => {
    // Switch to demo mode if connection fails after max attempts
    if (!wsResult.connected &&
        wsResult.reconnectAttempt >= wsResult.maxReconnectAttempts &&
        enableDemoFallback) {
      console.log('Switching to demo mode');
      setUsingDemo(true);

      if (!demoStreamRef.current) {
        demoStreamRef.current = createDemoDataStream();
      }
    }
  }, [wsResult.connected, wsResult.reconnectAttempt]);

  // Return appropriate stream based on mode
  // ...
}
```

### 4.5 Strike Batching

```typescript
// client/src/services/dataStreams/strikeBatcher.ts
interface BatchConfig {
  maxBatchSize: number;      // Max strikes per batch
  maxBatchInterval: number;  // Max ms to wait before flushing
  maxStrikesPerSecond: number;
}

export function createStrikeBatcher(
  config: BatchConfig,
  onBatch: (strikes: LightningStrike[]) => void
) {
  let buffer: LightningStrike[] = [];
  let flushTimeout: number | null = null;
  let lastFlush = 0;
  let droppedCount = 0;

  function add(strike: LightningStrike) {
    const now = Date.now();
    const elapsed = now - lastFlush;
    const currentRate = buffer.length / (elapsed / 1000);

    // Rate limiting: drop strikes if we're receiving too fast
    if (currentRate > config.maxStrikesPerSecond && buffer.length > 10) {
      droppedCount++;
      return;
    }

    buffer.push(strike);

    if (buffer.length >= config.maxBatchSize) {
      flush();
    } else if (!flushTimeout) {
      flushTimeout = window.setTimeout(flush, config.maxBatchInterval);
    }
  }

  function flush() {
    if (flushTimeout) {
      clearTimeout(flushTimeout);
      flushTimeout = null;
    }

    if (buffer.length > 0) {
      onBatch([...buffer]);
      buffer = [];
      lastFlush = Date.now();
    }
  }

  return { add, flush, getDroppedCount: () => droppedCount };
}
```

---

## 5. Testing Strategy

### 5.1 Mock Blitzortung Server

```typescript
// server/test/mock_blitzortung.ts
import { WebSocketServer } from 'ws';

export function createMockBlitzortungServer(port = 3100) {
  const wss = new WebSocketServer({ port });
  let interval: NodeJS.Timeout | null = null;

  wss.on('connection', (ws) => {
    ws.on('message', (msg) => {
      const data = JSON.parse(msg.toString());
      if (data.time !== undefined) {
        // Start sending strikes
        interval = setInterval(() => {
          const strike = generateMockStrike();
          ws.send(encode(JSON.stringify(strike))); // Use LZW encode
        }, 200 + Math.random() * 300);
      }
    });

    ws.on('close', () => {
      if (interval) clearInterval(interval);
    });
  });

  return {
    close: () => wss.close(),
    address: () => `ws://localhost:${port}/`,
  };
}

function generateMockStrike() {
  return {
    time: Date.now() * 1000000, // ns
    lat: -90 + Math.random() * 180,
    lon: -180 + Math.random() * 360,
    alt: Math.random() < 0.1 ? Math.random() * 10000 : 0,
    pol: Math.random() < 0.1 ? 1 : -1,
    mds: Math.random() * 100000,
    mcg: Math.random() * 60,
    delay: Math.random() * 5,
    region: Math.floor(Math.random() * 5) + 1,
    sig: Array.from({ length: 5 + Math.floor(Math.random() * 20) }, (_, i) => ({
      sta: `STA${i}`,
      time: Math.random() * 1000,
      lat: 0,
      lon: 0,
      alt: 0,
      status: 0,
    })),
  };
}
```

### 5.2 Integration Tests

```typescript
// server/test/integration.test.ts
describe('Blitzortung Client', () => {
  let mockServer: MockBlitzortungServer;
  let client: BlitzortungClient;

  beforeEach(() => {
    mockServer = createMockBlitzortungServer(3100);
    client = createBlitzortungClient({
      serverUrl: mockServer.address(),
      // ...
    });
  });

  afterEach(() => {
    client.disconnect();
    mockServer.close();
  });

  test('connects and receives strikes', async () => {
    const strikes: Strike[] = [];
    client.onStrike = (s) => strikes.push(s);

    client.connect();
    await waitFor(() => strikes.length >= 5, { timeout: 5000 });

    expect(strikes.length).toBeGreaterThanOrEqual(5);
    expect(strikes[0]).toHaveProperty('lat');
    expect(strikes[0]).toHaveProperty('lng');
  });

  test('reconnects after disconnect', async () => {
    const connections: boolean[] = [];
    client.onConnect = () => connections.push(true);
    client.onDisconnect = () => connections.push(false);

    client.connect();
    await waitFor(() => connections.includes(true));

    mockServer.close();
    await waitFor(() => connections.filter(c => !c).length >= 1);

    mockServer = createMockBlitzortungServer(3100);
    await waitFor(() => connections.filter(c => c).length >= 2);
  });
});
```

### 5.3 Load Testing

```typescript
// server/test/load.test.ts
describe('Client Load Handling', () => {
  test('handles burst of 100 strikes/second', async () => {
    const mockServer = createMockBlitzortungServer(3100, {
      strikeRate: 100 // per second
    });

    const processedStrikes: Strike[] = [];
    const droppedStrikes: number[] = [];

    // Run for 10 seconds
    await runFor(10000, () => {
      // Collect metrics
    });

    // Assertions about throughput, memory, latency
  });
});
```

---

## 6. Alternative Data Sources

### 6.1 Research Summary

| Source | Availability | Coverage | Latency | Cost |
|--------|--------------|----------|---------|------|
| Blitzortung.org | Public (WebSocket) | Global, crowd-sourced | ~2-5s | Free |
| WWLLN | Academic only | Global | Minutes | Subscription |
| GLD360 (Vaisala) | Commercial | Global, high accuracy | Real-time | $$$ |
| ENTLN | Commercial | N. America | Real-time | $$$ |
| NASA EONET | Public API | Event-based, not real-time | Hours-days | Free |
| Open-Meteo | API | Weather, not lightning | Real-time | Free tier |

### 6.2 Recommended Fallback Strategy

1. **Primary**: Blitzortung WebSocket (current approach, improved)
2. **Secondary**: Demo mode with realistic synthetic data
3. **Future consideration**: Cache Blitzortung data in a database for replay/historical views

### 6.3 Protocol Resilience

If Blitzortung changes their protocol:

1. **Detection**: Health check endpoint shows `lastStrike: null` for extended period
2. **Alert**: Log warnings when decode fails consistently
3. **Fallback**: Automatic switch to demo mode after N failures
4. **Update path**: The decode function and message parsing are isolated in single files

---

## 7. Implementation Phases

### Phase 1: Direct WebSocket (Week 1)
- [ ] Create `blitzortung_client.ts` with direct WebSocket connection
- [ ] Implement exponential backoff reconnection
- [ ] Add health check endpoint
- [ ] Remove Puppeteer dependency
- [ ] Test against live Blitzortung servers

### Phase 2: Enhanced Data (Week 1-2)
- [ ] Expand strike processing to capture all fields
- [ ] Update `LightningStrike` interface (both server and client)
- [ ] Wire intensity/polarity to visual parameters
- [ ] Add region filtering option (future: user can filter by region)

### Phase 3: Client Resilience (Week 2)
- [ ] Implement connection status UI
- [ ] Add demo/offline mode
- [ ] Implement strike batching
- [ ] Add manual reconnect button

### Phase 4: Testing & Polish (Week 2-3)
- [ ] Create mock Blitzortung server for testing
- [ ] Write integration tests
- [ ] Load test with high strike rates
- [ ] Document API for other potential consumers

---

## 8. Open Questions

1. **Rate Limiting Policy**: Should we limit client connections per IP? What's the expected user count?

2. **Historical Data**: Should we persist strikes to a database for replay/historical visualization?

3. **Region Filtering**: Allow users to subscribe to specific regions only? (Reduces data volume)

4. **Strike Deduplication**: Blitzortung sometimes sends duplicate strikes with slight variations. Dedupe on server or client?

5. **Altitude Threshold**: What altitude distinguishes cloud-to-cloud from cloud-to-ground for visualization? (Proposed: >1000m = CC)

---

## 9. References

- [Blitzortung Live Data Documentation](https://www.limaps.org/live-data.html)
- [SimonSchick/BlitzortungAPI (TypeScript)](https://github.com/SimonSchick/BlitzortungAPI)
- [blitzortung Rust crate](https://docs.rs/blitzortung)
- [Home Assistant Blitzortung Integration](https://github.com/mrk-its/homeassistant-blitzortung)
- [Blitzortung.org Official Map](https://map.blitzortung.org/)
