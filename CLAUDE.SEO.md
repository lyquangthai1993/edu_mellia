# CLAUDE.SEO.md - SEO Implementation Guidelines

## Overview

This document provides comprehensive SEO guidelines for the Next.js + Strapi website. All implementations must prioritize search engine optimization to ensure maximum visibility and organic traffic. SEO configuration is managed through Strapi CMS for dynamic control and easy content editor management.

## Strapi SEO Configuration

### 1. Strapi Content Types for SEO

#### SEO Component (Reusable)
Create a reusable SEO component in Strapi that can be attached to all content types:

```json
{
  "metaTitle": "Text (Short, 60 chars max)",
  "metaDescription": "Text (Long, 160 chars max)",
  "keywords": "JSON (Array of strings)",
  "canonicalURL": "Text (Short)",
  "ogTitle": "Text (Short)",
  "ogDescription": "Text (Long)",
  "ogImage": "Media (Single)",
  "twitterTitle": "Text (Short)",
  "twitterDescription": "Text (Long)",
  "twitterImage": "Media (Single)",
  "noIndex": "Boolean (Default: false)",
  "noFollow": "Boolean (Default: false)",
  "structuredData": "JSON (Custom schema markup)"
}
```

#### Global SEO Settings (Single Type)
Create a global SEO settings single type for site-wide defaults:

```json
{
  "siteName": "Text (Short, Required)",
  "siteDescription": "Text (Long, Required)",
  "defaultOgImage": "Media (Single)",
  "favicon": "Media (Single)",
  "appleTouchIcon": "Media (Single)",
  "googleAnalyticsId": "Text (Short)",
  "googleSearchConsoleId": "Text (Short)",
  "twitterHandle": "Text (Short)",
  "facebookAppId": "Text (Short)",
  "organizationSchema": "JSON (Organization structured data)",
  "websiteSchema": "JSON (Website structured data)",
  "defaultKeywords": "JSON (Array of strings)",
  "robots": "Text (Long, robots.txt content)"
}
```

### 2. Strapi API Functions for SEO

