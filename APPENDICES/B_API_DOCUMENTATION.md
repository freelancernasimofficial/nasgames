# 📡 API Documentation
## Core API Endpoints

---

## 🔐 Authentication APIs

### POST /auth/login
```json
Request:
{
  "email": "user@example.com",
  "password": "password123"
}

Response:
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "username": "username",
    "role": "admin",
    "clientId": "client_id"
  }
}
```

### POST /auth/register
```json
Request:
{
  "username": "newuser",
  "email": "user@example.com",
  "password": "password123",
  "userType": "player",
  "parentUserId": "parent_id"
}

Response:
{
  "user": {
    "id": "new_user_id",
    "username": "newuser",
    "email": "user@example.com",
    "userType": "player"
  }
}
```

---

## 👥 User Management APIs

### GET /users
```json
Response:
{
  "users": [
    {
      "id": "user_id",
      "username": "username",
      "email": "email@example.com",
      "userType": "admin",
      "balance": 1000.00,
      "status": "active"
    }
  ]
}
```

### POST /users
```json
Request:
{
  "username": "newuser",
  "email": "user@example.com",
  "password": "password123",
  "userType": "sub_agent",
  "parentUserId": "parent_id"
}
```

### PUT /users/:id/balance
```json
Request:
{
  "amount": 500.00,
  "type": "add"
}

Response:
{
  "newBalance": 1500.00,
  "transaction": {
    "id": "txn_id",
    "amount": 500.00,
    "type": "transfer_in"
  }
}
```

---

## 🎮 Game APIs

### GET /games
```json
Response:
{
  "games": [
    {
      "id": "game_id",
      "name": "Game Name",
      "type": "slot",
      "provider": "JDB",
      "thumbnail": "image_url",
      "minBet": 1.00,
      "maxBet": 1000.00
    }
  ]
}
```

### POST /games/launch
```json
Request:
{
  "gameId": "game_id",
  "userId": "user_id"
}

Response:
{
  "gameUrl": "https://game-provider.com/game?token=xyz",
  "sessionId": "session_id"
}
```

---

## 💰 Financial APIs

### GET /transactions
```json
Response:
{
  "transactions": [
    {
      "id": "txn_id",
      "userId": "user_id",
      "type": "deposit",
      "amount": 100.00,
      "status": "completed",
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### POST /payments/deposit
```json
Request:
{
  "amount": 100.00,
  "paymentMethod": "bkash",
  "accountInfo": {
    "phoneNumber": "01700000000"
  }
}

Response:
{
  "paymentId": "payment_id",
  "status": "pending",
  "instructions": "Send money to 01800000000"
}
```

---

## 📊 Reporting APIs

### GET /reports/daily
```json
Query Parameters:
- startDate: 2024-01-01
- endDate: 2024-01-31
- clientId: client_id (optional)

Response:
{
  "reports": [
    {
      "date": "2024-01-01",
      "totalBets": 10000.00,
      "totalWins": 8500.00,
      "grossRevenue": 1500.00,
      "activeUsers": 150
    }
  ]
}
```

---

## 🏢 Client Management APIs

### GET /clients
```json
Response:
{
  "clients": [
    {
      "id": "client_id",
      "name": "88taka.com",
      "domain": "88taka.com",
      "status": "active",
      "revenueShare": 30.00,
      "currentBalance": 5000.00
    }
  ]
}
```

### POST /clients
```json
Request:
{
  "name": "New Casino",
  "domain": "newcasino.com",
  "contactEmail": "admin@newcasino.com",
  "revenueShare": 25.00,
  "setupFee": 15000.00,
  "monthlyFee": 3000.00
}
```

---

## 🔒 Error Responses

### Standard Error Format
```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": {
    "field": "validation error details"
  }
}
```

### Common HTTP Status Codes
- 200: Success
- 201: Created
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 500: Internal Server Error