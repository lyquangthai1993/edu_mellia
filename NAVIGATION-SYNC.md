# Navigation Auto-Sync Setup

This setup ensures that when you update Post or Page titles in Strapi, the navigation menu in Next.js automatically updates to reflect the changes.

## How It Works

### Strapi Side (Simple)
- **No Redis integration in Strapi** - keeps Strapi lightweight
- Global lifecycle hooks in `/edu_strapi/src/index.ts`
- When Post/Page is created/updated/deleted → triggers HTTP request to Next.js

### Next.js Side (Cached)
- Uses Redis for navigation caching (2 minutes TTL)
- API endpoint `/api/cache/invalidate` receives Strapi notifications
- Clears navigation cache when content changes

## Architecture Flow

```
Strapi Admin Panel
    ↓ (Save Post/Page)
Lifecycle Hook Triggers
    ↓ (HTTP Request)
Next.js API Endpoint
    ↓ (Cache Invalidation)
Navigation Updates Automatically
```

## Files Modified

### Strapi
- `/edu_strapi/src/index.ts` - Global lifecycle hooks
- `/edu_strapi/.env` - Added `NEXTJS_URL=http://localhost:3000`

### Next.js  
- `/edu_nextjs/src/app/api/cache/invalidate/route.ts` - Cache invalidation API
- `/edu_nextjs/src/app/api/cache/test/route.ts` - Test endpoint
- `/edu_nextjs/src/app/api/cache/status/route.ts` - Status monitoring

## Testing

### Manual Test
```bash
# Test navigation cache invalidation
curl http://localhost:3000/api/cache/test

# Check cache status
curl http://localhost:3000/api/cache/status

# API info
curl http://localhost:3000/api/cache/invalidate
```

### Real-World Test
1. Start both applications:
   ```bash
   # Terminal 1: Strapi
   cd edu_strapi && npm run develop
   
   # Terminal 2: Next.js
   cd edu_nextjs && npm run dev
   ```

2. Edit a Post/Page title in Strapi admin
3. Save the changes
4. Check Next.js navigation - it should update automatically!

## Console Output

### When you save content in Strapi:
```
🔄 Content changed: post - My Post Title
📡 Notifying Next.js to update navigation cache...
✅ Next.js navigation cache invalidated successfully
```

### In Next.js console:
```
🗑️ Navigation cache invalidation triggered by post change
📄 Content: My Post Title from strapi-lifecycle
✅ Successfully invalidated navigation cache
```

## Benefits

- ✅ **Automatic sync** - No manual cache clearing needed
- ✅ **Fast updates** - Navigation updates within 2 minutes maximum
- ✅ **Simple Strapi** - No Redis complexity in Strapi
- ✅ **Reliable** - Handles errors gracefully
- ✅ **Scalable** - Can extend to other content types easily

## Configuration

### Environment Variables

**Strapi (`.env`):**
```env
NEXTJS_URL=http://localhost:3000
```

**Next.js (`.env.local`):**
```env
REDIS_HOST=localhost
REDIS_PORT=6379
```

### Production Setup
For production, update `NEXTJS_URL` in Strapi to point to your production Next.js domain:
```env
NEXTJS_URL=https://your-nextjs-app.com
```