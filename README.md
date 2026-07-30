# Chargebyte CSMP Platform

## Overview

The **Chargebyte CSMP (Charging Station Management Platform)** is a comprehensive cloud-based IoT platform designed to manage Chargebyte powerbank rental stations across multiple locations in Africa. The platform provides end-to-end management of powerbank rentals, from station monitoring to payment processing and customer management.

### Key Capabilities

- **Device & Station Management** - Register, monitor, and manage charging stations remotely
- **Battery Inventory Management** - Track battery health, availability, and lifecycle
- **Customer Rental Management** - Handle powerbank rentals, returns, and refunds
- **Payment Processing** - Integrated M-Pesa STK push payments and automated refunds
- **IoT Telemetry Collection** - Real-time data from stations and batteries
- **Remote Diagnostics** - Monitor station health and perform remote troubleshooting
- **Alerts & Notifications** - Automated alerts for system events and anomalies
- **Reporting & Analytics** - Generate insights on usage, revenue, and performance

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     IoT Devices (Stations)                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   Station 1   │    │   Station 2   │    │   Station N   │     │
│  │  DTN03048     │    │  DTN03049     │    │  DTN03050     │     │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘     │
└─────────┼───────────────────┼───────────────────┼─────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
    ┌─────────────────────────────────────────────────────────┐
    │                    Manufacturer API                     │
    │     https://developer.chargenow.top/cdb-open-api        │
    └─────────────────────────────────────────────────────────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                    ┌─────────────────┐
                    │   Webhooks      │
                    │   Callbacks     │
                    └────────┬────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Backend Services                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  REST API Layer                         │  │
