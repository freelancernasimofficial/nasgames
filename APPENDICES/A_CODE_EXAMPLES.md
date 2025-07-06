# 💻 Code Examples
## Essential Implementation Samples

---

## 🔐 Authentication System

### JWT Service
```typescript
// auth.service.ts
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

export class AuthService {
  static generateToken(user: User): string {
    return jwt.sign(
      { 
        userId: user.id, 
        clientId: user.clientId, 
        role: user.userType 
      },
      process.env.JWT_SECRET!,
      { expiresIn: '24h' }
    );
  }

  static async hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, 10);
  }

  static async validatePassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }
}
```

### Auth Middleware
```typescript
// auth.middleware.ts
export const authenticateToken = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Access token required' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(403).json({ error: 'Invalid token' });
  }
};
```

---

## 💰 Financial System

### Balance Transfer
```typescript
// balance.service.ts
export class BalanceService {
  static async transferBalance(fromUserId: string, toUserId: string, amount: number) {
    const transaction = await db.transaction();
    
    try {
      const fromUser = await User.findByPk(fromUserId, { lock: true, transaction });
      const toUser = await User.findByPk(toUserId, { lock: true, transaction });

      if (fromUser.balance < amount) {
        throw new Error('Insufficient balance');
      }

      fromUser.balance -= amount;
      toUser.balance += amount;

      await fromUser.save({ transaction });
      await toUser.save({ transaction });

      await Transaction.create({
        userId: fromUserId,
        type: 'transfer_out',
        amount: -amount,
        balanceBefore: fromUser.balance + amount,
        balanceAfter: fromUser.balance,
        toUserId
      }, { transaction });

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

## 🎮 Game Integration

### JDB Service
```typescript
// jdb.service.ts
export class JDBService {
  private apiUrl = 'https://api.jdb1688.net';
  private dc = process.env.JDB_DC;
  private key = process.env.JDB_KEY;

  async launchGame(userId: string, gameCode: string): Promise<string> {
    const playerData = { userId, gameCode };
    const encryptedData = this.encrypt(JSON.stringify(playerData));
    
    const response = await axios.post(`${this.apiUrl}/game/launch`, {
      dc: this.dc,
      data: encryptedData
    });

    return response.data.gameUrl;
  }

  private encrypt(data: string): string {
    const cipher = crypto.createCipher('aes-256-cbc', this.key);
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
  }
}
```

---

## 🏗️ Database Models

### User Model
```typescript
// user.model.ts
export interface User {
  id: string;
  clientId: string;
  username: string;
  email: string;
  userType: 'super_admin' | 'admin' | 'master_agent' | 'sub_agent' | 'player';
  parentUserId?: string;
  balance: number;
  status: 'active' | 'suspended' | 'inactive';
}
```

---

## 📊 API Endpoints

### User Controller
```typescript
// user.controller.ts
export class UserController {
  static async createUser(req: Request, res: Response) {
    try {
      const { username, email, password, userType, parentUserId } = req.body;
      const clientId = req.user.clientId;

      const hashedPassword = await AuthService.hashPassword(password);
      
      const user = await User.create({
        clientId,
        username,
        email,
        passwordHash: hashedPassword,
        userType,
        parentUserId
      });

      res.status(201).json({ user: { ...user, passwordHash: undefined } });
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}
```

---

## 🎯 React Components

### Balance Display
```tsx
// BalanceDisplay.tsx
import React, { useState, useEffect } from 'react';

export const BalanceDisplay: React.FC<{ userId: string }> = ({ userId }) => {
  const [balance, setBalance] = useState(0);

  useEffect(() => {
    fetchBalance(userId).then(setBalance);
  }, [userId]);

  return (
    <div className="balance-display">
      <span>Balance: ${balance.toFixed(2)}</span>
    </div>
  );
};
```