```typescript
// lib/strapi-seo.ts
import { strapiApi } from './strapi'

export interface SeoData {
  metaTitle?: string
  metaDescription?: string
  keywords?: string[]
  canonicalURL?: string
  ogTitle?: string
  ogDescription?: string
  ogImage?: any
  twitterTitle?: string
  twitterDescription?: string
  twitterImage?: any
  noIndex?: boolean
  noFollow?: boolean
  structuredData?: any
}

export interface GlobalSeoSettings {
  siteName: string
  siteDescription: string
  defaultOgImage?: any
  favicon?: any
  appleTouchIcon?: any
  googleAnalyticsId?: string
  googleSearchConsoleId?: string
  twitterHandle?: string
  facebookAppId?: string
  organizationSchema?: any
  websiteSchema?: any
  defaultKeywords?: string[]
  robots?: string
}

// Fetch global SEO settings from Strapi
export async function getGlobalSeoSettings(): Promise<GlobalSeoSettings | null> {
  try {
    const response = await strapiApi.get('/global-seo-setting', {
      params: {
        populate: ['defaultOgImage', 'favicon', 'appleTouchIcon']
      }
    })
    return response.data.data
  } catch (error) {
    console.error('Error fetching global SEO settings:', error)
    return null
  }
}

// Build SEO metadata from Strapi content
export function buildSeoMetadata(
  content: any, 
  globalSeo: GlobalSeoSettings | null, 
  path: string,
  contentType: 'article' | 'website' = 'website'
) {
  const seo: SeoData = content.seo || {}
  
  // Build fallback description from content
  const fallbackDescription = content.excerpt || 
    (content.content && typeof content.content === 'string' 
      ? content.content.substring(0, 160) + '...' 
      : globalSeo?.siteDescription || '')

  // Build image array with fallbacks
  const buildImageArray = () => {
    if (seo.ogImage) return [seo.ogImage.url]
    if (seo.twitterImage) return [seo.twitterImage.url]
    if (content.featuredImage) return [content.featuredImage.url]
    if (globalSeo?.defaultOgImage) return [globalSeo.defaultOgImage.url]
    return []
  }

  return {
    title: seo.metaTitle || content.title || globalSeo?.siteName || 'Untitled',
    description: seo.metaDescription || fallbackDescription,
    keywords: seo.keywords || content.tags || globalSeo?.defaultKeywords || [],
    canonical: seo.canonicalURL || path,
    openGraph: {
      title: seo.ogTitle || seo.metaTitle || content.title || globalSeo?.siteName,
      description: seo.ogDescription || seo.metaDescription || fallbackDescription,
      images: buildImageArray(),
      type: contentType,
      siteName: globalSeo?.siteName,
      locale: 'en_US',
    },
    twitter: {
      title: seo.twitterTitle || seo.ogTitle || seo.metaTitle || content.title,
      description: seo.twitterDescription || seo.ogDescription || seo.metaDescription || fallbackDescription,
      images: seo.twitterImage ? [seo.twitterImage.url] : buildImageArray(),
      creator: globalSeo?.twitterHandle,
    },
    robots: {
      index: !seo.noIndex,
      follow: !seo.noFollow,
    },
    structuredData: seo.structuredData || null
  }
}

// Generate Next.js metadata from Strapi content
export async function generateStrapiMetadata(
  content: any,
  path: string,
  contentType: 'article' | 'website' = 'website'
) {
  const globalSeo = await getGlobalSeoSettings()
  const seoData = buildSeoMetadata(content, globalSeo, path, contentType)

  const metadata: any = {
    title: seoData.title,
    description: seoData.description,
    keywords: seoData.keywords.join(', '),
    openGraph: {
      ...seoData.openGraph,
      url: path,
    },
    twitter: {
      card: 'summary_large_image',
      ...seoData.twitter,
    },
    alternates: {
      canonical: seoData.canonical,
    },
    robots: seoData.robots,
  }

  // Add article-specific metadata
  if (contentType === 'article' && content.publishedAt) {
    metadata.openGraph.publishedTime = content.publishedAt
    metadata.openGraph.modifiedTime = content.updatedAt
    metadata.openGraph.authors = content.author ? [content.author.username] : []
    metadata.openGraph.section = content.categories?.[0]?.name
    metadata.openGraph.tags = content.tags
  }

  // Add structured data if available
  if (seoData.structuredData) {
    metadata.other = {
      'structured-data': JSON.stringify(seoData.structuredData)
    }
  }

  return metadata
}
```

## Core SEO Requirements

### 3. Dynamic Meta Tags Implementation

All pages must include comprehensive meta tags managed through Strapi:

```typescript
// app/layout.tsx - Global metadata from Strapi
import { getGlobalSeoSettings } from '@/lib/strapi-seo'

export async function generateMetadata(): Promise<Metadata> {
  const globalSeo = await getGlobalSeoSettings()

  return {
    metadataBase: new URL(process.env.NEXT_PUBLIC_SITE_URL || 'https://yourdomain.com'),
    title: {
      default: globalSeo?.siteName || 'Your Site Name',
      template: `%s | ${globalSeo?.siteName || 'Your Site Name'}`
    },
    description: globalSeo?.siteDescription || 'Your site description',
    keywords: globalSeo?.defaultKeywords || [],
    authors: [{ name: globalSeo?.siteName || 'Your Name' }],
    creator: globalSeo?.siteName || 'Your Name',
    publisher: globalSeo?.siteName || 'Your Site Name',
    formatDetection: {
      email: false,
      address: false,
      telephone: false,
    },
    openGraph: {
      type: 'website',
      locale: 'en_US',
      url: process.env.NEXT_PUBLIC_SITE_URL || 'https://yourdomain.com',
      title: globalSeo?.siteName || 'Your Site Name',
      description: globalSeo?.siteDescription || 'Your site description',
      siteName: globalSeo?.siteName || 'Your Site Name',
      images: globalSeo?.defaultOgImage ? [{
        url: globalSeo.defaultOgImage.url,
        width: 1200,
        height: 630,
        alt: globalSeo.siteName,
      }] : [],
    },
    twitter: {
      card: 'summary_large_image',
      title: globalSeo?.siteName || 'Your Site Name',
      description: globalSeo?.siteDescription || 'Your site description',
      creator: globalSeo?.twitterHandle || '@yourusername',
      images: globalSeo?.defaultOgImage ? [globalSeo.defaultOgImage.url] : [],
    },
    robots: {
      index: true,
      follow: true,
      googleBot: {
        index: true,
        follow: true,
        'max-video-preview': -1,
        'max-image-preview': 'large',
        'max-snippet': -1,
      },
    },
    verification: {
      google: globalSeo?.googleSearchConsoleId,
    },
    icons: {
      icon: globalSeo?.favicon?.url,
      apple: globalSeo?.appleTouchIcon?.url,
    },
  }
}
```

