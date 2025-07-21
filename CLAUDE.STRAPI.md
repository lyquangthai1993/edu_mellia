# CLAUDE.STRAPI.md - Strapi Backend Configuration

## Strapi v5 Installation & Configuration

```bash
# Create Strapi project with latest version
npx create-strapi@latest edu_strapi --quickstart

# Install additional plugins (if needed for v5)
npm install @strapi/plugin-users-permissions
npm install @strapi/plugin-email
npm install @strapi/plugin-slugify
```

## Content Types Structure

### Post Collection Configuration

#### Schema Definition (JSON)
```json
{
  "title": "Text (Short, Required)",
  "slug": "UID (Auto-generated from title, Required, Unique)",
  "content": "Rich Text (Required)",
  "excerpt": "Text (Long)",
  "featuredImage": "Media (Single)",
  "publishedAt": "DateTime",
  "author": "Relation (User)",
  "categories": "Relation (Many-to-Many)",
  "tags": "JSON (Array of strings)",
  "status": "Enumeration (draft, published, archived)",
  "seo": "Component (SEO)",
  "viewCount": "Number (Default: 0)"
}
```

#### Post Model Schema File

**File: `src/api/post/content-types/post/schema.json`**
```json
{
  "kind": "collectionType",
  "collectionName": "posts",
  "info": {
    "singularName": "post",
    "pluralName": "posts",
    "displayName": "Post",
    "description": "Blog posts with SEO optimization and slug-based routing"
  },
  "options": {
    "draftAndPublish": true
  },
  "pluginOptions": {
    "i18n": {
      "localized": false
    }
  },
  "attributes": {
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "minLength": 3
    },
    "slug": {
      "type": "uid",
      "targetField": "title",
      "required": true,
      "unique": true
    },
    "content": {
      "type": "richtext",
      "required": true
    },
    "excerpt": {
      "type": "text",
      "maxLength": 500
    },
    "featuredImage": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "author": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "plugin::users-permissions.user",
      "inversedBy": "posts"
    },
    "categories": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::category.category",
      "mappedBy": "posts"
    },
    "tags": {
      "type": "json",
      "default": []
    },
    "status": {
      "type": "enumeration",
      "enum": ["draft", "published", "archived"],
      "default": "draft",
      "required": true
    },
    "seo": {
      "type": "component",
      "repeatable": false,
      "component": "seo.seo-fields"
    },
    "viewCount": {
      "type": "integer",
      "default": 0,
      "min": 0
    },
    "readingTime": {
      "type": "integer",
      "default": 0,
      "min": 0
    },
    "isFeatured": {
      "type": "boolean",
      "default": false
    },
    "publishedAt": {
      "type": "datetime"
    }
  }
}
```

#### Category Model Schema File

**File: `src/api/category/content-types/category/schema.json`**
```json
{
  "kind": "collectionType",
  "collectionName": "categories",
  "info": {
    "singularName": "category",
    "pluralName": "categories",
    "displayName": "Category",
    "description": "Blog post categories"
  },
  "options": {
    "draftAndPublish": false
  },
  "pluginOptions": {},
  "attributes": {
    "name": {
      "type": "string",
      "required": true,
      "unique": true,
      "maxLength": 100
    },
    "slug": {
      "type": "uid",
      "targetField": "name",
      "required": true,
      "unique": true
    },
    "description": {
      "type": "text",
      "maxLength": 500
    },
    "color": {
      "type": "string",
      "regex": "^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$",
      "default": "#3B82F6"
    },
    "posts": {
      "type": "relation",
      "relation": "manyToMany",
      "target": "api::post.post",
      "inversedBy": "categories"
    },
    "parentCategory": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "api::category.category",
      "inversedBy": "childCategories"
    },
    "childCategories": {
      "type": "relation",
      "relation": "oneToMany",
      "target": "api::category.category",
      "mappedBy": "parentCategory"
    },
    "order": {
      "type": "integer",
      "default": 0
    },
    "isActive": {
      "type": "boolean",
      "default": true
    }
  }
}
```

