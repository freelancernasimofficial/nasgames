# 🏗️ Technical Architecture
## NasGames.com White Label Casino Platform

---

## 🎯 Architecture Overview

The NasGames.com platform uses a **modern, scalable microservices architecture** designed for high availability, security, and rapid client onboarding. The system supports multiple white label clients with isolated environments while sharing core infrastructure.

---

## 🔧 Technology Stack

### Frontend Technologies
```
Primary Stack:
├── React.js 18+ with TypeScript
├── Next.js 14+ for SSR and optimization
├── Material-UI (MUI) for component library
├── Redux Toolkit for state management
├── React Query for server state management
├── Socket.io-client for real-time updates
├── Framer Motion for animations
└── PWA support for mobile experience

Development Tools:
├── Vite for fast development builds
├── ESLint + Prettier for code quality
├── Jest + React Testing Library for testing
└── Storybook for component documentation
```

### Backend Technologies
```
Core Backend:
├── Node.js 20+ with Express.js
├── TypeScript for type safety
├── Prisma ORM for database operations
├── JWT + Passport.js for authentication
├── Socket.io for real-time communication
├── Bull Queue for background jobs
├── Winston for logging
└── Helmet for security headers

API & Integration:
├── GraphQL with Apollo Server
├── REST APIs for external integrations
├── OpenAPI/Swagger for documentation
├── Rate limiting with express-rate-limit
└── CORS configuration for security
```

### Database & Storage
```
Primary Database:
├── MySQL 8.0+ (Primary database)
├── Redis 7+ (Caching and sessions)
├── MongoDB (Analytics and logs)
└── InfluxDB (Time-series metrics)

File Storage:
├── AWS S3 (Primary file storage)
├── CloudFront CDN (Content delivery)
└── Local storage (Development)
```

### Infrastructure & DevOps
```
Cloud Infrastructure:
├── AWS/DigitalOcean for hosting
├── Docker containers for deployment
├── Kubernetes for orchestration
├── Nginx for reverse proxy and load balancing
└── Let's Encrypt for SSL certificates

Monitoring & Analytics:
├── Prometheus for metrics collection
├── Grafana for visualization
├── ELK Stack (Elasticsearch, Logstash, Kibana)
├── Sentry for error tracking
└── New Relic for APM
```

---

## 🏛️ System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Load Balancer (Nginx)                    │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────────────┐
│                    API Gateway                                  │
│  ├── Authentication & Authorization                             │
│  ├── Rate Limiting & Throttling                                │
│  ├── Request Routing                                           │
│  └── API Documentation                                         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────────────┐
│                  Microservices Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│
│  │   User      │ │  Financial  │ │    Game     │ │   Client    ││
│  │  Service    │ │   Service   │ │  Service    │ │  Service    ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘│
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│
│  │ Notification│ │  Reporting  │ │   Audit     │ │   Payment   ││
│  │  Service    │ │   Service   │ │  Service    │ │  Service    ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘│
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────────────┐
│                    Data Layer                                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│
│  │    MySQL    │ │    Redis    │ │   MongoDB   │ │  InfluxDB   ││
│  │ (Primary)   │ │ (Cache)     │ │ (Analytics) │ │ (Metrics)   ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘│
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────────────┐
│                External Integrations                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐│
│  │ LuckySports │ │ JDB Games   │ │  Payment    │ │    SMS/     ││
│  │    API      │ │    API      │ │ Gateways    │ │   Email     ││
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Detailed Service Architecture

### 1. User Service
```typescript
interface UserService {
  // Core user management
  createUser(userData: CreateUserDto): Promise<User>;
  updateUser(id: string, updates: UpdateUserDto): Promise<User>;
  deleteUser(id: string): Promise<void>;
  getUserById(id: string): Promise<User>;
  getUsersByClient(clientId: string): Promise<User[]>;
  
  // Hierarchy management
  createUserHierarchy(parentId: string, userData: CreateUserDto): Promise<User>;
  getUserChildren(parentId: string): Promise<User[]>;
  getUserAncestors(userId: string): Promise<User[]>;
  
  // Authentication
  authenticateUser(credentials: LoginDto): Promise<AuthResult>;
  refreshToken(refreshToken: string): Promise<AuthResult>;
  logoutUser(userId: string): Promise<void>;
  
  // Permissions
  assignPermissions(userId: string, permissions: string[]): Promise<void>;
  checkPermission(userId: string, permission: string): Promise<boolean>;
}
```

