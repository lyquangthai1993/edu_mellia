# CLAUDE.DEPLOYMENT.md - Deployment and Production Configuration

## Deployment Strategy Overview

This document covers the deployment and production configuration for the Next.js + Strapi application stack.

## Frontend Deployment (Vercel)

### Setup Steps

1. **Connect Repository**
   - Connect GitHub repository to Vercel
   - Select the `edu_nextjs` directory as the root folder
   - Configure build settings

2. **Environment Variables**
   Configure the following environment variables in Vercel dashboard:
   ```env
   NEXTAUTH_URL=https://your-domain.vercel.app
   NEXTAUTH_SECRET=your-production-secret-key
   
   NEXT_PUBLIC_STRAPI_API_URL=https://your-strapi-domain.com/api
   STRAPI_API_URL=https://your-strapi-domain.com/api
   
   REDIS_HOST=your-redis-host
   REDIS_PORT=6379
   REDIS_PASSWORD=your-redis-password
   ```

3. **Build Configuration**
   - Framework Preset: Next.js
   - Build Command: `npm run build`
   - Output Directory: `.next`
   - Install Command: `npm install`

4. **Domain Configuration**
   - Add custom domain in Vercel dashboard
   - Configure DNS records
   - Enable automatic HTTPS

### Vercel Deployment Optimizations

```javascript
// vercel.json
{
  "functions": {
    "app/api/**/*.js": {
      "maxDuration": 30
    }
  },
  "regions": ["iad1"],
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "installCommand": "npm install",
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        }
      ]
    }
  ]
}
```

## Backend Deployment (Railway/DigitalOcean)

### Railway Deployment

1. **Database Setup**
   ```bash
   # Create PostgreSQL database on Railway
   railway add postgresql
   
   # Get database connection string
   railway variables
   ```

2. **Strapi Deployment**
   ```bash
   # Login to Railway
   railway login
   
   # Link project
   railway link
   
   # Deploy
   railway up
   ```

3. **Environment Variables (Railway)**
   ```env
   # Database
   DATABASE_URL=postgresql://user:password@host:port/database
   DATABASE_CLIENT=postgres
   
   # Strapi Secrets
   APP_KEYS=key1,key2,key3,key4
   API_TOKEN_SALT=your-api-token-salt
   ADMIN_JWT_SECRET=your-admin-jwt-secret
   TRANSFER_TOKEN_SALT=your-transfer-token-salt
   JWT_SECRET=your-jwt-secret
   
   # Redis
   REDIS_URL=redis://user:password@host:port
   
   # Media Storage
   CLOUDINARY_NAME=your-cloudinary-name
   CLOUDINARY_KEY=your-cloudinary-key
   CLOUDINARY_SECRET=your-cloudinary-secret
   
   # Application
   NODE_ENV=production
   HOST=0.0.0.0
   PORT=1337
   ```

### DigitalOcean App Platform Deployment

1. **App Specification**
   ```yaml
   # .do/app.yaml
   name: edu-strapi
   services:
   - name: strapi
     source_dir: /edu_strapi
     github:
       repo: your-username/edu_mellia
       branch: main
     run_command: npm start
     environment_slug: node-js
     instance_count: 1
     instance_size_slug: basic-xxs
     envs:
     - key: NODE_ENV
       value: production
     - key: DATABASE_URL
       value: ${db.DATABASE_URL}
     - key: APP_KEYS
       value: ${APP_KEYS}
   databases:
   - name: db
     engine: PG
     version: "13"
   ```

2. **Build Configuration**
   ```json
   // package.json (add to scripts)
   {
     "scripts": {
       "start": "strapi start",
       "build": "strapi build",
       "develop": "strapi develop"
     }
   }
   ```

## Database Configuration

### PostgreSQL Production Setup