### SEO Component Configuration

#### SEO Component Schema File

**File: `src/components/seo/seo-fields.json`**
```json
{
  "collectionName": "components_seo_seo_fields",
  "info": {
    "displayName": "SEO Fields",
    "description": "SEO metadata for content optimization"
  },
  "options": {},
  "attributes": {
    "metaTitle": {
      "type": "string",
      "maxLength": 60,
      "minLength": 5
    },
    "metaDescription": {
      "type": "text",
      "maxLength": 160,
      "minLength": 50
    },
    "keywords": {
      "type": "json",
      "default": []
    },
    "canonicalURL": {
      "type": "string",
      "regex": "^(https?:\\/\\/)?(www\\.)?[a-zA-Z0-9-]+(\\.[a-zA-Z]{2,})+(\\/.*)?$"
    },
    "ogTitle": {
      "type": "string",
      "maxLength": 60
    },
    "ogDescription": {
      "type": "text",
      "maxLength": 160
    },
    "ogImage": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "twitterTitle": {
      "type": "string",
      "maxLength": 70
    },
    "twitterDescription": {
      "type": "text",
      "maxLength": 200
    },
    "twitterImage": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "noIndex": {
      "type": "boolean",
      "default": false
    },
    "noFollow": {
      "type": "boolean",
      "default": false
    },
    "structuredData": {
      "type": "json",
      "default": {}
    },
    "focusKeyword": {
      "type": "string",
      "maxLength": 100
    },
    "metaRobots": {
      "type": "enumeration",
      "enum": ["index,follow", "index,nofollow", "noindex,follow", "noindex,nofollow"],
      "default": "index,follow"
    }
  }
}
```

### Global SEO Settings Collection (Single Type)

**File: `src/api/global-seo-setting/content-types/global-seo-setting/schema.json`**
```json
{
  "kind": "singleType",
  "collectionName": "global_seo_settings",
  "info": {
    "singularName": "global-seo-setting",
    "pluralName": "global-seo-settings",
    "displayName": "Global SEO Settings",
    "description": "Global SEO configuration for the website"
  },
  "options": {
    "draftAndPublish": false
  },
  "pluginOptions": {},
  "attributes": {
    "siteName": {
      "type": "string",
      "required": true,
      "maxLength": 100,
      "minLength": 3
    },
    "siteDescription": {
      "type": "text",
      "required": true,
      "maxLength": 300,
      "minLength": 50
    },
    "siteUrl": {
      "type": "string",
      "required": true,
      "regex": "^https?:\\/\\/[^\\s/$.?#].[^\\s]*$"
    },
    "defaultOgImage": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "favicon": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "appleTouchIcon": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "googleAnalyticsId": {
      "type": "string",
      "regex": "^(G-[A-Z0-9]+|UA-[0-9]+-[0-9]+)$"
    },
    "googleSearchConsoleId": {
      "type": "string"
    },
    "googleTagManagerId": {
      "type": "string",
      "regex": "^GTM-[A-Z0-9]+$"
    },
    "twitterHandle": {
      "type": "string",
      "regex": "^@[A-Za-z0-9_]+$"
    },
    "facebookAppId": {
      "type": "string",
      "regex": "^[0-9]+$"
    },
    "organizationSchema": {
      "type": "json",
      "default": {}
    },
    "websiteSchema": {
      "type": "json",
      "default": {}
    },
    "defaultMetaTitle": {
      "type": "string",
      "maxLength": 60
    },
    "defaultMetaDescription": {
      "type": "text",
      "maxLength": 160
    },
    "defaultKeywords": {
      "type": "json",
      "default": []
    },
    "socialMediaLinks": {
      "type": "json",
      "default": {}
    },
    "contactInfo": {
      "type": "json",
      "default": {}
    }
  }
}
```