### 2. Financial Service
```typescript
interface FinancialService {
  // Balance management
  getBalance(userId: string): Promise<Balance>;
  updateBalance(userId: string, amount: number, type: TransactionType): Promise<Transaction>;
  transferBalance(fromId: string, toId: string, amount: number): Promise<Transfer>;
  
  // Transaction management
  createTransaction(transactionData: CreateTransactionDto): Promise<Transaction>;
  getTransactionHistory(userId: string, filters: TransactionFilters): Promise<Transaction[]>;
  
  // Commission calculation
  calculateCommission(gameSession: GameSession): Promise<Commission>;
  processCommissions(clientId: string, period: DateRange): Promise<Commission[]>;
  
  // Revenue sharing
  calculateRevenueShare(clientId: string, period: DateRange): Promise<RevenueShare>;
  distributeRevenue(clientId: string, amount: number): Promise<void>;
}
```

### 3. Game Service
```typescript
interface GameService {
  // Game provider management
  registerProvider(providerData: GameProviderDto): Promise<GameProvider>;
  getAvailableGames(clientId: string): Promise<Game[]>;
  
  // Game session management
  launchGame(userId: string, gameId: string): Promise<GameSession>;
  endGameSession(sessionId: string, result: GameResult): Promise<void>;
  
  // Transfer wallet operations
  transferToProvider(userId: string, providerId: string, amount: number): Promise<Transfer>;
  transferFromProvider(userId: string, providerId: string, amount: number): Promise<Transfer>;
  getProviderBalance(userId: string, providerId: string): Promise<number>;
  
  // Game statistics
  getGameStats(gameId: string, period: DateRange): Promise<GameStats>;
  getUserGameHistory(userId: string): Promise<GameSession[]>;
}
```

### 4. Client Service
```typescript
interface ClientService {
  // Client management
  createClient(clientData: CreateClientDto): Promise<Client>;
  updateClient(clientId: string, updates: UpdateClientDto): Promise<Client>;
  getClientById(clientId: string): Promise<Client>;
  getAllClients(): Promise<Client[]>;
  
  // Client configuration
  updateClientSettings(clientId: string, settings: ClientSettings): Promise<void>;
  getClientSettings(clientId: string): Promise<ClientSettings>;
  
  // Client analytics
  getClientMetrics(clientId: string, period: DateRange): Promise<ClientMetrics>;
  getClientRevenue(clientId: string, period: DateRange): Promise<RevenueReport>;
}
```

---

## 🗄️ Database Architecture

### Multi-Tenant Strategy
```sql
-- Client isolation strategy
-- Option 1: Shared database with client_id column (Chosen approach)
-- Pros: Cost-effective, easier maintenance, shared resources
-- Cons: Requires careful query filtering, potential data leakage risk

-- Option 2: Separate database per client
-- Pros: Complete isolation, better security
-- Cons: Higher costs, complex maintenance, resource waste

-- Our Implementation: Hybrid approach
-- Core shared tables with client_id
-- Client-specific tables when needed
-- Row-level security for sensitive data
```

### Database Connection Strategy
```typescript
// Database connection pool configuration
const dbConfig = {
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '3306'),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  
  // Connection pool settings
  pool: {
    min: 5,
    max: 20,
    idle: 10000,
    acquire: 30000,
  },
  
  // Performance optimizations
  dialectOptions: {
    charset: 'utf8mb4',
    collate: 'utf8mb4_unicode_ci',
    supportBigNumbers: true,
    bigNumberStrings: true,
  },
  
  // Logging and monitoring
  logging: (sql: string, timing?: number) => {
    if (timing && timing > 1000) {
      logger.warn(`Slow query detected: ${timing}ms`, { sql });
    }
  },
};
```

---

## 🔐 Security Architecture

