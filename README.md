# Reddit Pixel Shopify Server-Side Tracking

Server-side conversion tracking for Reddit Ads using Shopify webhooks and Reddit Conversions API. This implementation bypasses browser-based tracking limitations (ad blockers, privacy settings, iOS restrictions) to capture conversions that client-side pixels miss.

## Why Server-Side Tracking?

Reddit officially recommends using both Pixel and Conversions API together for optimal tracking accuracy. According to their documentation:

> "CAPI is more resilient to signal loss because it operates server-side, making it less susceptible to ad blockers and browser restrictions."

Client-side pixel tracking loses 30-50% of conversions due to:
- Ad blockers
- Browser privacy settings (Safari, Firefox)
- iOS App Tracking Transparency
- Third-party cookie blocking

This server-side implementation captures conversions that browser-based tracking misses by sending data directly from your server to Reddit's API.

## Features

- ✅ Production-ready Docker setup
- ✅ HMAC webhook verification (Shopify security)
- ✅ PII hashing (SHA-256 for privacy)
- ✅ Event deduplication (PostgreSQL)
- ✅ Automatic retry logic with exponential backoff
- ✅ Rate limiting handling (429 responses)
- ✅ Comprehensive logging (Winston)
- ✅ Health check endpoint
- ✅ SSL/HTTPS support (Nginx reverse proxy example)

## Quick Start

**Prerequisites:**
- Linux server with Docker installed
- Domain/subdomain pointing to your server
- Reddit Ads account with Pixel created
- Shopify store with admin access

**Installation (5 minutes):**

```bash
# Clone repository
git clone https://github.com/91369673/reddit-pixel-shopify-server-side-tracking.git
cd reddit-pixel-shopify-server-side-tracking

# Copy environment template
cp .env.example .env

# Edit .env with your credentials
nano .env

# Start services
docker compose up -d

# Check logs
docker compose logs -f reddit-capi
```

## Configuration

### 1. Reddit Ads Manager

1. Go to Reddit Ads Manager → Events Manager
2. Create new Pixel
3. Copy **Pixel ID** (format: `t2_abc123`)
4. Go to Settings → Conversions API
5. Generate **Access Token**
6. Add to `.env`:

```env
REDDIT_PIXEL_ID=t2_your_pixel_id
REDDIT_ACCESS_TOKEN=your_access_token_here
```

### 2. Shopify Webhooks

1. Shopify Admin → Settings → Notifications → Webhooks
2. Create webhook:
   - **Event:** Order creation
   - **Format:** JSON
   - **URL:** `https://your-domain.com/webhooks/shopify`
   - **API version:** Latest
3. Copy **Signing Secret** → add to `.env`:

```env
SHOPIFY_WEBHOOK_SECRET=your_webhook_secret
```

4. Restart: `docker compose restart reddit-capi`

### 3. Nginx Reverse Proxy (Optional but Recommended)

Create `/etc/nginx/sites-available/reddit-tracking.conf`:

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Enable and reload:
```bash
ln -s /etc/nginx/sites-available/reddit-tracking.conf /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## Architecture

```
Shopify Store (order created)
    |
    v
Shopify Webhook (HTTPS POST)
    |
    v
Nginx Reverse Proxy (SSL termination)
    |
    v
Node.js Application (Docker container)
    |
    v
1. HMAC Verification
2. Event Transformation
3. PII Hashing (SHA-256)
4. Deduplication Check
5. Send to Reddit CAPI
6. Log Result
    |
    v
Reddit Ads Platform
```

## Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `SHOPIFY_WEBHOOK_SECRET` | Yes | Shopify webhook signing secret | `ba32d1d12da1ffc...` |
| `REDDIT_PIXEL_ID` | Yes | Reddit Pixel ID from Events Manager | `t2_abc123` |
| `REDDIT_ACCESS_TOKEN` | Yes | Reddit Conversions API access token | `eyJhbGciOiJSUzI1...` |
| `DB_PASSWORD` | Yes | PostgreSQL database password | `strong_password` |
| `PORT` | No | Application port (default: 3000) | `3000` |

## Testing

**Health check:**
```bash
curl https://your-domain.com/health
```

**Expected response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-11-22T12:00:00.000Z",
  "uptime": 12345.67,
  "database": "connected"
}
```