### 4. Dynamic Page Metadata from Strapi

Each page must generate dynamic metadata based on Strapi content and SEO configuration:

```typescript
// app/posts/[slug]/page.tsx
import { generateStrapiMetadata } from '@/lib/strapi-seo'
import { getPostBySlug } from '@/lib/strapi'

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPostBySlug(params.slug)
  
  if (!post) {
    return {
      title: 'Post Not Found',
      description: 'The requested post could not be found.',
      robots: { index: false, follow: false }
    }
  }

  return await generateStrapiMetadata(post, `/posts/${post.slug}`, 'article')
}

// Usage example for pages
// app/[slug]/page.tsx
export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const page = await getPageBySlug(params.slug)
  
  if (!page) {
    return {
      title: 'Page Not Found',
      robots: { index: false, follow: false }
    }
  }

  return await generateStrapiMetadata(page, `/${page.slug}`, 'website')
}
```

### 5. Strapi Admin SEO Interface

Configure Strapi admin interface for easy SEO management:

#### SEO Component Configuration in Strapi Admin

1. **Create SEO Component**: Go to Content-Type Builder > Components > Create new component "SEO"
2. **Add Fields**:
   - `metaTitle`: Text (Short text, maxLength: 60)
   - `metaDescription`: Text (Long text, maxLength: 160)
   - `keywords`: JSON field for array of strings
   - `canonicalURL`: Text (Short text)
   - `ogTitle`: Text (Short text, maxLength: 60)
   - `ogDescription`: Text (Long text, maxLength: 160)
   - `ogImage`: Media (Single image)
   - `twitterTitle`: Text (Short text, maxLength: 60)
   - `twitterDescription`: Text (Long text, maxLength: 160)
   - `twitterImage`: Media (Single image)
   - `noIndex`: Boolean (default: false)
   - `noFollow`: Boolean (default: false)
   - `structuredData`: JSON field

#### Global SEO Settings Configuration

1. **Create Single Type**: "Global SEO Settings"
2. **Add Fields**:
   - `siteName`: Text (required)
   - `siteDescription`: Long text (required)
   - `defaultOgImage`: Media (Single image)
   - `favicon`: Media (Single image, .ico format)
   - `appleTouchIcon`: Media (Single image, .png format)
   - `googleAnalyticsId`: Text (Short text)
   - `googleSearchConsoleId`: Text (Short text)
   - `twitterHandle`: Text (Short text, starts with @)
   - `facebookAppId`: Text (Short text)
   - `defaultKeywords`: JSON field
   - `organizationSchema`: JSON field
   - `websiteSchema`: JSON field
   - `robots`: Long text field for robots.txt content
```

### 6. Strapi-Based Structured Data (JSON-LD)

Implement Schema.org structured data managed through Strapi:

#### Article Schema (Blog Posts) from Strapi

```typescript
// lib/strapi-schemas.ts
import { getGlobalSeoSettings } from './strapi-seo'

