# 👥 User Management System
## 2-Table Architecture for Security & Hierarchy

---

## 🎯 User Management Overview

The system uses a **2-table architecture** for optimal security separation:
- **Platform Users**: NasGames.com staff (separate table)
- **Client Users**: All client-side users with hierarchy (single table)

---

## 🏗️ Database Architecture

### Platform Users Table
```sql
CREATE TABLE platform_users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    user_type ENUM('platform_owner', 'platform_admin', 'support_agent') NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    status ENUM('active', 'suspended', 'inactive') DEFAULT 'active',
    two_factor_enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_username (username),
    INDEX idx_user_type (user_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Client Users Table
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    user_type ENUM('super_admin', 'admin', 'master_agent', 'sub_agent', 'player') NOT NULL,
    parent_user_id BIGINT NULL,
    full_name VARCHAR(255) NOT NULL,
    balance DECIMAL(15,2) DEFAULT 0.00,
    status ENUM('active', 'suspended', 'inactive') DEFAULT 'active',
    is_independent_user BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_user_id) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_client_user_type (client_id, user_type),
    INDEX idx_parent_user (parent_user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 🔐 Authentication System

### Separate Login Systems
```typescript
// Platform login
app.post('/platform/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await PlatformUser.findOne({ where: { email } });
  
  if (user && await bcrypt.compare(password, user.passwordHash)) {
    const token = jwt.sign({ 
      userId: user.id, 
      userType: 'platform',
      role: user.userType 
    }, process.env.JWT_SECRET);
    
    res.json({ token, user: { ...user, passwordHash: undefined } });
  }
});

// Client login
app.post('/client/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ where: { email } });
  
  if (user && await bcrypt.compare(password, user.passwordHash)) {
    const token = jwt.sign({ 
      userId: user.id, 
      clientId: user.clientId,
      userType: 'client',
      role: user.userType 
    }, process.env.JWT_SECRET);
    
    res.json({ token, user: { ...user, passwordHash: undefined } });
  }
});
```

---

## 👑 User Hierarchy System

### Client User Hierarchy
```
Client (88taka.com):
├── ARIF (super_admin, parent_user_id=NULL)
│   ├── Admin1 (admin, parent_user_id=ARIF_ID)
│   │   └── MasterAgent1 (master_agent, parent_user_id=Admin1_ID)
│   │       └── SubAgent1 (sub_agent, parent_user_id=MasterAgent1_ID)
│   │           ├── Player1 (player, is_independent_user=FALSE)
│   │           └── Player2 (player, is_independent_user=FALSE)
│   └── IndependentPlayer (player, parent_user_id=ARIF_ID, is_independent_user=TRUE)
```

### Hierarchy Query
```sql
-- Get complete user hierarchy for a client
WITH RECURSIVE user_hierarchy AS (
    SELECT id, username, full_name, user_type, parent_user_id, 0 as level
    FROM users 
    WHERE client_id = ? AND parent_user_id IS NULL
    
    UNION ALL
    
    SELECT u.id, u.username, u.full_name, u.user_type, u.parent_user_id, uh.level + 1
    FROM users u
    INNER JOIN user_hierarchy uh ON u.parent_user_id = uh.id
    WHERE u.client_id = ?
)
SELECT * FROM user_hierarchy ORDER BY level, username;
```

---

## 🔑 Permission System

### Role-Based Permissions
```typescript
interface UserPermissions {
  platform_owner: string[];
  platform_admin: string[];
  support_agent: string[];
  super_admin: string[];
  admin: string[];
  master_agent: string[];
  sub_agent: string[];
  player: string[];
}

