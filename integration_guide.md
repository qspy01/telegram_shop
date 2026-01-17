# 🔗 Complete System Integration Guide

## 📋 Overview of All Features

Your Telegram Auto-Sales Bot now includes:

### ✅ Core Features
1. **Discount System** - Strike-through pricing with percentage off
2. **Atomic Transactions** - Database-level locking prevents race conditions
3. **Reservation System** - 15-minute holds with automatic cleanup
4. **Dual Management** - Bot admin panel + Web dashboard

### ✅ Payment Systems
5. **Crypto Auto-Deposit** - Bitcoin/Litecoin with blockchain monitoring
6. **Manual Deposits** - BLIK/Bank transfer with admin approval
7. **Balance Management** - Internal wallet system

### ✅ Engagement Features
8. **Referral Program** - 3% commission on referee deposits
9. **Tier System** - Bronze → Silver → Gold → Platinum → Diamond
10. **Achievements** - Unlock badges based on activity
11. **Leaderboard** - Top referrers ranking

---

## 🏗️ Main Bot Entry Point

### `bot_app/main.py` - Complete Integration

```python
"""
bot_app/main.py - Main Bot Application
=======================================
"""

import asyncio
import logging
import sys
from aiogram import Bot, Dispatcher
from aiogram.fsm.storage.memory import MemoryStorage
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# Configuration
from bot_app.config import settings

# Database
from shared.database import init_db

# Handlers
from bot_app.handlers.user import start, shop, wallet, profile
from bot_app.handlers.admin import inventory, discounts, users as admin_users, broadcast
from bot_app.services.referral import router as referral_router

# Services
from bot_app.services.crypto_listener import start_crypto_monitor
from bot_app.handlers.user.shop import scheduled_reservation_cleanup

# Middleware
from bot_app.middleware.throttling import ThrottlingMiddleware
from bot_app.middleware.auth import AdminAuthMiddleware


# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('bot.log')
    ]
)
logger = logging.getLogger(__name__)


# Global bot instance (for services to use)
bot: Bot = None


async def on_startup():
    """Execute on bot startup"""
    logger.info("🚀 Bot starting up...")
    
    # Initialize database
    await init_db()
    logger.info("✓ Database initialized")
    
    # Start crypto payment monitor
    await start_crypto_monitor()
    logger.info("✓ Crypto payment monitor started")
    
    logger.info("✅ Bot startup complete!")


async def on_shutdown():
    """Execute on bot shutdown"""
    logger.info("🛑 Bot shutting down...")
    
    # Stop crypto monitor
    from bot_app.services.crypto_listener import get_payment_monitor
    monitor = get_payment_monitor()
    monitor.stop()
    
    logger.info("✅ Shutdown complete")


async def main():
    """Main bot execution"""
    global bot
    
    # Initialize bot and dispatcher
    bot = Bot(token=settings.BOT_TOKEN, parse_mode="HTML")
    dp = Dispatcher(storage=MemoryStorage())
    
    # Register middlewares
    dp.message.middleware(ThrottlingMiddleware(rate_limit=2))  # 2 messages per second
    dp.callback_query.middleware(ThrottlingMiddleware(rate_limit=5))
    
    # Register user handlers
    dp.include_router(start.router)
    dp.include_router(shop.router)
    dp.include_router(wallet.router)
    dp.include_router(wallet.admin_router)  # Admin balance management
    dp.include_router(profile.router)
    dp.include_router(referral_router)
    
    # Register admin handlers
    dp.include_router(inventory.router)
    dp.include_router(discounts.router)
    dp.include_router(admin_users.router)
    dp.include_router(broadcast.router)
    
    # Setup scheduler for background tasks
    scheduler = AsyncIOScheduler()
    
    # Cleanup expired reservations every minute
    scheduler.add_job(
        scheduled_reservation_cleanup,
        'interval',
        minutes=1,
        id='cleanup_reservations'
    )
    
    # Update crypto exchange rates every 10 minutes
    scheduler.add_job(
        update_exchange_rates,
        'interval',
        minutes=10,
        id='update_rates'
    )
    
    scheduler.start()
    logger.info("✓ Scheduler started")
    
    # Startup
    await on_startup()
    
    try:
        # Start polling
        logger.info("📡 Starting polling...")
        await dp.start_polling(bot, allowed_updates=dp.resolve_used_update_types())
    finally:
        await on_shutdown()
        await bot.session.close()


async def update_exchange_rates():
    """Background task to update crypto exchange rates"""
    from bot_app.services.crypto_listener import get_payment_monitor
    monitor = get_payment_monitor()
    await monitor.update_exchange_rates()


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Bot stopped by user")
```