### Navigation Collection Schema

**File: `src/api/navigation/content-types/navigation/schema.json`**
```json
{
  "kind": "singleType",
  "collectionName": "navigations",
  "info": {
    "singularName": "navigation",
    "pluralName": "navigations",
    "displayName": "Navigation",
    "description": "Main website navigation"
  },
  "options": {
    "draftAndPublish": false
  },
  "pluginOptions": {},
  "attributes": {
    "title": {
      "type": "string",
      "required": true,
      "default": "Main Navigation"
    },
    "items": {
      "type": "component",
      "repeatable": true,
      "component": "navigation.navigation-item"
    }
  }
}
```

### Navigation Item Component Schema

**File: `src/components/navigation/navigation-item.json`**
```json
{
  "collectionName": "components_navigation_navigation_items",
  "info": {
    "displayName": "Navigation Item",
    "description": "Single navigation menu item"
  },
  "options": {},
  "attributes": {
    "label": {
      "type": "string",
      "required": true,
      "maxLength": 50
    },
    "url": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "isExternal": {
      "type": "boolean",
      "default": false
    },
    "order": {
      "type": "integer",
      "default": 0,
      "min": 0
    },
    "isActive": {
      "type": "boolean",
      "default": true
    },
    "icon": {
      "type": "string",
      "maxLength": 50
    },
    "description": {
      "type": "text",
      "maxLength": 200
    },
    "children": {
      "type": "component",
      "repeatable": true,
      "component": "navigation.navigation-sub-item"
    }
  }
}
```

### Navigation Sub-Item Component Schema

**File: `src/components/navigation/navigation-sub-item.json`**
```json
{
  "collectionName": "components_navigation_navigation_sub_items",
  "info": {
    "displayName": "Navigation Sub Item",
    "description": "Sub-navigation menu item for dropdowns"
  },
  "options": {},
  "attributes": {
    "label": {
      "type": "string",
      "required": true,
      "maxLength": 50
    },
    "url": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "isExternal": {
      "type": "boolean",
      "default": false
    },
    "order": {
      "type": "integer",
      "default": 0,
      "min": 0
    },
    "isActive": {
      "type": "boolean",
      "default": true
    },
    "icon": {
      "type": "string",
      "maxLength": 50
    },
    "description": {
      "type": "text",
      "maxLength": 200
    }
  }
}
```

### Page Collection Schema

**File: `src/api/page/content-types/page/schema.json`**
```json
{
  "kind": "collectionType",
  "collectionName": "pages",
  "info": {
    "singularName": "page",
    "pluralName": "pages",
    "displayName": "Page",
    "description": "Static pages with templates"
  },
  "options": {
    "draftAndPublish": true
  },
  "pluginOptions": {},
  "attributes": {
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "minLength": 3
    },
    "slug": {
      "type": "uid",
      "targetField": "title",
      "required": true,
      "unique": true
    },
    "content": {
      "type": "richtext",
      "required": true
    },
    "excerpt": {
      "type": "text",
      "maxLength": 500
    },
    "template": {
      "type": "enumeration",
      "enum": ["default", "about", "contact", "services", "portfolio", "privacy", "terms"],
      "default": "default",
      "required": true
    },
    "featuredImage": {
      "type": "media",
      "multiple": false,
      "required": false,
      "allowedTypes": ["images"]
    },
    "seo": {
      "type": "component",
      "repeatable": false,
      "component": "seo.seo-fields"
    },
    "isActive": {
      "type": "boolean",
      "default": true
    },
    "showInMenu": {
      "type": "boolean",
      "default": false
    },
    "menuOrder": {
      "type": "integer",
      "default": 0
    }
  }
}
```

## Model Configuration Summary