const PERMISSIONS: UserPermissions = {
  platform_owner: ['*'], // All permissions
  platform_admin: [
    'manage_clients', 'view_all_reports', 'manage_platform_settings',
    'impersonate_users', 'manage_game_providers'
  ],
  support_agent: [
    'view_client_data', 'assist_users', 'create_support_tickets'
  ],
  super_admin: [
    'manage_client_users', 'manage_client_settings', 'view_client_reports',
    'manage_balance', 'create_admins'
  ],
  admin: [
    'create_master_agents', 'manage_downline_users', 'view_reports',
    'manage_downline_balance'
  ],
  master_agent: [
    'create_sub_agents', 'manage_sub_agents', 'view_downline_reports'
  ],
  sub_agent: [
    'create_players', 'manage_players', 'manage_player_balance'
  ],
  player: [
    'play_games', 'view_own_data', 'deposit', 'withdraw'
  ]
};
```

### Permission Middleware
```typescript
const checkPermission = (requiredPermission: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = req.user;
    const userPermissions = PERMISSIONS[user.role] || [];
    
    if (userPermissions.includes('*') || userPermissions.includes(requiredPermission)) {
      next();
    } else {
      res.status(403).json({ error: 'Insufficient permissions' });
    }
  };
};
```

---

## 💰 Balance Management

### Balance Transfer System
```typescript
class BalanceService {
  static async transferBalance(fromUserId: string, toUserId: string, amount: number) {
    const transaction = await db.transaction();
    
    try {
      const fromUser = await User.findByPk(fromUserId, { lock: true, transaction });
      const toUser = await User.findByPk(toUserId, { lock: true, transaction });
      
      // Validate hierarchy (can only transfer to direct children)
      if (toUser.parentUserId !== fromUser.id) {
        throw new Error('Can only transfer to direct downline users');
      }
      
      if (fromUser.balance < amount) {
        throw new Error('Insufficient balance');
      }
      
      fromUser.balance -= amount;
      toUser.balance += amount;
      
      await fromUser.save({ transaction });
      await toUser.save({ transaction });
      
      // Record transactions
      await Transaction.bulkCreate([
        {
          clientId: fromUser.clientId,
          userId: fromUserId,
          transactionType: 'transfer_out',
          amount: -amount,
          balanceBefore: fromUser.balance + amount,
          balanceAfter: fromUser.balance,
          toUserId
        },
        {
          clientId: toUser.clientId,
          userId: toUserId,
          transactionType: 'transfer_in',
          amount: amount,
          balanceBefore: toUser.balance - amount,
          balanceAfter: toUser.balance,
          fromUserId
        }
      ], { transaction });
      
      await transaction.commit();
      return { success: true };
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}
```

---

## 🎯 User Creation Flow

### Create User with Hierarchy
```typescript
class UserService {
  static async createUser(creatorId: string, userData: CreateUserDto) {
    const creator = await User.findByPk(creatorId);
    
    // Validate creation permissions
    const canCreate = this.validateUserCreation(creator.userType, userData.userType);
    if (!canCreate) {
      throw new Error('Cannot create this user type');
    }
    
    // Set parent relationship
    const newUser = await User.create({
      ...userData,
      clientId: creator.clientId,
      parentUserId: creatorId,
      createdBy: creatorId
    });
    
    return newUser;
  }
  
  private static validateUserCreation(creatorType: string, targetType: string): boolean {
    const validCreations = {
      super_admin: ['admin', 'player'],
      admin: ['master_agent'],
      master_agent: ['sub_agent'],
      sub_agent: ['player']
    };
    
    return validCreations[creatorType]?.includes(targetType) || false;
  }
}
```

---

## 📊 User Analytics

### User Statistics
```sql
-- Platform user statistics
SELECT 
    user_type,
    COUNT(*) as total_users,
    COUNT(CASE WHEN status = 'active' THEN 1 END) as active_users
FROM platform_users
GROUP BY user_type;

-- Client user statistics
SELECT 
    c.company_name,
    u.user_type,
    COUNT(u.id) as total_users,
    SUM(u.balance) as total_balance
FROM white_label_clients c
LEFT JOIN users u ON c.id = u.client_id
GROUP BY c.id, c.company_name, u.user_type;

-- User hierarchy depth analysis
WITH RECURSIVE hierarchy_depth AS (
    SELECT id, username, user_type, parent_user_id, 0 as depth
    FROM users WHERE parent_user_id IS NULL
    
    UNION ALL
    
    SELECT u.id, u.username, u.user_type, u.parent_user_id, hd.depth + 1
    FROM users u
    JOIN hierarchy_depth hd ON u.parent_user_id = hd.id
)
SELECT 
    client_id,
    MAX(depth) as max_hierarchy_depth,
    COUNT(*) as total_users_in_hierarchy
FROM hierarchy_depth h
JOIN users u ON h.id = u.id
GROUP BY client_id;
```

---

## 🔒 Security Features

### User Access Control
```typescript
class SecurityService {
  // Platform admin impersonation for support
  static async impersonateUser(platformAdminId: number, targetUserId: string) {
    const platformAdmin = await PlatformUser.findByPk(platformAdminId);
    const targetUser = await User.findByPk(targetUserId);
    
    if (platformAdmin.userType === 'platform_admin' || platformAdmin.userType === 'platform_owner') {
      return jwt.sign({
        userId: targetUser.id,
        clientId: targetUser.clientId,
        userType: 'client',
        role: targetUser.userType,
        impersonatedBy: platformAdminId
      }, process.env.JWT_SECRET, { expiresIn: '1h' });
    }
    
    throw new Error('Unauthorized impersonation attempt');
  }
  
  // Client data isolation
  static async validateClientAccess(userId: string, requestedClientId: number) {
    const user = await User.findByPk(userId);
    
    if (user.clientId !== requestedClientId) {
      throw new Error('Access denied: Client data isolation violation');
    }
    
    return true;
  }
}
```

---

## 🎯 Benefits of 2-Table System

### Security Benefits
- ✅ **Zero data mixing** between platform and client users
- ✅ **Separate authentication** systems with different security levels
- ✅ **Clear access boundaries** preventing accidental data exposure
- ✅ **Simplified queries** without complex client_id filtering

### Management Benefits
- ✅ **Easy hierarchy management** within client users table
- ✅ **Simple balance transfers** within same table
- ✅ **Unified client user operations** 
- ✅ **Clear separation of concerns**

### Scalability Benefits
- ✅ **Independent scaling** of platform vs client operations
- ✅ **Optimized indexes** for each user type
- ✅ **Flexible permission systems** for different user categories
- ✅ **Future-proof architecture** for additional user types

**This 2-table architecture provides the perfect balance of security, simplicity, and scalability for the white label casino platform.**