export async function generateArticleSchema(post: any, globalSeo?: any) {
  if (!globalSeo) {
    globalSeo = await getGlobalSeoSettings()
  }

  // Check if custom structured data exists in Strapi SEO component
  if (post.seo?.structuredData) {
    return post.seo.structuredData
  }

  // Generate default article schema
  const baseUrl = process.env.NEXT_PUBLIC_SITE_URL || 'https://yourdomain.com'
  
  return {
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "headline": post.seo?.metaTitle || post.title,
    "description": post.seo?.metaDescription || post.excerpt || post.content.substring(0, 160),
    "image": post.seo?.ogImage?.url || post.featuredImage?.url ? {
      "@type": "ImageObject",
      "url": post.seo?.ogImage?.url || post.featuredImage.url,
      "width": 1200,
      "height": 630,
      "alt": post.seo?.ogImage?.alternativeText || post.featuredImage?.alternativeText || post.title
    } : globalSeo?.defaultOgImage ? {
      "@type": "ImageObject",
      "url": globalSeo.defaultOgImage.url,
      "width": 1200,
      "height": 630,
      "alt": globalSeo.siteName
    } : undefined,
    "datePublished": post.publishedAt,
    "dateModified": post.updatedAt,
    "author": {
      "@type": "Person",
      "name": post.author?.username || "Anonymous",
      "url": post.author ? `${baseUrl}/authors/${post.author.username}` : undefined
    },
    "publisher": globalSeo?.organizationSchema || {
      "@type": "Organization",
      "name": globalSeo?.siteName || "Your Site Name",
      "logo": {
        "@type": "ImageObject",
        "url": globalSeo?.defaultOgImage?.url || `${baseUrl}/logo.png`,
        "width": 180,
        "height": 60
      }
    },
    "mainEntityOfPage": {
      "@type": "WebPage",
      "@id": `${baseUrl}/posts/${post.slug}`
    },
    "url": `${baseUrl}/posts/${post.slug}`,
    "keywords": post.seo?.keywords?.join(', ') || post.tags?.join(', '),
    "articleSection": post.categories?.map((cat: any) => cat.name),
    "wordCount": post.content ? post.content.split(' ').length : 0,
    "commentCount": 0, // Implement if you have comments
    "isAccessibleForFree": true,
    "inLanguage": "en-US"
  }
}
```

#### Organization Schema (Global)

```typescript
// lib/schemas.ts
export const organizationSchema = {
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "Your Site Name",
  "url": "https://yourdomain.com",
  "logo": "https://yourdomain.com/logo.png",
  "description": "Your organization description",
  "address": {
    "@type": "PostalAddress",
    "streetAddress": "Your Street Address",
    "addressLocality": "Your City",
    "addressRegion": "Your State",
    "postalCode": "Your ZIP",
    "addressCountry": "Your Country"
  },
  "contactPoint": {
    "@type": "ContactPoint",
    "telephone": "+1-xxx-xxx-xxxx",
    "contactType": "customer service",
    "email": "contact@yourdomain.com"
  },
  "sameAs": [
    "https://www.facebook.com/yourpage",
    "https://www.twitter.com/yourusername",
    "https://www.linkedin.com/company/yourcompany",
    "https://www.instagram.com/yourusername"
  ]
}

export const websiteSchema = {
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "Your Site Name",
  "url": "https://yourdomain.com",
  "description": "Your site description",
  "publisher": organizationSchema,
  "potentialAction": {
    "@type": "SearchAction",
    "target": {
      "@type": "EntryPoint",
      "urlTemplate": "https://yourdomain.com/search?q={search_term_string}"
    },
    "query-input": "required name=search_term_string"
  }
}
```

### 4. Sitemap Generation

Create dynamic sitemaps for better crawlability:

```typescript
// app/sitemap.ts
import { getAllPostSlugs, getAllPageSlugs } from '@/lib/strapi'