**Send test webhook from Shopify:**
1. Shopify Admin → Settings → Notifications → Webhooks
2. Click on your webhook
3. "Send test notification"
4. Check logs: `docker compose logs -f reddit-capi`

**Verify in Reddit:**
1. Reddit Ads Manager → Events Manager
2. Click your Pixel → "Test Events" tab
3. Event should appear within a few minutes

## Monitoring

**View logs:**
```bash
docker compose logs -f reddit-capi
```

**Check recent events:**
```bash
docker compose exec postgres psql -U reddit_capi -d reddit_capi -c \
  "SELECT event_id, event_type, status, created_at FROM events ORDER BY created_at DESC LIMIT 10;"
```

**Failed events:**
```bash
docker compose exec postgres psql -U reddit_capi -d reddit_capi -c \
  "SELECT event_id, error_message FROM events WHERE status = 'failed';"
```

## Troubleshooting

### Events Not Appearing in Reddit

**Check logs for API errors:**
```bash
docker compose logs reddit-capi | grep "Reddit API error"
```

**Common errors:**

| Error | Cause | Solution |
|-------|-------|----------|
| "unexpected type number" | Value sent as decimal | Verify multiplication by 100 in transformer |
| "unknown field event_id" | Invalid field in payload | Remove `event_id` from Reddit event object |
| 401 Unauthorized | Invalid access token | Regenerate token in Reddit Events Manager |
| 429 Too Many Requests | Rate limiting | Handled automatically with retry logic |

### HMAC Verification Failed

```bash
# Check if secret matches
docker compose exec reddit-capi printenv SHOPIFY_WEBHOOK_SECRET

# Update .env and restart
docker compose restart reddit-capi
```

### Database Connection Issues

```bash
# Check PostgreSQL status
docker compose ps postgres

# Test connection
docker compose exec postgres pg_isready -U reddit_capi
```

## Database Schema

```sql
CREATE TABLE events (
  event_id VARCHAR(255) PRIMARY KEY,
  event_type VARCHAR(50) NOT NULL,
  shopify_id VARCHAR(255) NOT NULL,
  shopify_payload JSONB,
  reddit_payload JSONB,
  reddit_response JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP,
  status VARCHAR(20) DEFAULT 'pending',
  error_message TEXT
);
```

## Security Features

1. **HMAC Verification:** Every webhook validated with SHA-256 HMAC
2. **HTTPS Only:** TLS 1.2/1.3 encryption
3. **Localhost Binding:** Container only accessible via reverse proxy
4. **PII Hashing:** Email/phone SHA-256 hashed before sending to Reddit
5. **Environment Secrets:** All sensitive data in `.env` (git-ignored)

## Advanced Configuration

### Multi-Currency Support

The implementation automatically handles different currencies. For currencies with non-standard decimal places:

```javascript
// Zero-decimal currencies (JPY, KRW): multiply by 1
// Standard currencies (USD, EUR): multiply by 100
// Three-decimal currencies (BHD, KWD): multiply by 1000

value: Math.round(parseFloat(order.total_price) * 100)
```

### Additional Event Types

Extend `src/services/shopifyTransformer.js` for more events:

```javascript
export function transformProductView(product) {
  return {
    event_at: new Date().toISOString(),
    event_type: { tracking_type: 'ViewContent' },
    // ... user data
    event_metadata: {
      conversion_id: `product_${product.id}_${Date.now()}`,
      item_id: String(product.id),
    },
  };
}
```

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test your changes
4. Submit a pull request

## Support

- **Issues:** [GitHub Issues](https://github.com/91369673/reddit-pixel-shopify-server-side-tracking/issues)
- **Documentation:** [Full setup guide](https://www.tva.sg/reddit-pixel-shopify-tracking)
- **Reddit API Docs:** [Reddit Conversions API](https://reddithelp.com/hc/en-us/articles/4410892225940-Reddit-Conversions-API)

## Acknowledgments

Based on production implementation for live Shopify stores. Tested with:
- Reddit Conversions API v2.0
- Shopify API 2025-01
- Docker Compose v2.x
- PostgreSQL 16
- Node.js 18

---

**⭐ If this helped you, consider starring the repo!**
