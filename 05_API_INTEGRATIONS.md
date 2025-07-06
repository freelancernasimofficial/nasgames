# 🎮 API Integrations & Game Providers
## Master Account Architecture for White Label Clients

---

## 🎯 Master Account Strategy

### Single Credential Architecture
**Use ONE set of API credentials to serve ALL white label clients** through sub-account isolation.

```
NasGames.com Master Accounts:
├── JDB Master: DC="ARIF" (serves all clients)
├── LuckySports Master: ID="smt-arif-subt-880r657" (serves all clients)
└── Client Isolation: Player ID prefixing system
```

### Cost Benefits
```
Traditional Approach (Per Client):
├── JDB Individual Account: $500-1,000/month per client
├── LuckySports Individual: $300-800/month per client
├── 10 clients = $8,000-18,000/month in API costs

Master Account Approach:
├── JDB Master Account: $3,000/month (all clients)
├── LuckySports Master: $2,000/month (all clients)
├── 10 clients = $5,000/month total
├── SAVINGS: $3,000-13,000/month
```

---

## 🏗️ Database Schema for Master Account

### Game Provider Players Mapping
```sql
-- Maps internal users to game provider player IDs
CREATE TABLE game_provider_players (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    provider_id INT NOT NULL,
    
    -- Provider-specific player ID (with client prefix)
    provider_player_id VARCHAR(100) NOT NULL, -- "ARIF_88TAKA_user123"
    provider_username VARCHAR(100), -- Provider internal username
    
    -- Account status
    status ENUM('active', 'suspended', 'closed') DEFAULT 'active',
    balance DECIMAL(15,2) DEFAULT 0.00, -- Provider-side balance
    
    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (provider_id) REFERENCES game_providers(id) ON DELETE CASCADE,
    
    -- Indexes
    UNIQUE KEY unique_provider_player (provider_id, provider_player_id),
    UNIQUE KEY unique_client_user_provider (client_id, user_id, provider_id),
    INDEX idx_client_provider (client_id, provider_id),
    INDEX idx_provider_player_id (provider_player_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Master Account Credentials
```sql
-- Store master account credentials (encrypted)
CREATE TABLE master_api_credentials (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_id INT NOT NULL,
    
    -- Credential details
    credential_type ENUM('master_account', 'individual_account') DEFAULT 'master_account',
    api_endpoint VARCHAR(500) NOT NULL,
    
    -- Encrypted credentials (JSON format)
    credentials JSON NOT NULL, -- {"dc": "ARIF", "key": "encrypted_key", "iv": "encrypted_iv"}
    
    -- Usage tracking
    total_clients_served INT DEFAULT 0,
    monthly_cost DECIMAL(10,2) DEFAULT 0.00,
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (provider_id) REFERENCES game_providers(id) ON DELETE CASCADE,
    INDEX idx_provider_active (provider_id, is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 💻 Implementation Code

### JDB Master Account Service
```typescript
// jdb-master.service.ts
export class JDBMasterService {
  private masterCredentials = {
    dc: "ARIF", // Your existing DC
    key: process.env.JDB_KEY, // "2b4e390639e91c9e"
    iv: process.env.JDB_IV,   // "d0f183cf4254ac53"
    apiUrl: "https://api.jdb1688.net"
  };

  // Create client-specific player ID
  private generatePlayerID(clientCode: string, userId: string): string {
    return `${this.masterCredentials.dc}_${clientCode}_${userId}`;
  }

  // Create player account for any client
  async createPlayerAccount(clientId: number, userId: string): Promise<string> {
    const client = await WhiteLabelClient.findByPk(clientId);
    const playerId = this.generatePlayerID(client.clientCode, userId);
    
    const playerData = {
      dc: this.masterCredentials.dc,
      playerId: playerId,
      currency: client.currency || 'USD'
    };

    const encryptedData = this.encrypt(JSON.stringify(playerData));
    const response = await axios.post(`${this.masterCredentials.apiUrl}/player/create`, {
      dc: this.masterCredentials.dc,
      data: encryptedData
    });

    // Store mapping in database
    await GameProviderPlayer.create({
      clientId,
      userId,
      providerId: 1, // JDB provider ID
      providerPlayerId: playerId,
      status: 'active'
    });

    return playerId;
  }

  // Launch game for any client's user
  async launchGame(clientId: number, userId: string, gameCode: string): Promise<string> {
    const playerMapping = await GameProviderPlayer.findOne({
      where: { clientId, userId, providerId: 1 }
    });

    if (!playerMapping) {
      // Create player account if doesn't exist
      await this.createPlayerAccount(clientId, userId);
    }

    const gameData = {
      dc: this.masterCredentials.dc,
      playerId: playerMapping.providerPlayerId,
      gameCode: gameCode
    };

    const encryptedData = this.encrypt(JSON.stringify(gameData));
    const response = await axios.post(`${this.masterCredentials.apiUrl}/game/launch`, {
      dc: this.masterCredentials.dc,
      data: encryptedData
    });

    return response.data.gameUrl;
  }
}
```

### LuckySports Master Account Service
```typescript
// luckysports-master.service.ts
export class LuckySportsMasterService {
  private masterCredentials = {
    id: "smt-arif-subt-880r657", // Your existing ID
    username: "880r657@merchant.uni247.xyz",
    apiKey: process.env.LUCKYSPORTS_API_KEY,
    backendUrl: "https://smt-arif-subt-880r657.uni247.online/"
  };

  // Create client-specific user ID
  private generateUserID(clientCode: string, userId: string): string {
    return `${clientCode}_${userId}`;
  }

  // Create sports betting account for any client
  async createSportsUser(clientId: number, userId: string): Promise<string> {
    const client = await WhiteLabelClient.findByPk(clientId);
    const sportsUserId = this.generateUserID(client.clientCode, userId);
    
    const userData = {
      userId: sportsUserId,
      currency: client.currency || 'USD',
      parentAccount: this.masterCredentials.id
    };

    const response = await axios.post(`${this.masterCredentials.backendUrl}/api/user/create`, userData, {
      headers: {
        'Authorization': `Bearer ${this.masterCredentials.apiKey}`
      }
    });

    // Store mapping
    await GameProviderPlayer.create({
      clientId,
      userId,
      providerId: 2, // LuckySports provider ID
      providerPlayerId: sportsUserId,
      status: 'active'
    });

    return sportsUserId;
  }
}
```

---

## 📊 Client Isolation Examples

### Player ID Structure
```
Master Account: "ARIF"
├── Client 88taka (ARIF_88TAKA):
│   ├── User 1: ARIF_88TAKA_user001
│   ├── User 2: ARIF_88TAKA_user002
│   └── User N: ARIF_88TAKA_userN
├── Client NewCasino (ARIF_NEWCASINO):
│   ├── User 1: ARIF_NEWCASINO_user001
│   └── User N: ARIF_NEWCASINO_userN
└── Client ThirdSite (ARIF_THIRDSITE):
    └── Users: ARIF_THIRDSITE_userN
```

### Database Query Examples
```sql
-- Get all JDB players for a specific client
SELECT gpp.*, u.username, u.balance
FROM game_provider_players gpp
JOIN users u ON gpp.user_id = u.id
WHERE gpp.client_id = 1 AND gpp.provider_id = 1; -- JDB

-- Get provider balance for a specific user
SELECT provider_player_id, balance, status
FROM game_provider_players
WHERE client_id = 1 AND user_id = 123 AND provider_id = 1;

-- Track total players per client per provider
SELECT 
    c.company_name,
    gp.provider_name,
    COUNT(gpp.id) as total_players,
    SUM(gpp.balance) as total_balance
FROM game_provider_players gpp
JOIN white_label_clients c ON gpp.client_id = c.id
JOIN game_providers gp ON gpp.provider_id = gp.id
GROUP BY c.id, gp.id;
```

---

## 💰 Revenue Tracking with Master Account

### Commission Calculation
```typescript
// Calculate commission from master account GGR
export class MasterAccountCommissionService {
  static async calculateClientCommission(clientId: number, period: DateRange) {
    // Get all game sessions for client in period
    const sessions = await GameSession.findAll({
      where: {
        clientId,
        startedAt: { [Op.between]: [period.start, period.end] },
        status: 'completed'
      }
    });

    const totalBets = sessions.reduce((sum, s) => sum + s.betAmount, 0);
    const totalWins = sessions.reduce((sum, s) => sum + s.winAmount, 0);
    const grossRevenue = totalBets - totalWins;

    // Master account fees (already paid by platform)
    const providerFees = grossRevenue * 0.15; // 15% to JDB/LuckySports
    const netRevenue = grossRevenue - providerFees;

    // Client revenue share
    const client = await WhiteLabelClient.findByPk(clientId);
    const clientShare = netRevenue * (client.revenueSharePercentage / 100);
    const platformCommission = netRevenue - clientShare;

    return {
      grossRevenue,
      providerFees,
      netRevenue,
      clientShare,
      platformCommission
    };
  }
}
```

---

## 🔧 Migration Script

### Convert Existing Setup to Master Account
```sql
-- Insert master account credentials
INSERT INTO master_api_credentials (provider_id, credential_type, api_endpoint, credentials, monthly_cost) VALUES
(1, 'master_account', 'https://api.jdb1688.net', 
 JSON_OBJECT('dc', 'ARIF', 'key', 'encrypted_key', 'iv', 'encrypted_iv'), 3000.00),
(2, 'master_account', 'https://smt-arif-subt-880r657.uni247.online/', 
 JSON_OBJECT('id', 'smt-arif-subt-880r657', 'apiKey', 'encrypted_key'), 2000.00);

-- Create player mappings for existing users
INSERT INTO game_provider_players (client_id, user_id, provider_id, provider_player_id, status)
SELECT 
    u.client_id,
    u.id,
    1, -- JDB provider
    CONCAT('ARIF_', c.client_code, '_', u.id),
    'active'
FROM users u
JOIN white_label_clients c ON u.client_id = c.id
WHERE u.user_type = 'player';
```

---

## 🎯 Benefits Summary

### Cost Savings
- **87% reduction** in API costs (vs individual accounts)
- **No additional purchases** needed for new clients
- **Immediate scalability** with existing credentials

### Technical Benefits
- **Single integration point** for all clients
- **Centralized credential management**
- **Simplified maintenance and updates**
- **Consistent API behavior** across clients

### Business Advantages
- **Faster client onboarding** (no API setup delays)
- **Higher profit margins** (lower provider costs)
- **Competitive pricing** for clients
- **Scalable growth model**

**This master account architecture maximizes profitability while minimizing complexity and costs.**