### Authentication & Authorization
```typescript
// JWT Token Structure
interface JWTPayload {
  userId: string;
  clientId: string;
  role: UserRole;
  permissions: string[];
  sessionId: string;
  iat: number;
  exp: number;
}

// Role-Based Access Control
class RBACMiddleware {
  static checkPermission(requiredPermission: string) {
    return (req: Request, res: Response, next: NextFunction) => {
      const user = req.user as AuthenticatedUser;
      
      if (!user.permissions.includes(requiredPermission)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }
      
      next();
    };
  }
  
  static checkClientAccess(req: Request, res: Response, next: NextFunction) {
    const user = req.user as AuthenticatedUser;
    const requestedClientId = req.params.clientId;
    
    // Super admin can access all clients
    if (user.role === 'super_admin') {
      return next();
    }
    
    // Regular users can only access their own client
    if (user.clientId !== requestedClientId) {
      return res.status(403).json({ error: 'Access denied' });
    }
    
    next();
  }
}
```

### Data Encryption
```typescript
// Encryption service for sensitive data
class EncryptionService {
  private static readonly algorithm = 'aes-256-gcm';
  private static readonly keyLength = 32;
  
  static encrypt(text: string, key: string): EncryptedData {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(this.algorithm, key);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
    };
  }
  
  static decrypt(encryptedData: EncryptedData, key: string): string {
    const decipher = crypto.createDecipher(this.algorithm, key);
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

---

## 🚀 Performance Optimization

### Caching Strategy
```typescript
// Multi-layer caching implementation
class CacheService {
  private redis: Redis;
  private memoryCache: NodeCache;
  
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.memoryCache = new NodeCache({ stdTTL: 300 }); // 5 minutes
  }
  
  async get<T>(key: string): Promise<T | null> {
    // Level 1: Memory cache (fastest)
    const memoryResult = this.memoryCache.get<T>(key);
    if (memoryResult) {
      return memoryResult;
    }
    
    // Level 2: Redis cache
    const redisResult = await this.redis.get(key);
    if (redisResult) {
      const parsed = JSON.parse(redisResult) as T;
      this.memoryCache.set(key, parsed);
      return parsed;
    }
    
    return null;
  }
  
  async set<T>(key: string, value: T, ttl: number = 3600): Promise<void> {
    // Set in both caches
    this.memoryCache.set(key, value, ttl);
    await this.redis.setex(key, ttl, JSON.stringify(value));
  }
  
  async invalidate(pattern: string): Promise<void> {
    // Clear memory cache
    this.memoryCache.flushAll();
    
    // Clear Redis cache
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}
```

### Database Optimization
```sql
-- Essential indexes for performance
-- User queries
CREATE INDEX idx_users_client_role ON users(client_id, role, status);
CREATE INDEX idx_users_parent_hierarchy ON users(parent_id, created_at);
CREATE INDEX idx_users_email_unique ON users(email);

-- Transaction queries
CREATE INDEX idx_transactions_user_date ON transactions(user_id, created_at DESC);
CREATE INDEX idx_transactions_client_type ON transactions(client_id, transaction_type, created_at);
CREATE INDEX idx_transactions_reference ON transactions(reference_id);

-- Game session queries
CREATE INDEX idx_game_sessions_user_status ON game_sessions(user_id, status, started_at);
CREATE INDEX idx_game_sessions_game_date ON game_sessions(game_id, started_at DESC);

-- Commission queries
CREATE INDEX idx_commissions_client_period ON commissions(client_id, period_start, period_end);
CREATE INDEX idx_commissions_user_status ON commissions(user_id, status);

-- Composite indexes for complex queries
CREATE INDEX idx_users_client_role_parent ON users(client_id, role, parent_id);
CREATE INDEX idx_transactions_user_type_date ON transactions(user_id, transaction_type, created_at DESC);
```

---

## 🔄 Real-time Communication

### WebSocket Implementation
```typescript
// Real-time service using Socket.io
class RealtimeService {
  private io: Server;
  
  constructor(server: http.Server) {
    this.io = new Server(server, {
      cors: {
        origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
        methods: ['GET', 'POST'],
      },
    });
    
    this.setupEventHandlers();
  }
  
