# 🚀 Telegram Auto-Sales Bot - Complete Setup Guide

## 📋 Table of Contents
1. [System Architecture Overview](#architecture)
2. [Prerequisites](#prerequisites)
3. [Installation Steps](#installation)
4. [Database Setup](#database)
5. [Bot Configuration](#bot-config)
6. [Web Dashboard Setup](#web-dashboard)
7. [Running the System](#running)
8. [Testing the Discount System](#testing)
9. [Production Deployment](#production)

---

## 🏗️ System Architecture Overview {#architecture}

### Component Separation
```
┌─────────────────────────────────────────────────────────┐
│                  Shared PostgreSQL Database              │
│              (Products, Users, Orders, etc.)             │
└───────────────┬─────────────────────┬───────────────────┘
                │                     │
        ┌───────▼────────┐    ┌───────▼─────────┐
        │  Bot Service   │    │ Web Dashboard   │
        │   (Telegram)   │    │   (FastAPI)     │
        │                │    │                 │
        │ • User Chats   │    │ • Bulk Editor   │
        │ • Admin Panel  │    │ • Analytics     │
        │ • Crypto Watch │    │ • Reports       │
        └────────────────┘    └─────────────────┘
```

### Key Features
- ✅ **Discount System**: Users see strike-through prices (~~100 PLN~~ 80 PLN 🔥)
- ✅ **Atomic Transactions**: Database-level locking prevents race conditions
- ✅ **Dual Management**: Set discounts via Bot OR Web Dashboard
- ✅ **Reservation System**: 15-minute hold with automatic cleanup
- ✅ **Referral Commissions**: 3% automatic payout on deposits

---

## 📦 Prerequisites {#prerequisites}

### System Requirements
- **Python**: 3.10 or higher
- **Database**: PostgreSQL 13+ (or SQLite for testing)
- **Memory**: Minimum 512MB RAM
- **OS**: Linux/Windows/macOS

### Required Accounts
- Telegram Bot Token (from [@BotFather](https://t.me/botfather))
- PostgreSQL database (local or hosted)
- Optional: Bitcoin/Litecoin XPUB for crypto payments

---

## 🔧 Installation Steps {#installation}

### 1. Clone/Create Project Structure
```bash
mkdir telegram_shop
cd telegram_shop

# Create directory structure
mkdir -p bot_app/{handlers/{user,admin},keyboards,states,middleware,services}
mkdir -p web_dashboard/{routers,templates,static/{css,js}}
mkdir -p shared migrations
```

### 2. Install Dependencies
```bash
# Create virtual environment
python3.10 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install packages
pip install aiogram==3.4.1
pip install sqlalchemy[asyncio]==2.0.25
pip install asyncpg==0.29.0
pip install alembic==1.13.1
pip install fastapi==0.109.0
pip install uvicorn[standard]==0.27.0
pip install jinja2==3.1.3
pip install python-dotenv==1.0.0
pip install bcrypt==4.1.2
pip install apscheduler==3.10.4
```

### 3. Create `.env` File
```bash
cat > .env << 'EOF'
# Telegram Bot
BOT_TOKEN=your_bot_token_here
ADMIN_IDS=123456789,987654321  # Your Telegram user IDs

# Database
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/shop_db
# Or for SQLite testing: sqlite+aiosqlite:///./shop.db

# Crypto (Optional)
BTC_XPUB=your_btc_xpub_here
LTC_XPUB=your_ltc_xpub_here

# Web Dashboard
WEB_SECRET_KEY=$(python -c 'import secrets; print(secrets.token_hex(32))')
WEB_ADMIN_USERNAME=admin
WEB_ADMIN_PASSWORD=$(python -c 'import bcrypt; print(bcrypt.hashpw(b"YourPassword123", bcrypt.gensalt()).decode())')

# Redis (Optional, for rate limiting)
REDIS_URL=redis://localhost:6379/0
EOF
```

---

## 🗄️ Database Setup {#database}

### Option A: PostgreSQL (Recommended)
```bash
# Install PostgreSQL (Ubuntu/Debian)
sudo apt install postgresql postgresql-contrib

# Create database and user
sudo -u postgres psql
postgres=# CREATE DATABASE shop_db;
postgres=# CREATE USER shop_user WITH PASSWORD 'your_password';
postgres=# GRANT ALL PRIVILEGES ON DATABASE shop_db TO shop_user;
postgres=# \q
```

### Option B: SQLite (Testing Only)
No installation needed. Just set:
```env
DATABASE_URL=sqlite+aiosqlite:///./shop.db
```

### Initialize Database Schema
```python
# shared/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base
import os

DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_async_engine(DATABASE_URL, echo=True)
async_session_maker = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session():
    async with async_session_maker() as session:
        yield session

async def init_db():
    """Create all tables"""
    from shared.models import Base
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

Run initialization:
```python
# init_db.py
import asyncio
from shared.database import init_db

asyncio.run(init_db())
```

---

## 🤖 Bot Configuration {#bot-config}

### Main Bot Entry Point
```python
# bot_app/main.py
import asyncio
import logging
from aiogram import Bot, Dispatcher
from aiogram.fsm.storage.memory import MemoryStorage
from apscheduler.schedulers.asyncio import AsyncIOScheduler

from bot_app.handlers.user import shop, wallet, profile
from bot_app.handlers.admin import inventory, discounts, users
from bot_app.middleware.auth import AdminAuthMiddleware
from bot_app.middleware.throttling import ThrottlingMiddleware
from shared.database import init_db
from config import BOT_TOKEN

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def main():
    # Initialize database
    await init_db()
    logger.info("Database initialized")
    
    # Initialize bot
    bot = Bot(token=BOT_TOKEN, parse_mode="HTML")
    dp = Dispatcher(storage=MemoryStorage())
    
    # Register middlewares
    dp.message.middleware(ThrottlingMiddleware())
    dp.callback_query.middleware(ThrottlingMiddleware())
    
    # Register routers
    dp.include_router(shop.router)
    dp.include_router(wallet.router)
    dp.include_router(profile.router)
    dp.include_router(inventory.router)
    dp.include_router(discounts.router)
    dp.include_router(users.router)
    
    # Setup scheduler for reservation cleanup
    scheduler = AsyncIOScheduler()
    scheduler.add_job(
        shop.scheduled_reservation_cleanup,
        'interval',
        minutes=1,
        id='cleanup_reservations'
    )
    scheduler.start()
    logger.info("Scheduler started")
    
    # Start polling
    logger.info("Bot started!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
```

### Run the Bot
```bash
python bot_app/main.py
```

---

## 🌐 Web Dashboard Setup {#web-dashboard}

### Configuration File
```python
# web_dashboard/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    WEB_SECRET_KEY: str
    WEB_ADMIN_USERNAME: str
    WEB_ADMIN_PASSWORD: str
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### Run Web Dashboard
```bash
# Development mode
uvicorn web_dashboard.main:app --reload --host 0.0.0.0 --port 8000

# Production mode
uvicorn web_dashboard.main:app --host 0.0.0.0 --port 8000 --workers 4
```

Access at: `http://localhost:8000`

---

## 🏃 Running the System {#running}

### Development Mode (Two Terminals)

**Terminal 1: Bot**
```bash
source venv/bin/activate
python bot_app/main.py
```

**Terminal 2: Web Dashboard**
```bash
source venv/bin/activate
uvicorn web_dashboard.main:app --reload --port 8000
```

### Production Mode (Using Systemd)

**Bot Service** (`/etc/systemd/system/telegram-bot.service`):
```ini
[Unit]
Description=Telegram Shop Bot
After=network.target postgresql.service

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/telegram_shop
Environment="PATH=/home/botuser/telegram_shop/venv/bin"
ExecStart=/home/botuser/telegram_shop/venv/bin/python bot_app/main.py
Restart=always

[Install]
WantedBy=multi-user.target
```

**Web Dashboard Service** (`/etc/systemd/system/shop-web.service`):
```ini
[Unit]
Description=Shop Web Dashboard
After=network.target postgresql.service

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/telegram_shop
Environment="PATH=/home/botuser/telegram_shop/venv/bin"
ExecStart=/home/botuser/telegram_shop/venv/bin/uvicorn web_dashboard.main:app --host 0.0.0.0 --port 8000 --workers 4
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable telegram-bot shop-web
sudo systemctl start telegram-bot shop-web
```

---

## 🧪 Testing the Discount System {#testing}

### Test Scenario 1: Bot Admin Panel

1. Send `/set_discount` to bot
2. Navigate: City → District → Category → Product
3. Enter discount: `20` (for 20% off)
4. Verify display shows: "~~100 PLN~~ 80 PLN 🔥 -20%"

### Test Scenario 2: Web Dashboard Bulk Edit

1. Open `http://localhost:8000/products`
2. Login with admin credentials
3. Select multiple products (checkboxes)
4. Enter bulk discount: `30`
5. Click "Apply to Selected"
6. Verify products update

### Test Scenario 3: Atomic Purchase

**Setup:**
- Product: Base price 100 PLN, 20% discount → Final 80 PLN
- User A: Balance 85 PLN
- User B: Balance 85 PLN

**Execution (Simultaneous):**
1. Both users click "Buy" on same product
2. Product gets reserved for User A (first to click)
3. User B sees "Reserved by another user"
4. User A confirms purchase
5. Balance: 85 - 80 = 5 PLN remaining
6. User A receives delivery info
7. Product marked as sold

**Verification:**
```sql
-- Check order record
SELECT * FROM orders WHERE product_id = 123;
-- Should show: original_price=100, discount_applied=20, final_price=80

-- Check user balance
SELECT balance FROM users WHERE id = USER_A_ID;
-- Should show: 5.00
```

---

## 🚀 Production Deployment {#production}

### Security Checklist

#### 1. Database
- [ ] Use strong PostgreSQL password
- [ ] Enable SSL connections
- [ ] Restrict access to localhost or specific IPs
- [ ] Regular automated backups

#### 2. Web Dashboard
- [ ] Setup Nginx reverse proxy with HTTPS
- [ ] Use strong admin password (bcrypt hashed)
- [ ] Enable rate limiting
- [ ] Add IP whitelist if possible
- [ ] Enable CORS only for your domain

#### 3. Bot
- [ ] Keep BOT_TOKEN in `.env` (never commit)
- [ ] Validate all admin commands by user ID
- [ ] Log all admin actions
- [ ] Implement webhook instead of polling for production

### Nginx Configuration Example
```nginx
server {
    listen 443 ssl http2;
    server_name shop-admin.yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Monitoring & Maintenance

**Daily Tasks:**
- Check bot logs: `journalctl -u telegram-bot -f`
- Monitor expired reservations cleanup
- Review failed deposits

**Weekly Tasks:**
- Database backup
- Disk space check
- Review active discounts

**Monthly Tasks:**
- Security updates
- Performance optimization
- User engagement analysis

---

## 🎯 Key Features Summary

### ✅ Discount System
- **Bot Admin**: FSM wizard to set discounts per product
- **Web Dashboard**: Bulk editor for multiple products
- **User Display**: Strike-through pricing with fire emoji
- **Database**: `discount_percent` field (0-100)
- **Calculation**: Atomic, happens at checkout

### ✅ Security
- **Atomic Transactions**: `SELECT FOR UPDATE` prevents race conditions
- **Balance Validation**: Check before deduction
- **Reservation System**: 15-minute time-bound holds
- **Admin Auth**: User ID verification + HTTP Basic Auth

### ✅ Scalability
- **Async Architecture**: Handles thousands of concurrent users
- **Database Indexes**: Optimized queries on location and status
- **Background Tasks**: APScheduler for cleanup
- **Separation of Concerns**: Bot and Web share database but run independently

---

## 📞 Support & Next Steps

### Additional Features to Implement
1. **Analytics Dashboard**: Sales charts, top products
2. **Automated Reports**: Daily/weekly sales summaries
3. **Customer Support Chat**: Integrate with support ticket system
4. **Mobile App**: React Native frontend
5. **Multi-language**: i18n support

### Documentation
- API documentation: Visit `/docs` when web dashboard is running
- Database schema: See `shared/models.py`
- State machine flows: See `bot_app/states/`

---

**🎉 Your Telegram Auto-Sales Bot with Discount System is ready!**

**Start the bot**: `/start` → Explore the shop  
**Admin panel**: `/set_discount` → Manage promotions  
**Web dashboard**: `http://localhost:8000` → Bulk operations