---

## 📝 Configuration File

### `bot_app/config.py`

```python
"""
bot_app/config.py - Configuration Management
=============================================
"""

from pydantic_settings import BaseSettings
from typing import List


class Settings(BaseSettings):
    """Application settings from environment variables"""
    
    # Telegram Bot
    BOT_TOKEN: str
    ADMIN_IDS: str  # Comma-separated list
    
    # Database
    DATABASE_URL: str
    
    # Crypto
    BTC_XPUB: str = None
    LTC_XPUB: str = None
    BLOCKCHAIN_API_KEY: str = None
    
    # Web Dashboard
    WEB_SECRET_KEY: str
    WEB_ADMIN_USERNAME: str
    WEB_ADMIN_PASSWORD: str
    
    # Redis (optional)
    REDIS_URL: str = None
    
    class Config:
        env_file = ".env"
        env_file_encoding = 'utf-8'
    
    @property
    def admin_ids_list(self) -> List[int]:
        """Parse admin IDs from comma-separated string"""
        return [int(x.strip()) for x in self.ADMIN_IDS.split(',') if x.strip()]


settings = Settings()
```

---

## 🧪 Complete Testing Scenarios

### Test 1: User Registration with Referral

**Setup:**
1. User A sends `/start`
2. Gets referral link: `https://t.me/YourBot?start=ref_ABC123XY`
3. User B clicks link

**Expected:**
- User B registered with `referrer_id = User A's ID`
- User A receives notification: "New Referral!"
- User B sees welcome message mentioning User A

**Verification:**
```sql
SELECT id, referrer_id, referral_code FROM users WHERE id = USER_B_ID;
-- Should show User A's ID in referrer_id
```

---

### Test 2: Crypto Deposit with Referral Commission

**Setup:**
- User B (referred by User A) deposits 100 PLN via BTC

**Steps:**
1. User B: `/wallet` → "Deposit" → "Bitcoin"
2. Receives BTC address: `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa`
3. Sends 0.0005 BTC (= ~100 PLN at current rate)
4. Wait for blockchain confirmation (~10 minutes)

**Expected:**
- Payment monitor detects transaction
- User B balance: +100 PLN
- User A balance: +3 PLN (3% commission)
- User A referral_earnings: +3 PLN
- Both receive notification messages

**Verification:**
```sql
-- Check deposit
SELECT * FROM deposits WHERE user_id = USER_B_ID ORDER BY created_at DESC LIMIT 1;
-- status should be 'completed'

-- Check balances
SELECT id, balance, referral_earnings FROM users WHERE id IN (USER_A_ID, USER_B_ID);
-- User B: balance = 100.00
-- User A: referral_earnings = 3.00
```

---

### Test 3: Purchase with Discount (Atomic Transaction)

**Setup:**
- Product: Base price 100 PLN, 20% discount
- User balance: 85 PLN

**Steps:**
1. Admin sets discount: `/set_discount` → Select product → Enter `20`
2. User views product → Sees "~~100 PLN~~ 80 PLN 🔥 -20%"
3. User clicks "Buy" → Product reserved
4. User clicks "Confirm Purchase"

**Expected:**
- Balance check: 85 >= 80 ✓ (passes)
- User balance: 85 - 80 = 5 PLN
- Order created with:
  - original_price = 100
  - discount_applied = 20
  - final_price = 80
- User receives delivery info (photo, coords, description)

**Verification:**
```sql
-- Check order
SELECT original_price, discount_applied, final_price FROM orders 
WHERE user_id = USER_ID ORDER BY created_at DESC LIMIT 1;
-- Should show: 100.00, 20, 80.00

-- Check product
SELECT is_sold, sold_at FROM products WHERE id = PRODUCT_ID;
-- is_sold = true, sold_at = recent timestamp
```

---

### Test 4: Race Condition Prevention

**Setup:**
- Product A: Price 50 PLN
- User X: Balance 50 PLN
- User Y: Balance 50 PLN

**Steps:**
1. Both users click "Buy" on Product A simultaneously
2. User X's request reaches server first (by milliseconds)

**Expected:**
- Database lock acquired by User X's transaction
- User X completes purchase successfully
- User Y's request waits for lock
- When lock released, User Y sees: "Sorry, this product was just sold!"
- Only ONE order created
- Only ONE user charged