export default async function sitemap() {
  const baseUrl = 'https://yourdomain.com'
  
  // Get all post slugs
  const postSlugs = await getAllPostSlugs()
  const pageSlugs = await getAllPageSlugs()
  
  // Static pages
  const staticPages = [
    {
      url: baseUrl,
      lastModified: new Date(),
      changeFrequency: 'daily' as const,
      priority: 1,
    },
    {
      url: `${baseUrl}/posts`,
      lastModified: new Date(),
      changeFrequency: 'daily' as const,
      priority: 0.8,
    },
    {
      url: `${baseUrl}/about`,
      lastModified: new Date(),
      changeFrequency: 'weekly' as const,
      priority: 0.7,
    },
    {
      url: `${baseUrl}/contact`,
      lastModified: new Date(),
      changeFrequency: 'monthly' as const,
      priority: 0.6,
    },
  ]

  // Dynamic post pages
  const postPages = postSlugs.map((slug) => ({
    url: `${baseUrl}/posts/${slug}`,
    lastModified: new Date(),
    changeFrequency: 'weekly' as const,
    priority: 0.9,
  }))

  // Dynamic pages
  const dynamicPages = pageSlugs.map((slug) => ({
    url: `${baseUrl}/${slug}`,
    lastModified: new Date(),
    changeFrequency: 'monthly' as const,
    priority: 0.7,
  }))

  return [...staticPages, ...postPages, ...dynamicPages]
}
```

### 5. Robots.txt

```typescript
// app/robots.ts
export default function robots() {
  return {
    rules: {
      userAgent: '*',
      allow: '/',
      disallow: ['/api/', '/admin/', '/dashboard/'],
    },
    sitemap: 'https://yourdomain.com/sitemap.xml',
  }
}
```

### 6. Performance Optimization for SEO

#### Core Web Vitals Implementation

```typescript
// lib/web-vitals.ts
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals'

function sendToAnalytics(metric: any) {
  // Send to your analytics service
  console.log(metric)
}

export function reportWebVitals() {
  getCLS(sendToAnalytics)
  getFID(sendToAnalytics)
  getFCP(sendToAnalytics)
  getLCP(sendToAnalytics)
  getTTFB(sendToAnalytics)
}
```

#### Image Optimization

```typescript
// components/OptimizedImage.tsx
import Image from 'next/image'

interface OptimizedImageProps {
  src: string
  alt: string
  width: number
  height: number
  priority?: boolean
  className?: string
}

export default function OptimizedImage({ 
  src, 
  alt, 
  width, 
  height, 
  priority = false,
  className 
}: OptimizedImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      priority={priority}
      className={className}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAYEBQYFBAYGBQYHBwYIChAKCgkJChQODwwQFxQYGBcUFhYaHSUfGhsjHBYWICwgIyYnKSopGR8tMC0oMCUoKSj/2wBDAQcHBwoIChMKChMoGhYaKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCgoKCj/wAARCAAIAAoDASIAAhEBAxEB/8QAFQABAQAAAAAAAAAAAAAAAAAAAAv/xAAhEAACAQMDBQAAAAAAAAAAAAABAgMABAUGIWGRkqGx0f/EABUBAQEAAAAAAAAAAAAAAAAAAAMF/8QAGhEAAgIDAAAAAAAAAAAAAAAAAAECEgMRkf/aAAwDAQACEQMRAD8AltJagyeH0AthI5xdrLcNM91BF5pX2HaH9bcfaSXWGaRmknyJckliyjqTzSlT54b6bk+h0R//2Q=="
      quality={85}
      loading={priority ? 'eager' : 'lazy'}
      style={{
        objectFit: 'cover',
      }}
    />
  )
}
```

### 7. URL Structure and Routing

Implement SEO-friendly URL patterns:

- Homepage: `/`
- Blog listing: `/posts`
- Blog post: `/posts/[slug]`
- Static pages: `/[slug]`
- Category pages: `/category/[slug]`
- Tag pages: `/tag/[slug]`
- Author pages: `/author/[slug]`

### 8. Breadcrumb Implementation

```typescript
// components/Breadcrumbs.tsx
import Link from 'next/link'