│  │  /api/payments/initiate                                 │  │
│  │  /api/payments/mpesa-callback                           │  │
│  │  /api/payments/manufacturer-callback                    │  │
│  │  /api/machines/verify-qr                                │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │               M-Pesa Integration                        │  │
│  │  STK Push │ B2C Refunds │ Callback Processing           │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │               Payment Processing                        │  │
│  │  Initiate │ Polling │ Callback │ Refund Management      │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Rentals  │  │Transactns│  │ Callbacks│  │  Users   │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ Devices  │  │ Stations │  │  Meters  │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Admin Dashboard                              │
│  Real-time Monitoring │ Analytics │ Device Management          │
└─────────────────────────────────────────────────────────────────┘
```

## Core Data Models

### Rentals Table
```sql
CREATE TABLE rentals (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    machine_model VARCHAR(50) NOT NULL,
    manufacturer_trade_no VARCHAR(100),
    rental_code VARCHAR(100),
    deposit_amount DECIMAL(10,2),
    phone_number VARCHAR(20),
    status ENUM('pending_payment', 'active', 'completed', 'cancelled'),
    total_amount DECIMAL(10,2),
    deposit_refunded DECIMAL(10,2),
    deposit_refund_time DATETIME,
    start_time DATETIME,
    end_time DATETIME,
    duration_minutes INT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    metadata JSON
);
```

### Transactions Table
```sql
CREATE TABLE transactions (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50),
    rental_id VARCHAR(50),
    transaction_type ENUM('deposit', 'refund', 'payment'),
    amount DECIMAL(10,2),
    currency VARCHAR(10),
    phone_number VARCHAR(20),
    checkout_request_id VARCHAR(100),
    mpesa_receipt VARCHAR(50),
    status ENUM('pending', 'completed', 'failed'),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    metadata JSON,
    FOREIGN KEY (rental_id) REFERENCES rentals(id)
);
```

### M-Pesa Callbacks Table
```sql
CREATE TABLE mpesa_callbacks (
    id VARCHAR(50) PRIMARY KEY,
    transaction_id VARCHAR(50),
    merchant_request_id VARCHAR(100),
    checkout_request_id VARCHAR(100),
    result_code INT,
    result_desc TEXT,
    amount DECIMAL(10,2),
    mpesa_receipt_number VARCHAR(50),
    transaction_date DATETIME,
    phone_number VARCHAR(20),
    callback_data JSON,
    processed BOOLEAN DEFAULT FALSE,
    notes TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## IoT Telemetry Schema

### Station Telemetry Payload
```json
{
  "deviceId": "DTN03048",
  "stationId": "ST001",
  "timestamp": "2026-07-30T08:00:00Z",
  "location": {
    "latitude": -1.2882396885696308,
    "longitude": 36.82320792463758
  },
  "batterySlots": [
    {
      "slot": 1,
      "batteryId": "F0F001FF4B",
      "level": 87,
      "status": "available",
      "temperature": 25.5
    },
    {
      "slot": 2,
      "batteryId": "F0F001FF5B",
      "level": 45,
      "status": "rented",
      "temperature": 26.1
    }
  ],
  "system": {
    "temperature": 28.5,
    "network": "online",
    "firmwareVersion": "1.2.3",
    "uptime": 86400,
    "signalStrength": -65
  }
}
```

### Telemetry Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `deviceId` | string | Unique IoT device identifier |
| `stationId` | string | Charging station identifier |
| `timestamp` | datetime | Time of telemetry collection |
| `location.latitude` | float | Station GPS latitude |
| `location.longitude` | float | Station GPS longitude |
| `batterySlots[].slot` | integer | Slot number (1-20) |
| `batterySlots[].batteryId` | string | Unique battery identifier |
| `batterySlots[].level` | integer | Battery percentage (0-100) |
| `batterySlots[].status` | string | available, rented, charging, maintenance |
| `batterySlots[].temperature` | float | Battery temperature in °C |
| `system.temperature` | float | Station temperature in °C |
| `system.network` | string | online, offline, degraded |
| `system.firmwareVersion` | string | Current firmware version |
| `system.uptime` | integer | Seconds since last reboot |
| `system.signalStrength` | integer | Network signal strength (dBm) |

---

## Rental Flow

### 1. Customer Initiates Rental
```mermaid
sequenceDiagram
    Customer->>Mobile App: Scan QR Code
    Mobile App->>Backend: POST /api/payments/initiate
    Backend->>M-Pesa: STK Push Request
    M-Pesa-->>Customer: Enter PIN
    Customer->>M-Pesa: Confirm Payment
```

### 2. Payment Processing
```mermaid
sequenceDiagram
    M-Pesa->>Backend: POST /api/payments/mpesa-callback
    Backend->>Database: Create Transaction
    Backend->>Manufacturer API: Create Order
    Manufacturer API->>Cabinet: Eject Powerbank
    Cabinet-->>Manufacturer API: Ejection Confirmed
    Manufacturer API-->>Backend: tradeNo Response
    Backend-->>Mobile App: Rental Confirmed
```

### 3. Return & Refund
```mermaid
sequenceDiagram
    Customer->>Cabinet: Insert Powerbank
    Cabinet->>Manufacturer API: Return Confirmation
    Manufacturer API->>Backend: POST /api/payments/manufacturer-callback
    Backend->>Database: Calculate Duration
    Backend->>M-Pesa: B2C Refund Request
    M-Pesa-->>Customer: Refund Received
```

---

## API Endpoints

### Payment Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/payments/initiate` | POST | Initiate rental payment with STK push |
| `/api/payments/mpesa-callback` | POST | M-Pesa STK push callback handler |
| `/api/payments/manufacturer-callback` | POST | Manufacturer return callback handler |
| `/api/payments/status` | GET | Check payment and rental status |

### Machine Endpoints
| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/machines/verify-qr` | GET | Verify QR code for machine |
| `/api/manufacturer/stations` | GET | Get nearby stations |

---

## M-Pesa Integration

### STK Push Flow
1. Customer initiates payment
2. System sends STK push to customer's phone
3. Customer enters PIN
4. M-Pesa sends callback to our system
5. System processes payment and creates manufacturer order
6. Powerbank is ejected from cabinet

### B2C Refund Flow
1. Customer returns powerbank
2. Manufacturer sends return callback
3. System calculates duration and cost
4. System sends B2C refund to customer's M-Pesa
5. Customer receives refund

---

## Technology Stack

### Backend
- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: PostgreSQL
- **ORM/Query**: Raw SQL with connection pooling
- **Authentication**: JWT-based authentication
- **Payment Integration**: M-Pesa Daraja API (STK Push & B2C)
- **IoT Integration**: REST APIs & Webhooks

### Frontend (Dashboard)
- **Framework**: React.js
- **State Management**: Redux
- **UI Library**: Material-UI / Ant Design
- **Charts**: Recharts / Chart.js
- **Real-time**: WebSocket / Socket.io

### DevOps
- **Hosting**: Ubuntu Server
- **Process Management**: PM2
- **Reverse Proxy**: Nginx
- **SSL**: Let's Encrypt
- **Monitoring**: PM2 logs, custom logging

---

## Environment Variables

```env
# Server
PORT=5001
BASE_URL=https://rent.chargebyte.io
NODE_ENV=production

# Database
DB_HOST=localhost
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=chargebyte

# M-Pesa
MPESA_CONSUMER_KEY=your_consumer_key
MPESA_CONSUMER_SECRET=your_consumer_secret
MPESA_PASSKEY=your_passkey
MPESA_SHORTCODE=your_shortcode

# Manufacturer API
CHARGEBYTE_API_USERNAME=Chargebyte
CHARGEBYTE_API_PASSWORD=Chargebyte@2026

# B2C
B2C_INITIATOR_NAME=ChargebyteTransfer
B2C_SECURITY_CREDENTIAL=your_security_credential
B2C_TIMEOUT_URL=https://wikiteq.co.ke/api/payments/v1/b2c-timeout
B2C_RESULT_URL=https://wikiteq.co.ke/api/payments/v1/b2c-result
```

---

## Error Handling & Monitoring

### Logging
- Application logs via PM2
- Error logs separated from output logs
- Structured logging with timestamps

### Common Error Codes
| Code | Description |
|------|-------------|
| 504 | Gateway Timeout - Manufacturer API unavailable |
| 1032 | Transaction cancelled by user |
| 2004 | Device not online |

### Recovery Procedures
1. **Failed STK Push**: Retry or user initiates again
2. **Manufacturer API Timeout**: Automatic retry with fallback
3. **Double Ejection Prevention**: Race condition protection
4. **Failed Refund**: Manual intervention possible

---

## Deployment

### Production Deployment
```bash
# Clone repository
git clone https://github.com/chargebyte-africa/chargebyte-backend.git

# Install dependencies
npm install

# Setup environment
cp .env.example .env
nano .env

# Start with PM2
pm2 start server.js --name chargebyte-rental
pm2 save
pm2 startup
```

### Monitoring
```bash
# View logs
pm2 logs chargebyte-rental

# View specific lines
pm2 logs chargebyte-rental --lines 100

# Monitor process
pm2 monit
```

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## License

Proprietary - All rights reserved. Chargebyte Africa © 2026

---

## Contact

- **Website**: https://chargebyte.io
- **Support**: support@chargebyte.io
- **Management**: management@chargebyte.io

---

## Version History

- **v1.0.0** - Initial release with core rental and payment functionality
- **v1.1.0** - Added refund processing with B2C
- **v1.2.0** - Race condition fixes and improved error handling