**Verification:**
```sql
-- Check orders
SELECT COUNT(*) FROM orders WHERE product_id = PRODUCT_A_ID;
-- Should be exactly 1

-- Check balances
SELECT id, balance FROM users WHERE id IN (USER_X_ID, USER_Y_ID);
-- User X: 0.00 (50 - 50)
-- User Y: 50.00 (unchanged)
```

---

### Test 5: Referral Tier Progression

**Setup:**
- User A has 0 referral earnings

**Steps:**
1. User A invites 10 users
2. Each deposits 100 PLN
3. Total deposits: 1000 PLN
4. Commission: 1000 * 3% = 30 PLN

**Expected Tier Progression:**
```
   0 PLN → 🥉 Bronze
  30 PLN → 🥉 Bronze (still)
 100 PLN → 🥈 Silver
 500 PLN → 🥇 Gold
1000 PLN → 💎 Platinum
5000 PLN → 👑 Diamond
```

**Verification:**
```python
from bot_app.services.referral import ReferralTier

tier_name, tier_key = ReferralTier.get_tier(Decimal('30.00'))
assert tier_name == "🥉 Bronze"

tier_name, tier_key = ReferralTier.get_tier(Decimal('500.00'))
assert tier_name == "🥇 Gold"
```

---

### Test 6: Web Dashboard Bulk Discount

**Setup:**
- 50 products in "Warsaw → Center" location
- Admin wants to apply 30% discount to all

**Steps:**
1. Login to web dashboard: `http://localhost:8000`
2. Navigate to Products page
3. Filter by city: "Warsaw"
4. Click "Select All" (50 products selected)
5. Enter bulk discount: `30`
6. Click "Apply to Selected"

**Expected:**
- API call: POST `/api/discount/bulk`
- Database updates all 50 products
- Response: "Updated 50 products with 30% discount"
- Bot users now see all products with discount formatting

**Verification:**
```sql
SELECT COUNT(*) FROM products 
WHERE city = 'Warsaw' AND district = 'Center' AND discount_percent = 30;
-- Should return 50
```

---

## 🔍 Monitoring & Debugging

### Log Analysis

**Check crypto payments:**
```bash
grep "Payment detected" bot.log
# Output: 💰 Payment detected! User 123456 received 0.0005 BTC = 100.00 PLN
```

**Check referral commissions:**
```bash
grep "Referral commission" bot.log
# Output: Referral commission: 3.00 PLN to user 789012
```

**Check reservation cleanup:**
```bash
grep "CLEANUP" bot.log
# Output: [CLEANUP] Released 3 expired reservations
```

### Database Queries

**Find users with highest referral earnings:**
```sql
SELECT id, username, referral_earnings, 
       (SELECT COUNT(*) FROM users WHERE referrer_id = u.id) as referral_count
FROM users u
WHERE referral_earnings > 0
ORDER BY referral_earnings DESC
LIMIT 10;
```

**Products with active discounts:**
```sql
SELECT city, district, name, base_price, discount_percent,
       ROUND(base_price * (1 - discount_percent::decimal / 100), 2) as final_price
FROM products
WHERE discount_percent > 0 AND is_sold = false
ORDER BY discount_percent DESC;
```

**Daily sales report:**
```sql
SELECT 
    DATE(created_at) as date,
    COUNT(*) as orders,
    SUM(final_price) as revenue,
    SUM(original_price - final_price) as total_discounts
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

---

## 🚨 Common Issues & Solutions

### Issue 1: Crypto Payments Not Detected

**Symptoms:**
- User deposits BTC but balance doesn't update
- No notification received

**Debug Steps:**
```bash
# Check if monitor is running
ps aux | grep crypto_listener

# Check recent logs
tail -f bot.log | grep "monitor"

# Manually trigger address check
python -c "
from bot_app.services.crypto_listener import get_payment_monitor
import asyncio
monitor = get_payment_monitor()
asyncio.run(monitor.monitor_all_addresses())
"
```

**Common Causes:**
- Invalid XPUB key in `.env`
- Blockchain API rate limit exceeded
- Address not saved in database

---

### Issue 2: Race Condition Still Occurring

**Symptoms:**
- Same product sold to two users
- Both users charged

**Verification:**
```sql
-- Check for duplicate orders
SELECT product_id, COUNT(*) 
FROM orders 
GROUP BY product_id 
HAVING COUNT(*) > 1;
```

**Solution:**
- Ensure `with_for_update()` is used in buy handler
- Check PostgreSQL isolation level: `SHOW transaction_isolation;`
- Should be `read committed` or `repeatable read`

---

### Issue 3: Referral Commission Not Credited

**Symptoms:**
- Referee deposits but referrer doesn't receive 3%

**Debug:**
```sql
-- Check referrer relationship
SELECT id, username, referrer_id FROM users WHERE id = REFEREE_ID;