### Content Types Created:
1. **Post** (Collection Type) - Blog posts with slug-based routing
2. **Category** (Collection Type) - Post categories with hierarchical support
3. **Page** (Collection Type) - Static pages with templates
4. **Navigation** (Single Type) - Main navigation menu
5. **Global SEO Settings** (Single Type) - Site-wide SEO configuration

### Components Created:
1. **SEO Fields** - Reusable SEO metadata component
2. **Navigation Item** - Menu items with dropdown support
3. **Navigation Sub Item** - Sub-menu items for dropdowns

### Key Features:
- **Slug-based routing** for all content types
- **SEO optimization** with dedicated fields and components
- **Media management** with image optimization
- **Hierarchical categories** with parent-child relationships
- **Flexible navigation** with dropdown support
- **Content status management** (draft, published, archived)
- **View tracking** for analytics
- **Reading time calculation** for posts

## Important Configuration Notes

### Slug Configuration
- Set slug field as **Required** and **Unique**
- Configure slug to auto-generate from title in Strapi admin
- Use slug as the primary identifier for frontend routing
- Slug should be URL-friendly (lowercase, hyphens, no special characters)
- Enable "Use as primary identifier"
- Set URL segment as: `/posts/{slug}`

### SEO Configuration
- All content types should include the SEO component for maximum control
- Global SEO settings provide fallbacks for pages without specific SEO data
- Structured data can be customized per content type or use global defaults
- SEO fields auto-populate from content but can be overridden by editors

## API Configuration

Configure Strapi API permissions in Settings > Users & Permissions Plugin:
- Public: Read access to Posts, Pages, Navigation
- Authenticated: Full CRUD for user's own content

## Strapi v5 API Routes Configuration

### File: `src/api/post/routes/post.js`
```javascript
module.exports = {
  routes: [
    {
      method: 'GET',
      path: '/posts',
      handler: 'post.find',
      config: {
        auth: false,
      },
    },
    {
      method: 'GET',
      path: '/posts/slug/:slug',
      handler: 'post.findBySlug',
      config: {
        auth: false,
      },
    },
    {
      method: 'GET',
      path: '/posts/:documentId',
      handler: 'post.findOne',
      config: {
        auth: false,
      },
    },
  ],
}
```

### File: `src/api/post/controllers/post.js`
```javascript
const { createCoreController } = require('@strapi/strapi').factories

module.exports = createCoreController('api::post.post', ({ strapi }) => ({
  // Custom controller to find post by slug
  async findBySlug(ctx) {
    const { slug } = ctx.params
    const { populate } = ctx.query

    const entity = await strapi.db.query('api::post.post').findOne({
      where: { 
        slug,
        publishedAt: { $notNull: true }
      },
      populate: populate || ['featuredImage', 'author', 'categories']
    })

    if (!entity) {
      return ctx.notFound('Post not found')
    }

    // Increment view count
    await strapi.db.query('api::post.post').update({
      where: { id: entity.id },
      data: { viewCount: entity.viewCount + 1 }
    })

    const sanitizedEntity = await this.sanitizeOutput(entity, ctx)
    return this.transformResponse(sanitizedEntity)
  },

  // Override default find to only show published posts
  async find(ctx) {
    ctx.query.filters = {
      ...ctx.query.filters,
      publishedAt: { $notNull: true }
    }

    const { data, meta } = await super.find(ctx)
    return { data, meta }
  },

  // Custom method to get all slugs for static generation
  async getSlugs(ctx) {
    const entities = await strapi.db.query('api::post.post').findMany({
      select: ['slug'],
      where: { publishedAt: { $notNull: true } }
    })

    return entities.map(entity => entity.slug)
  }
}))
```

## Strapi v5 Lifecycle Hooks for Slug Management

