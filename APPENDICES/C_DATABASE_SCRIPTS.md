# 🗄️ Database Scripts
## Essential SQL Scripts and Migrations

---

## 📋 Core Table Creation

### White Label Clients
```sql
CREATE TABLE white_label_clients (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_code VARCHAR(50) UNIQUE NOT NULL,
    company_name VARCHAR(255) NOT NULL,
    domain VARCHAR(255) UNIQUE NOT NULL,
    contact_email VARCHAR(255) NOT NULL,
    revenue_share_percentage DECIMAL(5,2) DEFAULT 30.00,
    setup_fee DECIMAL(15,2) NOT NULL,
    monthly_fee DECIMAL(15,2) NOT NULL,
    current_balance DECIMAL(15,2) DEFAULT 0.00,
    status ENUM('trial', 'active', 'suspended', 'terminated') DEFAULT 'trial',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_client_code (client_code),
    INDEX idx_domain (domain),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Users Table
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    user_type ENUM('super_admin', 'admin', 'master_agent', 'sub_agent', 'player') NOT NULL,
    parent_user_id BIGINT NULL,
    balance DECIMAL(15,2) DEFAULT 0.00,
    status ENUM('active', 'suspended', 'inactive') DEFAULT 'active',
    is_independent_user BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_user_id) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_client_user_type (client_id, user_type, status),
    INDEX idx_parent_user (parent_user_id),
    INDEX idx_username (username),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Transactions Table
```sql
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    transaction_type ENUM('deposit', 'withdrawal', 'transfer_in', 'transfer_out', 'bet', 'win', 'commission') NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    balance_before DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    reference_id VARCHAR(100),
    from_user_id BIGINT NULL,
    to_user_id BIGINT NULL,
    status ENUM('pending', 'completed', 'failed', 'cancelled') DEFAULT 'pending',
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (from_user_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (to_user_id) REFERENCES users(id) ON DELETE SET NULL,
    
    INDEX idx_client_user_type (client_id, user_id, transaction_type),
    INDEX idx_user_date (user_id, created_at DESC),
    INDEX idx_reference_id (reference_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 🎮 Game System Tables

### Game Providers
```sql
CREATE TABLE game_providers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_code VARCHAR(50) UNIQUE NOT NULL,
    provider_name VARCHAR(255) NOT NULL,
    provider_type ENUM('casino', 'sports', 'live_casino', 'slots') NOT NULL,
    api_endpoint VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    configuration JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_provider_code (provider_code),
    INDEX idx_provider_type (provider_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Games Table
```sql
CREATE TABLE games (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_id INT NOT NULL,
    game_code VARCHAR(100) NOT NULL,
    game_name VARCHAR(255) NOT NULL,
    game_type ENUM('slot', 'table', 'live_casino', 'sports', 'crash') NOT NULL,
    thumbnail_url VARCHAR(500),
    min_bet DECIMAL(10,2) DEFAULT 1.00,
    max_bet DECIMAL(10,2) DEFAULT 10000.00,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (provider_id) REFERENCES game_providers(id) ON DELETE CASCADE,
    UNIQUE KEY unique_provider_game (provider_id, game_code),
    INDEX idx_game_type (game_type),
    INDEX idx_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 📊 Sample Data Insertion

### Insert Sample Client
```sql
INSERT INTO white_label_clients (
    client_code, company_name, domain, contact_email,
    revenue_share_percentage, setup_fee, monthly_fee
) VALUES (
    'ARIF_88TAKA', '88taka.com', '88taka.com', 'arif@88taka.com',
    30.00, 15000.00, 3000.00
);
```

### Insert User Hierarchy
```sql
-- Super Admin
INSERT INTO users (
    client_id, username, email, password_hash, user_type, full_name, balance
) VALUES (
    1, 'arif_super', 'arif@88taka.com', '$2b$10$hashed_password', 
    'super_admin', 'ARIF', 5000.00
);

-- Admin
INSERT INTO users (
    client_id, username, email, password_hash, user_type, parent_user_id, 
    full_name, balance, created_by
) VALUES (
    1, 'admin1', 'admin1@88taka.com', '$2b$10$hashed_password',
    'admin', 1, 'Admin One', 0.00, 1
);
```

### Insert Game Providers
```sql
INSERT INTO game_providers (provider_code, provider_name, provider_type, api_endpoint) VALUES
('JDB', 'JDB Games', 'casino', 'https://api.jdb1688.net'),
('LUCKYSPORTS', 'Lucky Sports', 'sports', 'https://smt-arif-subt-880r657.uni247.online/');
```

---

## 🔍 Common Queries

### User Hierarchy Query
```sql
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

### Daily Revenue Report
```sql
SELECT 
    DATE(created_at) as report_date,
    SUM(CASE WHEN transaction_type = 'bet' THEN amount ELSE 0 END) as total_bets,
    SUM(CASE WHEN transaction_type = 'win' THEN amount ELSE 0 END) as total_wins,
    SUM(CASE WHEN transaction_type = 'bet' THEN amount ELSE 0 END) - 
    SUM(CASE WHEN transaction_type = 'win' THEN amount ELSE 0 END) as gross_revenue
FROM transactions 
WHERE client_id = ? 
    AND transaction_type IN ('bet', 'win')
    AND created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(created_at)
ORDER BY report_date DESC;
```

---

## 🔧 Database Maintenance

### Performance Indexes
```sql
-- Essential performance indexes
CREATE INDEX idx_users_hierarchy ON users(client_id, parent_user_id, user_type, status);
CREATE INDEX idx_transactions_user_type_date ON transactions(user_id, transaction_type, created_at DESC);
CREATE INDEX idx_transactions_client_amount ON transactions(client_id, amount, created_at DESC);
```

### Cleanup Scripts
```sql
-- Clean old sessions (run daily)
DELETE FROM user_sessions 
WHERE expires_at < NOW() - INTERVAL 7 DAY;

-- Archive old transactions (run monthly)
INSERT INTO transactions_archive 
SELECT * FROM transactions 
WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
```

---

## 📈 Monitoring Queries

### Database Health Check
```sql
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)',
    table_rows
FROM information_schema.tables 
WHERE table_schema = 'nasgames'
ORDER BY (data_length + index_length) DESC;
```

### Active Users Count
```sql
SELECT 
    c.company_name,
    COUNT(u.id) as total_users,
    COUNT(CASE WHEN u.status = 'active' THEN 1 END) as active_users
FROM white_label_clients c
LEFT JOIN users u ON c.id = u.client_id
GROUP BY c.id, c.company_name;
```