-- Check deposit record
SELECT user_id, amount, status FROM deposits WHERE user_id = REFEREE_ID;

-- Check referrer balance changes
SELECT balance, referral_earnings FROM users WHERE id = REFERRER_ID;
```

**Common Causes:**
- `referrer_id` is NULL (user didn't register via referral link)
- Deposit status is 'pending' (commission only on 'completed')
- Bug in commission calculation in `crypto_listener.py` or `wallet.py`

---

## 📊 Performance Optimization

### Database Indexes

Ensure these indexes exist for optimal performance:

```sql
-- User lookups
CREATE INDEX idx_users_referral ON users(referral_code);
CREATE INDEX idx_users_referrer ON users(referrer_id);

-- Product filtering
CREATE INDEX idx_products_location ON products(city, district, category);
CREATE INDEX idx_products_available ON products(is_sold, is_reserved) WHERE is_sold = false;
CREATE INDEX idx_products_discount ON products(discount_percent) WHERE discount_percent > 0;

-- Order history
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_product ON orders(product_id);

-- Deposits
CREATE INDEX idx_deposits_user_status ON deposits(user_id, status);
CREATE INDEX idx_deposits_tx_hash ON deposits(tx_hash);

-- Crypto addresses
CREATE INDEX idx_crypto_active ON crypto_addresses(currency, is_active) WHERE is_active = true;
```

### Caching Strategy

For high-traffic scenarios, consider Redis caching:

```python
# Cache product catalog (expires every 5 minutes)
from redis import asyncio as aioredis
import json

redis = aioredis.from_url(settings.REDIS_URL)

async def get_products_cached(city, district, category):
    cache_key = f"products:{city}:{district}:{category}"
    
    # Try cache first
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Query database
    products = await get_available_products(session, city, district, category)
    
    # Cache for 5 minutes
    await redis.setex(cache_key, 300, json.dumps([p.to_dict() for p in products]))
    
    return products
```

---

## ✅ Production Deployment Checklist

### Pre-Deployment

- [ ] All tests passing (run `pytest`)
- [ ] Database migrations applied (`alembic upgrade head`)
- [ ] Environment variables set in `.env`
- [ ] Crypto XPUB keys validated
- [ ] Admin IDs configured correctly
- [ ] Web dashboard password is strong (bcrypt hashed)

### Security

- [ ] PostgreSQL uses strong password
- [ ] Database accepts connections only from localhost
- [ ] Nginx configured with HTTPS (Let's Encrypt)
- [ ] Web dashboard behind HTTP Basic Auth
- [ ] Rate limiting enabled (middleware + nginx)
- [ ] Bot token kept secret (never committed to git)

### Monitoring

- [ ] Systemd services running (`systemctl status telegram-bot shop-web`)
- [ ] Log rotation configured (`/etc/logrotate.d/telegram-bot`)
- [ ] Disk space monitoring (10GB+ free recommended)
- [ ] Database backups scheduled (daily via cron)
- [ ] Uptime monitoring (UptimeRobot, Pingdom, etc.)

### Performance

- [ ] Database indexes created
- [ ] Connection pooling configured (SQLAlchemy `pool_size=20`)
- [ ] Redis cache enabled for hot paths
- [ ] Crypto monitor check interval tuned (60s default)
- [ ] Web dashboard using multiple workers (`--workers 4`)

---

## 🎉 Your Bot is Ready!

**Features Delivered:**
✅ Discount system with strike-through pricing  
✅ Atomic transactions preventing race conditions  
✅ Crypto auto-deposit (BTC/LTC)  
✅ 3% referral commission system  
✅ Tier progression (Bronze → Diamond)  
✅ Achievement badges  
✅ Web dashboard with bulk editor  
✅ Comprehensive admin panel  
✅ Real-time payment monitoring  

**Next Steps:**
1. Deploy to production server
2. Test all features end-to-end
3. Monitor logs for first 24 hours
4. Gather user feedback
5. Iterate and improve!

**Support:**
- Check logs: `journalctl -u telegram-bot -f`
- Database console: `psql -U user -d shop_db`
- Web dashboard: `http://your-server:8000`

Good luck with your Telegram shop! 🚀