interface BreadcrumbItem {
  label: string
  href: string
}

interface BreadcrumbsProps {
  items: BreadcrumbItem[]
}

export default function Breadcrumbs({ items }: BreadcrumbsProps) {
  const breadcrumbSchema = {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    "itemListElement": items.map((item, index) => ({
      "@type": "ListItem",
      "position": index + 1,
      "name": item.label,
      "item": `https://yourdomain.com${item.href}`
    }))
  }

  return (
    <>
      <nav aria-label="Breadcrumb" className="mb-4">
        <ol className="flex items-center space-x-2 text-sm text-gray-600">
          {items.map((item, index) => (
            <li key={item.href} className="flex items-center">
              {index > 0 && <span className="mx-2">/</span>}
              {index === items.length - 1 ? (
                <span className="text-gray-900 font-medium">{item.label}</span>
              ) : (
                <Link 
                  href={item.href}
                  className="hover:text-blue-600 transition-colors"
                >
                  {item.label}
                </Link>
              )}
            </li>
          ))}
        </ol>
      </nav>
      
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{
          __html: JSON.stringify(breadcrumbSchema)
        }}
      />
    </>
  )
}
```

### 9. Content Optimization

#### Reading Time Calculation

```typescript
// lib/reading-time.ts
export function calculateReadingTime(content: string): number {
  const wordsPerMinute = 200
  const wordCount = content.split(/\s+/).length
  return Math.ceil(wordCount / wordsPerMinute)
}
```

#### Content Length Guidelines

- Blog post titles: 50-60 characters
- Meta descriptions: 150-160 characters
- Blog post content: Minimum 1000 words for better ranking
- Image alt text: Descriptive and keyword-rich
- Heading structure: Proper H1, H2, H3 hierarchy

### 10. Social Media Integration

```typescript
// components/SocialShare.tsx
interface SocialShareProps {
  url: string
  title: string
  description: string
}

export default function SocialShare({ url, title, description }: SocialShareProps) {
  const shareUrls = {
    twitter: `https://twitter.com/intent/tweet?url=${encodeURIComponent(url)}&text=${encodeURIComponent(title)}`,
    facebook: `https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(url)}`,
    linkedin: `https://www.linkedin.com/sharing/share-offsite/?url=${encodeURIComponent(url)}`,
    whatsapp: `https://wa.me/?text=${encodeURIComponent(`${title} ${url}`)}`,
  }

  return (
    <div className="flex space-x-4">
      <a href={shareUrls.twitter} target="_blank" rel="noopener noreferrer">
        Share on Twitter
      </a>
      <a href={shareUrls.facebook} target="_blank" rel="noopener noreferrer">
        Share on Facebook
      </a>
      <a href={shareUrls.linkedin} target="_blank" rel="noopener noreferrer">
        Share on LinkedIn
      </a>
    </div>
  )
}
```

### 11. Analytics and Monitoring

#### Google Analytics 4 Integration

```typescript
// lib/gtag.ts
export const GA_TRACKING_ID = process.env.NEXT_PUBLIC_GA_ID

export const pageview = (url: string) => {
  window.gtag('config', GA_TRACKING_ID, {
    page_path: url,
  })
}

export const event = ({ action, category, label, value }: {
  action: string
  category: string
  label?: string
  value?: number
}) => {
  window.gtag('event', action, {
    event_category: category,
    event_label: label,
    value: value,
  })
}
```

#### Search Console Setup

1. Verify site ownership in Google Search Console
2. Submit sitemap
3. Monitor Core Web Vitals
4. Track search performance
5. Monitor mobile usability

### 12. Content Guidelines

#### SEO Content Checklist

- [ ] Target keyword in title (near the beginning)
- [ ] Target keyword in first paragraph
- [ ] Target keyword in at least one H2 heading
- [ ] Related keywords throughout content
- [ ] Internal links to related content
- [ ] External links to authoritative sources
- [ ] Optimized images with alt text
- [ ] Proper heading hierarchy (H1, H2, H3)
- [ ] Meta description includes target keyword
- [ ] URL slug includes target keyword

#### Content Structure

```markdown
# H1 Title (Target Keyword)