  private setupEventHandlers(): void {
    this.io.on('connection', (socket: Socket) => {
      socket.on('join_client_room', (clientId: string) => {
        socket.join(`client_${clientId}`);
      });
      
      socket.on('join_user_room', (userId: string) => {
        socket.join(`user_${userId}`);
      });
      
      socket.on('disconnect', () => {
        console.log('Client disconnected:', socket.id);
      });
    });
  }
  
  // Emit balance updates
  emitBalanceUpdate(userId: string, newBalance: number): void {
    this.io.to(`user_${userId}`).emit('balance_update', {
      userId,
      balance: newBalance,
      timestamp: new Date(),
    });
  }
  
  // Emit game session updates
  emitGameUpdate(userId: string, gameData: GameSession): void {
    this.io.to(`user_${userId}`).emit('game_update', gameData);
  }
  
  // Emit system notifications
  emitNotification(clientId: string, notification: Notification): void {
    this.io.to(`client_${clientId}`).emit('notification', notification);
  }
}
```

---

## 📊 Monitoring & Observability

### Application Monitoring
```typescript
// Comprehensive monitoring setup
class MonitoringService {
  private prometheus: PrometheusRegistry;
  private metrics: {
    httpRequests: Counter;
    httpDuration: Histogram;
    activeUsers: Gauge;
    gamesSessions: Counter;
    balanceTransfers: Counter;
  };
  
  constructor() {
    this.prometheus = new PrometheusRegistry();
    this.setupMetrics();
  }
  
  private setupMetrics(): void {
    this.metrics = {
      httpRequests: new Counter({
        name: 'http_requests_total',
        help: 'Total number of HTTP requests',
        labelNames: ['method', 'route', 'status_code'],
        registers: [this.prometheus],
      }),
      
      httpDuration: new Histogram({
        name: 'http_request_duration_seconds',
        help: 'Duration of HTTP requests in seconds',
        labelNames: ['method', 'route'],
        registers: [this.prometheus],
      }),
      
      activeUsers: new Gauge({
        name: 'active_users_total',
        help: 'Number of currently active users',
        labelNames: ['client_id'],
        registers: [this.prometheus],
      }),
      
      gamesSessions: new Counter({
        name: 'game_sessions_total',
        help: 'Total number of game sessions',
        labelNames: ['client_id', 'game_type'],
        registers: [this.prometheus],
      }),
      
      balanceTransfers: new Counter({
        name: 'balance_transfers_total',
        help: 'Total number of balance transfers',
        labelNames: ['client_id', 'transfer_type'],
        registers: [this.prometheus],
      }),
    };
  }
  
  // Middleware to track HTTP requests
  trackHttpRequest() {
    return (req: Request, res: Response, next: NextFunction) => {
      const start = Date.now();
      
      res.on('finish', () => {
        const duration = (Date.now() - start) / 1000;
        
        this.metrics.httpRequests.inc({
          method: req.method,
          route: req.route?.path || req.path,
          status_code: res.statusCode.toString(),
        });
        
        this.metrics.httpDuration.observe(
          { method: req.method, route: req.route?.path || req.path },
          duration
        );
      });
      
      next();
    };
  }
}
```

---

## 🔧 Development Environment Setup

### Docker Configuration
```dockerfile
# Dockerfile for Node.js backend
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Build the application
RUN npm run build

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["npm", "start"]
```

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=mysql://user:password@mysql:3306/nasgames
      - REDIS_URL=redis://redis:6379
    depends_on:
      - mysql
      - redis
    volumes:
      - .:/app
      - /app/node_modules

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: nasgames
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

volumes:
  mysql_data:
  redis_data:
```

---

## 🚀 Deployment Strategy

### Production Deployment Pipeline
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run test
      - run: npm run lint
      - run: npm run type-check

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v3
        with:
          push: true
          tags: nasgames/platform:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/nasgames-app \
            app=nasgames/platform:${{ github.sha }}
          kubectl rollout status deployment/nasgames-app
```

---

**This technical architecture provides a solid foundation for building a scalable, secure, and maintainable white label casino platform that can grow with the business needs.**