```javascript
// config/database.js
module.exports = ({ env }) => {
  // Parse DATABASE_URL for services like Railway, Heroku
  const parse = require('pg-connection-string').parse;
  const config = parse(env('DATABASE_URL'));

  return {
    connection: {
      client: 'postgres',
      connection: {
        host: config.host,
        port: config.port,
        database: config.database,
        user: config.user,
        password: config.password,
        ssl: env.bool('DATABASE_SSL', true) && {
          rejectUnauthorized: env.bool('DATABASE_SSL_REJECT_UNAUTHORIZED', false),
        },
      },
      debug: false,
      pool: {
        min: 2,
        max: 10,
        acquireTimeoutMillis: 60000,
        createTimeoutMillis: 30000,
        destroyTimeoutMillis: 5000,
        idleTimeoutMillis: 30000,
        reapIntervalMillis: 1000,
        createRetryIntervalMillis: 100,
      },
    },
  };
};
```

### Redis Configuration

```javascript
// config/redis.js (custom config)
module.exports = ({ env }) => {
  if (env('REDIS_URL')) {
    return {
      url: env('REDIS_URL'),
    };
  }
  
  return {
    host: env('REDIS_HOST', 'localhost'),
    port: env.int('REDIS_PORT', 6379),
    password: env('REDIS_PASSWORD'),
  };
};
```

## Media Storage Configuration

### Cloudinary Setup

```javascript
// config/plugins.js
module.exports = ({ env }) => ({
  upload: {
    config: {
      provider: 'cloudinary',
      providerOptions: {
        cloud_name: env('CLOUDINARY_NAME'),
        api_key: env('CLOUDINARY_KEY'),
        api_secret: env('CLOUDINARY_SECRET'),
      },
      actionOptions: {
        upload: {},
        uploadStream: {
          folder: env('CLOUDINARY_FOLDER', 'strapi-uploads'),
        },
        delete: {},
      },
    },
  },
});
```

### AWS S3 Alternative

```javascript
// Alternative S3 configuration
module.exports = ({ env }) => ({
  upload: {
    config: {
      provider: 'aws-s3',
      providerOptions: {
        s3Options: {
          accessKeyId: env('AWS_ACCESS_KEY_ID'),
          secretAccessKey: env('AWS_ACCESS_SECRET'),
          region: env('AWS_REGION'),
          params: {
            Bucket: env('AWS_BUCKET'),
          },
        }
      },
    },
  },
});
```

## Security Configuration

### Production Security Settings

```javascript
// config/middlewares.js
module.exports = [
  'strapi::errors',
  {
    name: 'strapi::security',
    config: {
      contentSecurityPolicy: {
        useDefaults: true,
        directives: {
          'connect-src': ["'self'", 'https:'],
          'img-src': [
            "'self'",
            'data:',
            'blob:',
            'res.cloudinary.com',
          ],
          'media-src': [
            "'self'",
            'data:',
            'blob:',
            'res.cloudinary.com',
          ],
          upgradeInsecureRequests: null,
        },
      },
    },
  },
  'strapi::cors',
  'strapi::poweredBy',
  'strapi::logger',
  'strapi::query',
  'strapi::body',
  'strapi::session',
  'strapi::favicon',
  'strapi::public',
];
```

### CORS Configuration

```javascript
// config/middlewares.js (CORS section)
{
  name: 'strapi::cors',
  config: {
    origin: [
      'https://your-frontend-domain.vercel.app',
      'https://your-custom-domain.com',
      ...(process.env.NODE_ENV === 'development' ? ['http://localhost:3000'] : [])
    ],
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'HEAD', 'OPTIONS'],
    headers: ['Content-Type', 'Authorization', 'Origin', 'Accept'],
    keepHeaderOnError: true,
  },
}
```

## Performance Optimizations

### Caching Strategy

```javascript
// config/middlewares.js (add caching)
module.exports = [
  // ... other middlewares
  {
    name: 'strapi::response-time',
  },
  {
    name: 'strapi::cache',
    config: {
      type: 'redis',
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      password: process.env.REDIS_PASSWORD,
      settings: {
        ttl: 3600, // 1 hour
      },
    },
  },
];
```

### Database Indexing