### File: `src/api/post/content-types/post/lifecycles.js`
```javascript
module.exports = {
  // Ensure slug is generated before creation
  async beforeCreate(event) {
    const { data } = event.params
    
    if (data.title && !data.slug) {
      data.slug = await generateUniqueSlug(data.title)
    }
  },

  // Ensure slug is updated when title changes
  async beforeUpdate(event) {
    const { data, where } = event.params
    
    if (data.title) {
      const existingPost = await strapi.db.query('api::post.post').findOne({
        where
      })
      
      // Only update slug if title changed
      if (existingPost && existingPost.title !== data.title) {
        data.slug = await generateUniqueSlug(data.title, existingPost.id)
      }
    }
  },

  // Initialize view count for new posts
  async beforeCreate(event) {
    const { data } = event.params
    if (data.viewCount === undefined) {
      data.viewCount = 0
    }
  }
}

// Helper function to generate unique slug
async function generateUniqueSlug(title, excludeId = null) {
  const baseSlug = title
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
    .trim('-')

  let slug = baseSlug
  let counter = 1

  while (true) {
    const existingPost = await strapi.db.query('api::post.post').findOne({
      where: { 
        slug,
        ...(excludeId && { id: { $ne: excludeId } })
      }
    })

    if (!existingPost) {
      break
    }

    slug = `${baseSlug}-${counter}`
    counter++
  }

  return slug
}
```

## Environment Variables

### Backend Strapi v5 (.env)
```env
HOST=0.0.0.0
PORT=1337

# Secrets
APP_KEYS=your-app-keys-here
API_TOKEN_SALT=your-api-token-salt
ADMIN_JWT_SECRET=your-admin-jwt-secret
TRANSFER_TOKEN_SALT=your-transfer-token-salt
JWT_SECRET=your-jwt-secret

# Database (PostgreSQL recommended for production)
DATABASE_CLIENT=postgres
DATABASE_NAME=strapi_blog
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=root
DATABASE_PASSWORD=your_db_password
DATABASE_SSL=false

# Redis Configuration
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password

# Media Storage (Cloudinary)
CLOUDINARY_NAME=your_cloudinary_name
CLOUDINARY_KEY=your_cloudinary_key
CLOUDINARY_SECRET=your_cloudinary_secret

# Email Configuration (Optional)
EMAIL_PROVIDER=sendgrid
EMAIL_PROVIDER_OPTIONS_API_KEY=your_sendgrid_api_key
EMAIL_DEFAULT_FROM=noreply@yourdomain.com
EMAIL_DEFAULT_REPLY_TO=support@yourdomain.com
```

## Production Configuration

### File: `config/database.js`
```javascript
module.exports = ({ env }) => ({
  connection: {
    client: 'postgres',
    connection: {
      host: env('DATABASE_HOST', 'localhost'),
      port: env.int('DATABASE_PORT', 5432),
      database: env('DATABASE_NAME', 'strapi'),
      user: env('DATABASE_USERNAME', 'strapi'),
      password: env('DATABASE_PASSWORD', 'strapi'),
      ssl: env.bool('DATABASE_SSL', false) && {
        rejectUnauthorized: env.bool('DATABASE_SSL_REJECT_UNAUTHORIZED', true),
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
})
```

### File: `config/plugins.js`
```javascript
module.exports = ({ env }) => ({
  // Upload plugin configuration for Cloudinary
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
        uploadStream: {},
        delete: {},
      },
    },
  },
  
  // Email plugin configuration
  email: {
    config: {
      provider: 'sendgrid',
      providerOptions: {
        apiKey: env('EMAIL_PROVIDER_OPTIONS_API_KEY'),
      },
      settings: {
        defaultFrom: env('EMAIL_DEFAULT_FROM'),
        defaultReplyTo: env('EMAIL_DEFAULT_REPLY_TO'),
      },
    },
  },
  
  // Enable GraphQL if needed
  graphql: {
    enabled: true,
    config: {
      endpoint: '/graphql',
      shadowCRUD: true,
      playgroundAlways: false,
      depthLimit: 7,
      amountLimit: 100,
      introspection: true,
      apolloServer: {
        tracing: false,
      },
    },
  },
})
```