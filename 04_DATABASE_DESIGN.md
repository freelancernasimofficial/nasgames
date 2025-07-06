# 🗄️ Database Design & Schema

## NasGames.com White Label Casino Platform

---

## 📊 Database Overview

The database design follows a **multi-tenant architecture** with shared tables using client isolation. This approach balances cost-effectiveness with security and performance, supporting unlimited white label clients while maintaining data integrity.

---

## 🏗️ Database Architecture Strategy

### Multi-Tenant Approach: Shared Database with Row-Level Security

```
Advantages:
✅ Cost-effective (single database instance)
✅ Easier maintenance and updates
✅ Shared resources and optimizations
✅ Simplified backup and recovery
✅ Consistent schema across all clients

Implementation:
├── All tables include client_id column
├── Row-level security policies
├── Application-level access control
├── Indexed queries for performance
└── Audit trails for compliance
```

### Database Technology Stack

```
Primary Database: MySQL 8.0+
├── ACID compliance for financial transactions
├── JSON column support for flexible data
├── Advanced indexing capabilities
├── Replication support for scaling
└── Proven reliability for financial systems

Caching Layer: Redis 7+
├── Session storage
├── Frequently accessed data
├── Real-time leaderboards
├── Rate limiting counters
└── Pub/Sub for real-time updates

Analytics Database: MongoDB
├── User behavior tracking
├── Game analytics
├── Flexible schema for events
└── Aggregation pipelines

Time-Series Database: InfluxDB
├── Performance metrics
├── System monitoring
├── Real-time dashboards
└── Historical trend analysis
```

---

## 📋 Complete Database Schema

### 1. Core Platform Tables

#### White Label Clients