```sql
-- Add database indexes for better performance
CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_published_at ON posts(published_at);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_view_count ON posts(view_count);
```

## Monitoring and Logging

### Error Tracking (Sentry)

```javascript
// config/plugins.js (add Sentry)
module.exports = ({ env }) => ({
  // ... other plugins
  sentry: {
    enabled: env('NODE_ENV') === 'production',
    config: {
      dsn: env('SENTRY_DSN'),
      sendMetadata: true,
    },
  },
});
```

### Application Monitoring

```javascript
// config/server.js
module.exports = ({ env }) => ({
  host: env('HOST', '0.0.0.0'),
  port: env.int('PORT', 1337),
  app: {
    keys: env.array('APP_KEYS'),
  },
  webhooks: {
    populateRelations: env.bool('WEBHOOKS_POPULATE_RELATIONS', false),
  },
  logger: {
    level: env('LOG_LEVEL', 'info'),
    transports: [
      {
        type: 'console',
      },
    ],
  },
});
```

### Health Check Endpoint

```javascript
// src/api/health/routes/health.js
module.exports = {
  routes: [
    {
      method: 'GET',
      path: '/health',
      handler: 'health.check',
      config: {
        auth: false,
      },
    },
  ],
};

// src/api/health/controllers/health.js
module.exports = {
  async check(ctx) {
    try {
      // Check database connection
      await strapi.db.connection.raw('SELECT 1');
      
      ctx.body = {
        status: 'ok',
        timestamp: new Date().toISOString(),
        uptime: process.uptime(),
        environment: process.env.NODE_ENV,
      };
    } catch (error) {
      ctx.status = 503;
      ctx.body = {
        status: 'error',
        message: 'Service unavailable',
      };
    }
  },
};
```

## Backup and Recovery

### Database Backup Strategy

```bash
#!/bin/bash
# backup-db.sh

DB_NAME="strapi_blog"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.sql"

# Create backup
pg_dump $DATABASE_URL > $BACKUP_FILE

# Compress backup
gzip $BACKUP_FILE

# Remove backups older than 30 days
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE.gz"
```

### Media Files Backup

```bash
#!/bin/bash
# backup-media.sh

# For Cloudinary - automated backups
# For S3 - use AWS CLI sync
aws s3 sync s3://your-bucket/uploads /local/backup/media

# Or use Cloudinary API for backup automation
```

## Deployment Checklist

### Pre-deployment

- [ ] Environment variables configured
- [ ] Database migrations tested
- [ ] SSL certificates configured
- [ ] CDN setup (if applicable)
- [ ] Monitoring tools configured
- [ ] Backup procedures tested

### Post-deployment

- [ ] Health checks passing
- [ ] SSL/HTTPS working
- [ ] Database connections stable
- [ ] Media uploads working
- [ ] Caching functioning
- [ ] Error tracking active
- [ ] Performance monitoring enabled

### Security Checklist

- [ ] Strong passwords and secrets
- [ ] CORS properly configured
- [ ] Rate limiting enabled
- [ ] Input validation active
- [ ] SQL injection protection
- [ ] XSS protection enabled
- [ ] Security headers configured

## Troubleshooting

### Common Issues

1. **Database Connection Errors**
   ```bash
   # Check connection string format
   # Verify SSL settings
   # Test with direct psql connection
   ```

2. **Media Upload Failures**
   ```bash
   # Verify Cloudinary/S3 credentials
   # Check file size limits
   # Validate CORS settings
   ```

3. **Redis Connection Issues**
   ```bash
   # Test Redis connectivity
   # Check authentication
   # Verify network access
   ```

4. **Build Failures**
   ```bash
   # Clear node_modules and reinstall
   # Check Node.js version compatibility
   # Verify environment variables
   ```

### Useful Commands

```bash
# Check application logs
railway logs

# Connect to database
railway connect postgresql

# Check environment variables
railway variables

# Restart service
railway restart

# Scale application
railway scale
```

This deployment guide provides a comprehensive approach to deploying the Next.js + Strapi application in production with proper security, monitoring, and optimization configurations.