Introduction paragraph with target keyword in first 100 words.

## H2 Main Section (Related Keywords)

Content with internal links and keyword variations.

### H3 Subsection

Detailed content with supporting keywords.

## H2 Another Section

More valuable content with external links to authoritative sources.

## Conclusion

Summary with call-to-action and target keyword.
```

### 13. Technical SEO Requirements

#### Canonical URLs

```typescript
// Ensure canonical URLs are set for all pages
export const metadata = {
  alternates: {
    canonical: `/posts/${slug}`,
  },
}
```

#### Pagination

```typescript
// For paginated content
export const metadata = {
  alternates: {
    canonical: `/posts?page=${currentPage}`,
    ...(currentPage > 1 && { prev: `/posts?page=${currentPage - 1}` }),
    ...(currentPage < totalPages && { next: `/posts?page=${currentPage + 1}` }),
  },
}
```

#### Language and Locale

```typescript
// Set proper language attributes
export const metadata = {
  openGraph: {
    locale: 'en_US',
  },
}
```

### 14. Error Page Optimization

```typescript
// app/not-found.tsx
export const metadata = {
  title: '404 - Page Not Found',
  description: 'Sorry, the page you are looking for could not be found.',
  robots: {
    index: false,
    follow: false,
  },
}
```

### 15. Loading Performance

#### Bundle Optimization

```javascript
// next.config.js
const nextConfig = {
  experimental: {
    optimizeCss: true,
  },
  images: {
    formats: ['image/webp', 'image/avif'],
  },
  compress: true,
}
```

#### Lazy Loading

```typescript
// Implement lazy loading for non-critical components
import dynamic from 'next/dynamic'

const LazyComponent = dynamic(() => import('./LazyComponent'), {
  loading: () => <div>Loading...</div>,
})
```

### 16. SEO Monitoring Tools

1. **Google Search Console**: Monitor search performance
2. **Google Analytics 4**: Track user behavior
3. **Google PageSpeed Insights**: Monitor Core Web Vitals
4. **Lighthouse**: Regular performance audits
5. **Schema.org Validator**: Validate structured data
6. **Open Graph Debugger**: Test social media previews

### 17. Regular SEO Maintenance

#### Monthly Tasks

- [ ] Review search console errors
- [ ] Update sitemap
- [ ] Check broken links
- [ ] Monitor page speed
- [ ] Review analytics data
- [ ] Update meta descriptions for underperforming pages

#### Quarterly Tasks

- [ ] Comprehensive site audit
- [ ] Update schemas
- [ ] Review and update content
- [ ] Competitor analysis
- [ ] Technical SEO review

### 18. Content Types SEO Specifications

#### Blog Posts
- Title: 50-60 characters with target keyword
- Meta description: 150-160 characters
- Content: Minimum 1000 words
- Images: Alt text with keywords
- Internal links: 2-3 per post

#### Category Pages
- Title: "Category Name | Your Site"
- Description: Brief category description
- Content: Category overview with links to posts

#### Author Pages
- Title: "Author Name | Your Site"
- Description: Author bio summary
- Schema: Person markup

### Implementation Priority

1. **High Priority (Week 1)**
   - Meta tags and Open Graph
   - Structured data for articles
   - Sitemap generation
   - Robots.txt

2. **Medium Priority (Week 2)**
   - Breadcrumbs
   - Social sharing
   - Performance optimization
   - Analytics integration

3. **Low Priority (Week 3)**
   - Advanced schemas
   - Social media integration
   - Content optimization tools
   - Monitoring setup

This comprehensive SEO strategy ensures the website will be fully optimized for search engines and provide excellent user experience while maintaining high search rankings.