```sql
CREATE TABLE white_label_clients (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_code VARCHAR(50) UNIQUE NOT NULL COMMENT 'Unique identifier (e.g., ARIF_88TAKA)',
    company_name VARCHAR(255) NOT NULL COMMENT 'Business name',
    domain VARCHAR(255) UNIQUE NOT NULL COMMENT 'Client website domain',
    subdomain VARCHAR(100) UNIQUE NULL COMMENT 'Optional subdomain on our platform',

    -- Contact Information
    contact_person VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50),
    business_address TEXT,

    -- Business Configuration
    revenue_share_percentage DECIMAL(5,2) DEFAULT 30.00 COMMENT 'Platform commission %',
    setup_fee DECIMAL(15,2) NOT NULL COMMENT 'One-time setup cost',
    monthly_fee DECIMAL(15,2) NOT NULL COMMENT 'Recurring monthly fee',
    currency VARCHAR(3) DEFAULT 'USD' COMMENT 'Primary currency',
    timezone VARCHAR(50) DEFAULT 'UTC',

    -- Financial Tracking
    current_balance DECIMAL(15,2) DEFAULT 0.00 COMMENT 'Client current balance',
    total_revenue DECIMAL(15,2) DEFAULT 0.00 COMMENT 'Lifetime revenue generated',
    total_commission_paid DECIMAL(15,2) DEFAULT 0.00 COMMENT 'Total paid to platform',

    -- Status and Lifecycle
    status ENUM('trial', 'active', 'suspended', 'terminated') DEFAULT 'trial',
    trial_ends_at TIMESTAMP NULL,
    contract_start_date DATE,
    contract_end_date DATE,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by INT COMMENT 'Admin who created this client',

    -- Indexes
    INDEX idx_client_code (client_code),
    INDEX idx_domain (domain),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Client Configuration

```sql
CREATE TABLE client_configurations (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    -- Branding Configuration (JSON)
    branding_config JSON COMMENT 'Logo, colors, theme settings',

    -- Feature Toggles (JSON)
    enabled_features JSON COMMENT 'Available features for this client',

    -- Game Provider Access (JSON)
    game_providers JSON COMMENT 'Enabled game providers and settings',

    -- Payment Configuration (JSON)
    payment_methods JSON COMMENT 'Available payment methods',

    -- Localization Settings
    supported_languages JSON COMMENT 'Available languages',
    default_language VARCHAR(5) DEFAULT 'en',

    -- Business Rules (JSON)
    business_rules JSON COMMENT 'Custom business logic and limits',

    -- Compliance Settings
    kyc_required BOOLEAN DEFAULT TRUE,
    max_deposit_daily DECIMAL(15,2),
    max_withdrawal_daily DECIMAL(15,2),
    max_bet_amount DECIMAL(15,2),

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    INDEX idx_client_id (client_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 2. User Management System

#### Users Table (All User Types)

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL COMMENT 'Which white label client this user belongs to',

    -- Basic Information
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    full_name VARCHAR(255) NOT NULL,
    date_of_birth DATE,

    -- User Type and Hierarchy
    user_type ENUM('super_admin', 'admin', 'master_agent', 'sub_agent', 'player') NOT NULL,
    parent_user_id BIGINT NULL COMMENT 'Hierarchical parent',
    is_independent_user BOOLEAN DEFAULT TRUE COMMENT 'FALSE for sub_agent created users',

    -- Account Status
    status ENUM('active', 'suspended', 'inactive', 'pending_verification') DEFAULT 'pending_verification',
    email_verified BOOLEAN DEFAULT FALSE,
    phone_verified BOOLEAN DEFAULT FALSE,
    kyc_status ENUM('not_required', 'pending', 'approved', 'rejected') DEFAULT 'not_required',

    -- Financial Information
    balance DECIMAL(15,2) DEFAULT 0.00,
    bonus_balance DECIMAL(15,2) DEFAULT 0.00,
    total_deposited DECIMAL(15,2) DEFAULT 0.00,
    total_withdrawn DECIMAL(15,2) DEFAULT 0.00,
    total_wagered DECIMAL(15,2) DEFAULT 0.00,

    -- Limits and Permissions
    max_child_users INT DEFAULT 100 COMMENT 'Limit for creating downline users',
    daily_deposit_limit DECIMAL(15,2),
    daily_withdrawal_limit DECIMAL(15,2),
    daily_bet_limit DECIMAL(15,2),

    -- Security
    two_factor_enabled BOOLEAN DEFAULT FALSE,
    two_factor_secret VARCHAR(255),
    failed_login_attempts INT DEFAULT 0,
    locked_until TIMESTAMP NULL,

    -- Activity Tracking
    last_login TIMESTAMP NULL,
    last_activity TIMESTAMP NULL,
    login_count INT DEFAULT 0,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by BIGINT NULL COMMENT 'Who created this user',

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_user_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL,

    -- Indexes
    INDEX idx_client_user_type (client_id, user_type, status),
    INDEX idx_parent_user (parent_user_id),
    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_phone (phone),
    INDEX idx_status (status),
    INDEX idx_last_activity (last_activity)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### User Permissions

```sql
CREATE TABLE user_permissions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    permission_name VARCHAR(100) NOT NULL,
    permission_value BOOLEAN DEFAULT TRUE,
    granted_by BIGINT NULL,
    expires_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (granted_by) REFERENCES users(id) ON DELETE SET NULL,
    UNIQUE KEY unique_user_permission (user_id, permission_name),
    INDEX idx_user_permission (user_id, permission_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### User Sessions

```sql
CREATE TABLE user_sessions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    session_token VARCHAR(255) UNIQUE NOT NULL,
    refresh_token VARCHAR(255) UNIQUE NOT NULL,

    -- Session Information
    ip_address VARCHAR(45),
    user_agent TEXT,
    device_info JSON,
    location_info JSON,

    -- Session Status
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP NOT NULL,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_session_token (session_token),
    INDEX idx_user_expires (user_id, expires_at),
    INDEX idx_active_sessions (user_id, is_active, expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3. Financial System

#### Transactions (All Financial Activities)

```sql
CREATE TABLE transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,

    -- Transaction Details
    transaction_type ENUM(
        'deposit', 'withdrawal', 'transfer_in', 'transfer_out',
        'bet', 'win', 'commission', 'bonus', 'refund', 'adjustment'
    ) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',

    -- Balance Tracking
    balance_before DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    bonus_balance_before DECIMAL(15,2) DEFAULT 0.00,
    bonus_balance_after DECIMAL(15,2) DEFAULT 0.00,

    -- Reference Information
    reference_id VARCHAR(100) COMMENT 'External reference (payment gateway, game session, etc.)',
    reference_type ENUM('payment', 'game_session', 'transfer', 'manual', 'system') DEFAULT 'manual',

    -- Transfer Information (for balance transfers)
    from_user_id BIGINT NULL,
    to_user_id BIGINT NULL,

    -- Transaction Status
    status ENUM('pending', 'processing', 'completed', 'failed', 'cancelled', 'reversed') DEFAULT 'pending',
    failure_reason TEXT NULL,

    -- Processing Information
    processed_by BIGINT NULL,
    processed_at TIMESTAMP NULL,

    -- Additional Data
    description TEXT,
    metadata JSON COMMENT 'Additional transaction data',

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (from_user_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (to_user_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (processed_by) REFERENCES users(id) ON DELETE SET NULL,

    -- Indexes
    INDEX idx_client_user_type (client_id, user_id, transaction_type),
    INDEX idx_user_date (user_id, created_at DESC),
    INDEX idx_reference (reference_id, reference_type),
    INDEX idx_status (status),
    INDEX idx_type_date (transaction_type, created_at DESC),
    INDEX idx_amount_date (amount, created_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Commission Tracking

```sql
CREATE TABLE commissions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    -- Commission Recipients
    user_id BIGINT NOT NULL COMMENT 'Who earned the commission',
    source_user_id BIGINT NOT NULL COMMENT 'Player who generated the revenue',

    -- Commission Details
    commission_type ENUM('revenue_share', 'referral', 'bonus', 'override') NOT NULL,
    commission_level INT DEFAULT 1 COMMENT 'Level in hierarchy (1=direct, 2=override, etc.)',

    -- Financial Calculations
    gross_amount DECIMAL(15,2) NOT NULL COMMENT 'Original bet/loss amount',
    commission_rate DECIMAL(5,2) NOT NULL COMMENT 'Percentage rate',
    commission_amount DECIMAL(15,2) NOT NULL COMMENT 'Calculated commission',

    -- Time Period
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    calculation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Payment Status
    status ENUM('pending', 'approved', 'paid', 'cancelled') DEFAULT 'pending',
    paid_at TIMESTAMP NULL,
    payment_reference VARCHAR(100),

    -- Additional Information
    description TEXT,
    metadata JSON,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (source_user_id) REFERENCES users(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_client_user_period (client_id, user_id, period_start, period_end),
    INDEX idx_source_user_date (source_user_id, created_at),
    INDEX idx_status (status),
    INDEX idx_commission_type (commission_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 4. Game Integration System

#### Game Providers

```sql
CREATE TABLE game_providers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_code VARCHAR(50) UNIQUE NOT NULL COMMENT 'JDB, LUCKYSPORTS, etc.',
    provider_name VARCHAR(255) NOT NULL,
    provider_type ENUM('casino', 'sports', 'live_casino', 'slots', 'table_games') NOT NULL,

    -- API Configuration
    api_endpoint VARCHAR(500),
    api_version VARCHAR(20),
    authentication_type ENUM('api_key', 'oauth', 'basic_auth', 'custom') DEFAULT 'api_key',

    -- Provider Settings
    supports_transfer_wallet BOOLEAN DEFAULT TRUE,
    supports_seamless_wallet BOOLEAN DEFAULT FALSE,
    min_bet DECIMAL(10,2) DEFAULT 1.00,
    max_bet DECIMAL(10,2) DEFAULT 10000.00,

    -- Status and Configuration
    is_active BOOLEAN DEFAULT TRUE,
    configuration JSON COMMENT 'Provider-specific settings',
    credentials JSON COMMENT 'Encrypted API credentials',

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Indexes
    INDEX idx_provider_code (provider_code),
    INDEX idx_provider_type (provider_type),
    INDEX idx_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Master Account Credentials

```sql
-- Store master account credentials for serving all clients
CREATE TABLE master_api_credentials (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_id INT NOT NULL,

    -- Credential details
    credential_type ENUM('master_account', 'individual_account') DEFAULT 'master_account',
    api_endpoint VARCHAR(500) NOT NULL,

    -- Encrypted credentials (JSON format)
    credentials JSON NOT NULL COMMENT 'Encrypted master account credentials',

    -- Usage tracking
    total_clients_served INT DEFAULT 0,
    monthly_cost DECIMAL(10,2) DEFAULT 0.00,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (provider_id) REFERENCES game_providers(id) ON DELETE CASCADE,
    INDEX idx_provider_active (provider_id, is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Game Provider Players (Master Account Mapping)

```sql
-- Maps internal users to game provider player IDs using master account
CREATE TABLE game_provider_players (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    provider_id INT NOT NULL,

    -- Provider-specific player ID (with client prefix)
    provider_player_id VARCHAR(100) NOT NULL COMMENT 'e.g., ARIF_88TAKA_user123',
    provider_username VARCHAR(100) COMMENT 'Provider internal username',

    -- Account status
    status ENUM('active', 'suspended', 'closed') DEFAULT 'active',
    balance DECIMAL(15,2) DEFAULT 0.00 COMMENT 'Provider-side balance',

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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Games Catalog

```sql
CREATE TABLE games (
    id INT PRIMARY KEY AUTO_INCREMENT,
    provider_id INT NOT NULL,
    game_code VARCHAR(100) NOT NULL,
    game_name VARCHAR(255) NOT NULL,

    -- Game Classification
    game_type ENUM('slot', 'table', 'live_casino', 'sports', 'crash', 'fishing', 'lottery') NOT NULL,
    category VARCHAR(100) COMMENT 'Popular, New, Jackpot, etc.',
    subcategory VARCHAR(100),

    -- Game Information
    description TEXT,
    thumbnail_url VARCHAR(500),
    banner_url VARCHAR(500),
    demo_available BOOLEAN DEFAULT TRUE,

    -- Game Settings
    min_bet DECIMAL(10,2) DEFAULT 1.00,
    max_bet DECIMAL(10,2) DEFAULT 10000.00,
    rtp_percentage DECIMAL(5,2) COMMENT 'Return to Player percentage',
    volatility ENUM('low', 'medium', 'high'),

    -- Features
    has_jackpot BOOLEAN DEFAULT FALSE,
    has_bonus_rounds BOOLEAN DEFAULT FALSE,
    has_free_spins BOOLEAN DEFAULT FALSE,

    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (provider_id) REFERENCES game_providers(id) ON DELETE CASCADE,

    -- Indexes
    UNIQUE KEY unique_provider_game (provider_id, game_code),
    INDEX idx_game_type (game_type),
    INDEX idx_category (category),
    INDEX idx_active_featured (is_active, is_featured),
    INDEX idx_sort_order (sort_order)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Client Game Settings

```sql
CREATE TABLE client_games (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    game_id INT NOT NULL,

    -- Game Availability
    is_enabled BOOLEAN DEFAULT TRUE,
    is_featured BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,

    -- Custom Settings
    custom_name VARCHAR(255) NULL,
    custom_thumbnail VARCHAR(500) NULL,
    custom_min_bet DECIMAL(10,2) NULL,
    custom_max_bet DECIMAL(10,2) NULL,

    -- Commission Settings
    commission_rate DECIMAL(5,2) DEFAULT 0.00 COMMENT 'Client commission from this game',

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (game_id) REFERENCES games(id) ON DELETE CASCADE,

    -- Indexes
    UNIQUE KEY unique_client_game (client_id, game_id),
    INDEX idx_client_enabled (client_id, is_enabled),
    INDEX idx_client_featured (client_id, is_featured)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Game Sessions

```sql
CREATE TABLE game_sessions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    game_id INT NOT NULL,

    -- Session Information
    session_id VARCHAR(255) UNIQUE NOT NULL COMMENT 'External session ID from provider',
    provider_session_id VARCHAR(255) COMMENT 'Provider internal session ID',

    -- Financial Information
    bet_amount DECIMAL(15,2) NOT NULL,
    win_amount DECIMAL(15,2) DEFAULT 0.00,
    net_amount DECIMAL(15,2) NOT NULL COMMENT 'win_amount - bet_amount',
    bonus_amount DECIMAL(15,2) DEFAULT 0.00,

    -- Session Status
    status ENUM('active', 'completed', 'cancelled', 'error') DEFAULT 'active',

    -- Timing
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP NULL,
    duration_seconds INT NULL,

    -- Additional Data
    game_data JSON COMMENT 'Game-specific data and results',
    provider_data JSON COMMENT 'Raw provider response data',

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (game_id) REFERENCES games(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_client_user_game (client_id, user_id, game_id),
    INDEX idx_session_id (session_id),
    INDEX idx_user_date (user_id, started_at DESC),
    INDEX idx_game_date (game_id, started_at DESC),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 5. Payment System

#### Payment Methods

```sql
CREATE TABLE payment_methods (
    id INT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,

    -- Method Information
    method_name VARCHAR(100) NOT NULL COMMENT 'bKash, Nagad, Bank Transfer, etc.',
    method_code VARCHAR(50) NOT NULL,
    method_type ENUM('mobile_banking', 'bank_transfer', 'crypto', 'card', 'e_wallet') NOT NULL,

    -- Availability
    is_active BOOLEAN DEFAULT TRUE,
    available_for_deposit BOOLEAN DEFAULT TRUE,
    available_for_withdrawal BOOLEAN DEFAULT TRUE,

    -- Limits
    min_deposit_amount DECIMAL(10,2) DEFAULT 0.00,
    max_deposit_amount DECIMAL(10,2) DEFAULT 999999.99,
    min_withdrawal_amount DECIMAL(10,2) DEFAULT 0.00,
    max_withdrawal_amount DECIMAL(10,2) DEFAULT 999999.99,

    -- Fees
    deposit_fee_percentage DECIMAL(5,2) DEFAULT 0.00,
    deposit_fee_fixed DECIMAL(10,2) DEFAULT 0.00,
    withdrawal_fee_percentage DECIMAL(5,2) DEFAULT 0.00,
    withdrawal_fee_fixed DECIMAL(10,2) DEFAULT 0.00,

    -- Processing Times
    deposit_processing_time VARCHAR(100) DEFAULT 'Instant',
    withdrawal_processing_time VARCHAR(100) DEFAULT '24 hours',

    -- Configuration
    gateway_config JSON COMMENT 'Gateway-specific configuration',
    display_config JSON COMMENT 'UI display settings',

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,

    -- Indexes
    INDEX idx_client_active (client_id, is_active),
    INDEX idx_method_type (method_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### Payment Transactions

```sql
CREATE TABLE payment_transactions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    user_id BIGINT NOT NULL,
    payment_method_id INT NOT NULL,

    -- Transaction Information
    transaction_type ENUM('deposit', 'withdrawal') NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',

    -- Fees
    fee_amount DECIMAL(15,2) DEFAULT 0.00,
    net_amount DECIMAL(15,2) NOT NULL COMMENT 'Amount after fees',

    -- Status Tracking
    status ENUM('pending', 'processing', 'completed', 'failed', 'cancelled', 'expired') DEFAULT 'pending',

    -- Gateway Information
    gateway_transaction_id VARCHAR(255),
    gateway_reference VARCHAR(255),
    gateway_response JSON,

    -- User Information (for manual verification)
    user_account_info JSON COMMENT 'User provided account details',

    -- Processing Information
    processed_by BIGINT NULL,
    processed_at TIMESTAMP NULL,
    failure_reason TEXT NULL,

    -- Verification
    requires_manual_verification BOOLEAN DEFAULT FALSE,
    verification_status ENUM('not_required', 'pending', 'approved', 'rejected') DEFAULT 'not_required',
    verified_by BIGINT NULL,
    verified_at TIMESTAMP NULL,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (payment_method_id) REFERENCES payment_methods(id) ON DELETE CASCADE,
    FOREIGN KEY (processed_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (verified_by) REFERENCES users(id) ON DELETE SET NULL,

    -- Indexes
    INDEX idx_client_user_type (client_id, user_id, transaction_type),
    INDEX idx_status (status),
    INDEX idx_gateway_transaction (gateway_transaction_id),
    INDEX idx_created_at (created_at DESC),
    INDEX idx_verification (verification_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 6. Reporting and Analytics

#### Daily Reports (Pre-calculated)

```sql
CREATE TABLE daily_reports (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NOT NULL,
    report_date DATE NOT NULL,

    -- User Metrics
    total_users INT DEFAULT 0,
    new_registrations INT DEFAULT 0,
    active_users INT DEFAULT 0,
    verified_users INT DEFAULT 0,

    -- Financial Metrics
    total_deposits DECIMAL(15,2) DEFAULT 0.00,
    total_withdrawals DECIMAL(15,2) DEFAULT 0.00,
    total_bets DECIMAL(15,2) DEFAULT 0.00,
    total_wins DECIMAL(15,2) DEFAULT 0.00,
    gross_gaming_revenue DECIMAL(15,2) DEFAULT 0.00,
    net_gaming_revenue DECIMAL(15,2) DEFAULT 0.00,

    -- Game Metrics
    total_game_sessions INT DEFAULT 0,
    unique_players INT DEFAULT 0,
    average_session_duration DECIMAL(10,2) DEFAULT 0.00,

    -- Commission Metrics
    total_commissions DECIMAL(15,2) DEFAULT 0.00,
    platform_commission DECIMAL(15,2) DEFAULT 0.00,
    client_share DECIMAL(15,2) DEFAULT 0.00,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE CASCADE,

    -- Indexes
    UNIQUE KEY unique_client_date (client_id, report_date),
    INDEX idx_report_date (report_date),
    INDEX idx_client_date (client_id, report_date DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### System Logs

```sql
CREATE TABLE system_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id INT NULL,
    user_id BIGINT NULL,

    -- Log Information
    log_level ENUM('debug', 'info', 'warning', 'error', 'critical') NOT NULL,
    action VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL COMMENT 'auth, payment, game, system, etc.',

    -- Request Information
    ip_address VARCHAR(45),
    user_agent TEXT,
    request_id VARCHAR(100),

    -- Log Data
    message TEXT NOT NULL,
    details JSON,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Foreign Keys
    FOREIGN KEY (client_id) REFERENCES white_label_clients(id) ON DELETE SET NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,

    -- Indexes
    INDEX idx_client_action (client_id, action),
    INDEX idx_user_action (user_id, action),
    INDEX idx_level_date (log_level, created_at),
    INDEX idx_category_date (category, created_at),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 🔧 Database Optimization

### Essential Indexes

```sql
-- Performance-critical indexes
-- User hierarchy queries
CREATE INDEX idx_users_hierarchy ON users(client_id, parent_user_id, user_type, status);

-- Financial transaction queries
CREATE INDEX idx_transactions_user_type_date ON transactions(user_id, transaction_type, created_at DESC);
CREATE INDEX idx_transactions_client_amount ON transactions(client_id, amount, created_at DESC);

-- Game session queries
CREATE INDEX idx_game_sessions_user_game_date ON game_sessions(user_id, game_id, started_at DESC);
CREATE INDEX idx_game_sessions_client_status ON game_sessions(client_id, status, started_at DESC);

-- Commission calculation queries
CREATE INDEX idx_commissions_calculation ON commissions(client_id, source_user_id, period_start, period_end);

-- Payment processing queries
CREATE INDEX idx_payment_transactions_status_date ON payment_transactions(status, created_at DESC);
CREATE INDEX idx_payment_transactions_user_type ON payment_transactions(user_id, transaction_type, status);

-- Reporting queries
CREATE INDEX idx_daily_reports_client_date_range ON daily_reports(client_id, report_date DESC);

-- System monitoring queries
CREATE INDEX idx_system_logs_level_category_date ON system_logs(log_level, category, created_at DESC);

-- Master account and game provider player queries
CREATE INDEX idx_game_provider_players_client_provider ON game_provider_players(client_id, provider_id, status);
CREATE INDEX idx_game_provider_players_provider_player ON game_provider_players(provider_player_id, status);
CREATE INDEX idx_master_credentials_provider ON master_api_credentials(provider_id, is_active);
```

### Database Configuration

```sql
-- MySQL Configuration for Production
SET GLOBAL innodb_buffer_pool_size = 2147483648; -- 2GB
SET GLOBAL innodb_log_file_size = 268435456; -- 256MB
SET GLOBAL max_connections = 500;
SET GLOBAL query_cache_size = 134217728; -- 128MB
SET GLOBAL innodb_flush_log_at_trx_commit = 1; -- ACID compliance
SET GLOBAL sync_binlog = 1; -- Replication safety

-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2; -- Log queries taking more than 2 seconds
```

---

## 📊 Sample Data and Queries

### Sample Data Insertion

```sql
-- Insert sample white label client
INSERT INTO white_label_clients (
    client_code, company_name, domain, contact_person, contact_email,
    revenue_share_percentage, setup_fee, monthly_fee, currency
) VALUES (
    'ARIF_88TAKA', '88taka.com', '88taka.com', 'ARIF', 'arif@88taka.com',
    30.00, 15000.00, 3000.00, 'BDT'
);

-- Insert game providers
INSERT INTO game_providers (provider_code, provider_name, provider_type, api_endpoint) VALUES
('JDB', 'JDB Games', 'casino', 'https://api.jdb1688.net'),
('LUCKYSPORTS', 'Lucky Sports', 'sports', 'https://smt-arif-subt-880r657.uni247.online/');

-- Insert master account credentials
INSERT INTO master_api_credentials (provider_id, credential_type, api_endpoint, credentials, monthly_cost) VALUES
(1, 'master_account', 'https://api.jdb1688.net',
 JSON_OBJECT('dc', 'ARIF', 'key', 'encrypted_key', 'iv', 'encrypted_iv'), 3000.00),
(2, 'master_account', 'https://smt-arif-subt-880r657.uni247.online/',
 JSON_OBJECT('id', 'smt-arif-subt-880r657', 'apiKey', 'encrypted_key'), 2000.00);

-- Insert user hierarchy
-- Super Admin (ARIF)
INSERT INTO users (
    client_id, username, email, password_hash, user_type, full_name, balance
) VALUES (
    1, 'arif_super', 'arif@88taka.com', '$2b$10$hashed_password',
    'super_admin', 'ARIF', 5000.00
);

-- Admin under Super Admin
INSERT INTO users (
    client_id, username, email, password_hash, user_type, parent_user_id,
    full_name, balance, created_by
) VALUES (
    1, 'admin1', 'admin1@88taka.com', '$2b$10$hashed_password',
    'admin', 1, 'Admin One', 0.00, 1
);
```

### Common Queries

```sql
-- Get user hierarchy for a client
WITH RECURSIVE user_hierarchy AS (
    -- Base case: Super admin
    SELECT id, username, full_name, user_type, parent_user_id, 0 as level
    FROM users
    WHERE client_id = 1 AND parent_user_id IS NULL

    UNION ALL

    -- Recursive case: Children
    SELECT u.id, u.username, u.full_name, u.user_type, u.parent_user_id, uh.level + 1
    FROM users u
    INNER JOIN user_hierarchy uh ON u.parent_user_id = uh.id
    WHERE u.client_id = 1
)
SELECT * FROM user_hierarchy ORDER BY level, username;

-- Calculate daily revenue for a client
SELECT
    DATE(created_at) as report_date,
    SUM(CASE WHEN transaction_type = 'bet' THEN amount ELSE 0 END) as total_bets,
    SUM(CASE WHEN transaction_type = 'win' THEN amount ELSE 0 END) as total_wins,
    SUM(CASE WHEN transaction_type = 'bet' THEN amount ELSE 0 END) -
    SUM(CASE WHEN transaction_type = 'win' THEN amount ELSE 0 END) as gross_gaming_revenue
FROM transactions
WHERE client_id = 1
    AND transaction_type IN ('bet', 'win')
    AND created_at >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(created_at)
ORDER BY report_date DESC;

-- Get top performing games for a client
SELECT
    g.game_name,
    COUNT(gs.id) as total_sessions,
    SUM(gs.bet_amount) as total_bets,
    SUM(gs.win_amount) as total_wins,
    SUM(gs.net_amount) as net_revenue,
    AVG(gs.duration_seconds) as avg_duration
FROM game_sessions gs
JOIN games g ON gs.game_id = g.id
WHERE gs.client_id = 1
    AND gs.status = 'completed'
    AND gs.started_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY g.id, g.game_name
ORDER BY net_revenue DESC
LIMIT 10;

-- Get all game provider players for a client (master account)
SELECT
    gpp.provider_player_id,
    u.username,
    gp.provider_name,
    gpp.balance as provider_balance,
    gpp.status
FROM game_provider_players gpp
JOIN users u ON gpp.user_id = u.id
JOIN game_providers gp ON gpp.provider_id = gp.id
WHERE gpp.client_id = 1;

-- Track master account usage across clients
SELECT
    gp.provider_name,
    mac.monthly_cost,
    COUNT(DISTINCT gpp.client_id) as clients_served,
    COUNT(gpp.id) as total_players,
    SUM(gpp.balance) as total_provider_balance
FROM master_api_credentials mac
JOIN game_providers gp ON mac.provider_id = gp.id
LEFT JOIN game_provider_players gpp ON gp.id = gpp.provider_id
WHERE mac.is_active = TRUE
GROUP BY gp.id, gp.provider_name, mac.monthly_cost;
```

---

## 🔒 Security Considerations

### Row-Level Security

```sql
-- Create security policies for multi-tenant access
-- Example: Users can only see their own client's data
CREATE VIEW secure_users AS
SELECT * FROM users
WHERE client_id = @current_client_id;

-- Example: Transactions are filtered by client
CREATE VIEW secure_transactions AS
SELECT * FROM transactions
WHERE client_id = @current_client_id;
```

### Data Encryption

```sql
-- Encrypt sensitive columns
ALTER TABLE users
ADD COLUMN encrypted_phone VARBINARY(255),
ADD COLUMN encrypted_email VARBINARY(255);

-- Use MySQL's encryption functions
UPDATE users SET
    encrypted_phone = AES_ENCRYPT(phone, 'encryption_key'),
    encrypted_email = AES_ENCRYPT(email, 'encryption_key');
```

---

## 📈 Monitoring and Maintenance

### Database Health Monitoring

```sql
-- Monitor table sizes
SELECT
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)',
    table_rows
FROM information_schema.tables
WHERE table_schema = 'nasgames'
ORDER BY (data_length + index_length) DESC;

-- Monitor slow queries
SELECT
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    sql_text
FROM mysql.slow_log
ORDER BY query_time DESC
LIMIT 10;

-- Monitor index usage
SELECT
    table_name,
    index_name,
    cardinality,
    sub_part,
    packed,
    nullable,
    index_type
FROM information_schema.statistics
WHERE table_schema = 'nasgames'
ORDER BY table_name, seq_in_index;
```

---

**This database design provides a robust, scalable foundation for the white label casino platform with proper multi-tenancy, security, performance optimization, and cost-effective master account architecture for game providers.**
