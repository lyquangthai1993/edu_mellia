# CLAUDE.md - Next.js + Strapi Website Project

## Project Overview

This project creates a modern web application using Next.js as the frontend framework with Strapi as a headless CMS backend. The architecture includes authentication, Redis caching for performance optimization, Static Site Generation (SSG) for blog post pages, and comprehensive SEO optimization.

**SEO Requirements**: This website must be built with SEO as a top priority. All pages must include proper meta tags, Open Graph data, structured data (JSON-LD), sitemap generation, and optimized performance for search engine crawling. See [CLAUDE.SEO.md](./CLAUDE.SEO.md) for detailed SEO implementation guidelines.

## Documentation Structure

This project documentation is split into focused files for better organization:

- **[CLAUDE.md](./CLAUDE.md)** - This overview file with project structure and timeline
- **[CLAUDE.STRAPI.md](./CLAUDE.STRAPI.md)** - Complete Strapi backend configuration
- **[CLAUDE.NEXTJS.md](./CLAUDE.NEXTJS.md)** - Next.js frontend implementation
- **[CLAUDE.DEPLOYMENT.md](./CLAUDE.DEPLOYMENT.md)** - Production deployment and configuration
- **[CLAUDE.SEO.md](./CLAUDE.SEO.md)** - SEO implementation guidelines

## Project Structure

```
edu_mellia/
├── edu_nextjs/          # Next.js frontend application
│   ├── app/
│   ├── components/
│   ├── lib/
│   └── ...
├── edu_strapi/          # Strapi backend CMS
│   ├── src/
│   ├── config/
│   └── ...
├── CLAUDE.md           # Project overview (this file)
├── CLAUDE.STRAPI.md    # Strapi backend documentation
├── CLAUDE.NEXTJS.md    # Next.js frontend documentation
├── CLAUDE.DEPLOYMENT.md # Deployment and production config
└── CLAUDE.SEO.md       # SEO implementation guidelines
```

## Tech Stack

- **Frontend**: Next.js 14+ with App Router
- **Backend CMS**: Strapi v5 (latest)
- **Authentication**: NextAuth.js with Strapi integration
- **Caching**: Redis
- **Database**: PostgreSQL (recommended for Strapi)
- **Deployment**: Vercel (frontend) + Railway/DigitalOcean (backend)

## Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Next.js App   │───▶│   Strapi CMS    │───▶│   PostgreSQL    │
│  (Frontend)     │    │   (Backend)     │    │   (Database)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│     Redis       │    │  Media Storage  │
│   (Caching)     │    │ (Cloudinary/S3) │
└─────────────────┘    └─────────────────┘
```

## Quick Start Guide

### 1. Backend Setup (Strapi)
See [CLAUDE.STRAPI.md](./CLAUDE.STRAPI.md) for complete setup instructions:

```bash
# Create Strapi project
npx create-strapi@latest edu_strapi --quickstart

# Install additional plugins
npm install @strapi/plugin-users-permissions
npm install @strapi/plugin-email
```

### 2. Frontend Setup (Next.js)
See [CLAUDE.NEXTJS.md](./CLAUDE.NEXTJS.md) for complete setup instructions:

```bash
# Create Next.js project
npx create-next-app@latest edu_nextjs --typescript --tailwind --eslint --app

# Install dependencies
npm install next-auth ioredis axios react-markdown
```

### 3. Deployment
See [CLAUDE.DEPLOYMENT.md](./CLAUDE.DEPLOYMENT.md) for production deployment:

- **Frontend**: Deploy to Vercel
- **Backend**: Deploy to Railway or DigitalOcean
- **Database**: PostgreSQL
- **Caching**: Redis instance

## Development Timeline

**Week 1-2**: Backend Setup (Strapi v5)
- Strapi v5 installation and configuration
- Content types creation with slug-based routing
- API permissions and custom controllers setup
- Database and Redis setup
- Slug validation and lifecycle hooks

**Week 3-4**: Frontend Foundation
- Next.js 14+ project setup with App Router
- Authentication implementation with NextAuth.js
- Basic routing and components
- Slug-based navigation implementation

**Week 5-6**: Integration & Caching
- Strapi v5 API integration with slug routing
- Redis caching implementation
- SSG setup for post detail pages
- Image optimization and CDN setup

**Week 7-8**: Polish & Deployment
- Navigation and UI refinement
- SEO optimization with slug-based URLs
- Testing and performance optimization
- Production deployment with slug validation

## Key Features

### Slug-Based Architecture
- Clean URLs: `/posts/my-awesome-post-title`
- No numeric IDs in URLs for better SEO
- Automatic slug generation from titles
- Slug validation and uniqueness enforcement

### Performance Optimizations
- Redis caching for API responses
- Static Site Generation (SSG) for blog posts
- Image optimization with Next.js
- CDN integration for media files

### SEO Features
- Dynamic meta tags from Strapi
- Open Graph and Twitter Card support
- Schema.org structured data
- Automatic sitemap generation
- Robot.txt configuration

### Security & Authentication
- NextAuth.js integration with Strapi
- Protected routes and middleware
- CORS configuration
- Input validation and sanitization

## Environment Setup

### Development Environment Variables

**Frontend (.env.local)**
```env
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here
NEXT_PUBLIC_STRAPI_API_URL=http://localhost:1337/api
REDIS_HOST=localhost
REDIS_PORT=6379
```

**Backend (.env)**
```env
HOST=0.0.0.0
PORT=1337
APP_KEYS=your-app-keys-here
DATABASE_CLIENT=postgres
DATABASE_URL=postgresql://user:password@localhost:5432/strapi_blog
```

## Getting Help

For specific implementation details, refer to the dedicated documentation files:

- **Strapi Backend**: [CLAUDE.STRAPI.md](./CLAUDE.STRAPI.md)
- **Next.js Frontend**: [CLAUDE.NEXTJS.md](./CLAUDE.NEXTJS.md)
- **Deployment Guide**: [CLAUDE.DEPLOYMENT.md](./CLAUDE.DEPLOYMENT.md)
- **SEO Guidelines**: [CLAUDE.SEO.md](./CLAUDE.SEO.md)

## Performance & Monitoring

1. **Image Optimization**: Use Next.js Image component with Strapi media
2. **Caching Strategy**: Implement multi-level caching (Redis + CDN)
3. **Bundle Analysis**: Regular bundle size monitoring
4. **Database Indexing**: Optimize Strapi database queries
5. **Error Tracking**: Integrate Sentry for error monitoring
6. **Performance Monitoring**: Use Vercel Analytics
7. **SEO Tracking**: Google Search Console integration

## Security Considerations

1. **API Rate Limiting**: Implement rate limiting in Strapi
2. **CORS Configuration**: Proper CORS setup for production
3. **Input Validation**: Sanitize user inputs
4. **JWT Security**: Secure token storage and rotation
5. **HTTPS**: Force HTTPS in production

## Strapi Configuration Memories

- Use Strapi v5 format data
- Use typescript, please