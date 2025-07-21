# CLAUDE.NEXTJS.md - Next.js Frontend Configuration

## Project Setup

### Dependencies Installation

```bash
# Create Next.js project
npx create-next-app@latest edu_nextjs --typescript --tailwind --eslint --app

# Install additional packages
npm install next-auth
npm install @auth/prisma-adapter  # If using Prisma
npm install ioredis
npm install @types/ioredis
npm install axios
npm install react-markdown
npm install @tailwindcss/typography
```

### Project Structure

```
edu_nextjs/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   └── register/
│   ├── posts/
│   │   ├── page.tsx
│   │   └── [slug]/
│   │       └── page.tsx
│   ├── about/
│   │   └── page.tsx
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/
│   ├── ui/
│   ├── Navigation.tsx
│   ├── PostCard.tsx
│   └── AuthProvider.tsx
├── lib/
│   ├── strapi.ts
│   ├── redis.ts
│   ├── auth.ts
│   └── utils.ts
├── types/
│   └── strapi.ts
└── middleware.ts
```

## Authentication Implementation

### NextAuth.js Configuration

**File: `lib/auth.ts`**
```typescript
import NextAuth from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'

export const authOptions = {
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' }
      },
      async authorize(credentials) {
        // Authenticate with Strapi
        const res = await fetch(`${process.env.STRAPI_API_URL}/auth/local`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            identifier: credentials?.email,
            password: credentials?.password,
          }),
        })
        
        const user = await res.json()
        
        if (res.ok && user) {
          return {
            id: user.user.id,
            email: user.user.email,
            token: user.jwt,
          }
        }
        return null
      }
    })
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.strapiToken = user.token
      }
      return token
    },
    async session({ session, token }) {
      session.strapiToken = token.strapiToken
      return session
    },
  },
}

export default NextAuth(authOptions)
```

### Protected Routes Middleware

**File: `middleware.ts`**
```typescript
import { withAuth } from 'next-auth/middleware'

export default withAuth(
  function middleware(req) {
    // Add any additional middleware logic
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token,
    },
  }
)

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*']
}
```

## Strapi Integration

### Strapi API Client

**File: `lib/strapi.ts`**
```typescript
import axios from 'axios'

const strapiApi = axios.create({
  baseURL: process.env.NEXT_PUBLIC_STRAPI_API_URL || 'http://localhost:1337/api',
})

// Strapi v5 response structure
export interface StrapiResponse<T> {
  data: T
  meta: {
    pagination?: {
      page: number
      pageSize: number
      pageCount: number
      total: number
    }
  }
}

export interface Post {
  id: number
  documentId: string // New in Strapi v5
  title: string
  slug: string
  content: string
  excerpt?: string
  publishedAt: string
  status: 'draft' | 'published' | 'archived'
  featuredImage?: {
    id: number
    url: string
    alternativeText?: string
    caption?: string
  }
  author?: {
    id: number
    username: string
    email: string
  }
  categories?: Category[]
  tags?: string[]
  viewCount: number
  createdAt: string
  updatedAt: string
}

export interface Page {
  id: number
  documentId: string
  title: string
  slug: string
  content: string
  template: string
  createdAt: string
  updatedAt: string
}

export interface Category {
  id: number
  name: string
  slug: string
  description?: string
}

// Fetch all published posts (slug-based)
export async function getPosts(page = 1, pageSize = 10): Promise<StrapiResponse<Post[]>> {
  const response = await strapiApi.get('/posts', {
    params: {
      populate: ['featuredImage', 'author', 'categories'],
      filters: {
        status: {
          $eq: 'published'
        }
      },
      sort: ['publishedAt:desc'],
      pagination: {
        page,
        pageSize
      }
    }
  })
  return response.data
}

// Fetch single post by slug (primary method)
export async function getPostBySlug(slug: string): Promise<Post | null> {
  try {
    const response = await strapiApi.get('/posts', {
      params: {
        filters: {
          slug: {
            $eq: slug
          },
          status: {
            $eq: 'published'
          }
        },
        populate: ['featuredImage', 'author', 'categories']
      }
    })
    
    const posts = response.data.data
    return posts.length > 0 ? posts[0] : null
  } catch (error) {
    console.error('Error fetching post by slug:', error)
    return null
  }
}

// Get all post slugs for static generation
export async function getAllPostSlugs(): Promise<string[]> {
  try {
    const response = await strapiApi.get('/posts', {
      params: {
        fields: ['slug'],
        filters: {
          status: {
            $eq: 'published'
          }
        }
      }
    })
    
    return response.data.data.map((post: any) => post.slug)
  } catch (error) {
    console.error('Error fetching post slugs:', error)
    return []
  }
}

// Fetch page by slug
export async function getPageBySlug(slug: string): Promise<Page | null> {
  try {
    const response = await strapiApi.get('/pages', {
      params: {
        filters: {
          slug: {
            $eq: slug
          }
        }
      }
    })
    
    const pages = response.data.data
    return pages.length > 0 ? pages[0] : null
  } catch (error) {
    console.error('Error fetching page by slug:', error)
    return null
  }
}

// Fetch navigation
export async function getNavigation() {
  try {
    const response = await strapiApi.get('/navigation', {
      params: {
        populate: ['items']
      }
    })
    return response.data.data
  } catch (error) {
    console.error('Error fetching navigation:', error)
    return null
  }
}

// Increment post view count
export async function incrementPostViews(slug: string): Promise<void> {
  try {
    const post = await getPostBySlug(slug)
    if (post) {
      await strapiApi.put(`/posts/${post.documentId}`, {
        data: {
          viewCount: post.viewCount + 1
        }
      })
    }
  } catch (error) {
    console.error('Error incrementing post views:', error)
  }
}

// Search posts by title or content
export async function searchPosts(query: string): Promise<Post[]> {
  try {
    const response = await strapiApi.get('/posts', {
      params: {
        filters: {
          $or: [
            {
              title: {
                $containsi: query
              }
            },
            {
              content: {
                $containsi: query
              }
            }
          ],
          status: {
            $eq: 'published'
          }
        },
        populate: ['featuredImage', 'author', 'categories']
      }
    })
    
    return response.data.data
  } catch (error) {
    console.error('Error searching posts:', error)
    return []
  }
}

// Additional helper functions for Strapi v5

// Get posts by category slug
export async function getPostsByCategory(categorySlug: string, page = 1, pageSize = 10): Promise<StrapiResponse<Post[]>> {
  const response = await strapiApi.get('/posts', {
    params: {
      filters: {
        categories: {
          slug: {
            $eq: categorySlug
          }
        },
        status: {
          $eq: 'published'
        }
      },
      populate: ['featuredImage', 'author', 'categories'],
      sort: ['publishedAt:desc'],
      pagination: {
        page,
        pageSize
      }
    }
  })
  return response.data
}

// Get related posts based on categories
export async function getRelatedPosts(postSlug: string, limit = 3): Promise<Post[]> {
  try {
    const currentPost = await getPostBySlug(postSlug)
    if (!currentPost || !currentPost.categories?.length) {
      return []
    }

    const categoryIds = currentPost.categories.map(cat => cat.id)
    
    const response = await strapiApi.get('/posts', {
      params: {
        filters: {
          categories: {
            id: {
              $in: categoryIds
            }
          },
          slug: {
            $ne: postSlug
          },
          status: {
            $eq: 'published'
          }
        },
        populate: ['featuredImage', 'categories'],
        sort: ['publishedAt:desc'],
        pagination: {
          pageSize: limit
        }
      }
    })
    
    return response.data.data
  } catch (error) {
    console.error('Error fetching related posts:', error)
    return []
  }
}

// Get posts by tag
export async function getPostsByTag(tag: string, page = 1, pageSize = 10): Promise<StrapiResponse<Post[]>> {
  const response = await strapiApi.get('/posts', {
    params: {
      filters: {
        tags: {
          $contains: tag
        },
        status: {
          $eq: 'published'
        }
      },
      populate: ['featuredImage', 'author', 'categories'],
      sort: ['publishedAt:desc'],
      pagination: {
        page,
        pageSize
      }
    }
  })
  return response.data
}

// Get popular posts by view count
export async function getPopularPosts(limit = 5): Promise<Post[]> {
  try {
    const response = await strapiApi.get('/posts', {
      params: {
        filters: {
          status: {
            $eq: 'published'
          }
        },
        populate: ['featuredImage'],
        sort: ['viewCount:desc'],
        pagination: {
          pageSize: limit
        }
      }
    })
    
    return response.data.data
  } catch (error) {
    console.error('Error fetching popular posts:', error)
    return []
  }
}

// Validate slug format
export function isValidSlug(slug: string): boolean {
  const slugPattern = /^[a-z0-9]+(?:-[a-z0-9]+)*$/
  return slugPattern.test(slug)
}

// Fetch global SEO settings
export async function getGlobalSeoSettings() {
  try {
    const response = await strapiApi.get('/global-seo-setting')
    return response.data.data
  } catch (error) {
    console.error('Error fetching global SEO settings:', error)
    return null
  }
}

// Build SEO metadata from Strapi data
export function buildSeoMetadata(content: any, globalSeo: any, path: string) {
  const seo = content.seo || {}
  
  return {
    title: seo.metaTitle || content.title || globalSeo?.siteName,
    description: seo.metaDescription || content.excerpt || globalSeo?.siteDescription,
    keywords: seo.keywords || content.tags,
    canonical: seo.canonicalURL || path,
    openGraph: {
      title: seo.ogTitle || seo.metaTitle || content.title,
      description: seo.ogDescription || seo.metaDescription || content.excerpt,
      images: seo.ogImage ? [seo.ogImage.url] : content.featuredImage ? [content.featuredImage.url] : globalSeo?.defaultOgImage ? [globalSeo.defaultOgImage.url] : [],
      type: content.publishedAt ? 'article' : 'website',
    },
    twitter: {
      title: seo.twitterTitle || seo.ogTitle || seo.metaTitle || content.title,
      description: seo.twitterDescription || seo.ogDescription || seo.metaDescription || content.excerpt,
      images: seo.twitterImage ? [seo.twitterImage.url] : seo.ogImage ? [seo.ogImage.url] : content.featuredImage ? [content.featuredImage.url] : [],
    },
    robots: {
      index: !seo.noIndex,
      follow: !seo.noFollow,
    },
  }
}
```

## Redis Caching Implementation

### Redis Client Setup

**File: `lib/redis.ts`**
```typescript
import Redis from 'ioredis'

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
})

export async function getCachedData<T>(key: string): Promise<T | null> {
  try {
    const cached = await redis.get(key)
    return cached ? JSON.parse(cached) : null
  } catch (error) {
    console.error('Redis get error:', error)
    return null
  }
}

export async function setCachedData(
  key: string,
  data: any,
  ttl: number = 3600
): Promise<void> {
  try {
    await redis.setex(key, ttl, JSON.stringify(data))
  } catch (error) {
    console.error('Redis set error:', error)
  }
}

export async function deleteCachedData(key: string): Promise<void> {
  try {
    await redis.del(key)
  } catch (error) {
    console.error('Redis delete error:', error)
  }
}

export default redis
```

### Cached API Functions

**File: `lib/strapi-cached.ts`**
```typescript
import { getCachedData, setCachedData, deleteCachedData } from './redis'
import { getPosts, getPostBySlug, getPageBySlug, getNavigation, getAllPostSlugs } from './strapi'

export async function getCachedPosts(page = 1, pageSize = 10) {
  const cacheKey = `posts:page:${page}:${pageSize}`
  let posts = await getCachedData(cacheKey)
  
  if (!posts) {
    posts = await getPosts(page, pageSize)
    await setCachedData(cacheKey, posts, 1800) // 30 minutes
  }
  
  return posts
}

export async function getCachedPostBySlug(slug: string) {
  const cacheKey = `post:slug:${slug}`
  let post = await getCachedData(cacheKey)
  
  if (!post) {
    post = await getPostBySlug(slug)
    if (post) {
      await setCachedData(cacheKey, post, 3600) // 1 hour
    }
  }
  
  return post
}

export async function getCachedPageBySlug(slug: string) {
  const cacheKey = `page:slug:${slug}`
  let page = await getCachedData(cacheKey)
  
  if (!page) {
    page = await getPageBySlug(slug)
    if (page) {
      await setCachedData(cacheKey, page, 7200) // 2 hours
    }
  }
  
  return page
}

export async function getCachedNavigation() {
  const cacheKey = 'navigation:main'
  let navigation = await getCachedData(cacheKey)
  
  if (!navigation) {
    navigation = await getNavigation()
    await setCachedData(cacheKey, navigation, 3600) // 1 hour
  }
  
  return navigation
}

export async function getCachedPostSlugs() {
  const cacheKey = 'posts:slugs:all'
  let slugs = await getCachedData(cacheKey)
  
  if (!slugs) {
    slugs = await getAllPostSlugs()
    await setCachedData(cacheKey, slugs, 1800) // 30 minutes
  }
  
  return slugs
}

// Cache invalidation helpers
export async function invalidatePostCache(slug: string) {
  const keys = [
    `post:slug:${slug}`,
    'posts:slugs:all',
    // Invalidate paginated posts cache
    ...Array.from({length: 10}, (_, i) => `posts:page:${i + 1}:10`)
  ]
  
  for (const key of keys) {
    await deleteCachedData(key)
  }
}
```

## SSG Implementation for Posts

### Post Detail Page with SSG

**File: `app/posts/[slug]/page.tsx`**
```typescript
import { notFound } from 'next/navigation'
import { getPostBySlug, getAllPostSlugs, incrementPostViews, getGlobalSeoSettings, buildSeoMetadata } from '@/lib/strapi'
import ReactMarkdown from 'react-markdown'
import Image from 'next/image'

interface PostPageProps {
  params: {
    slug: string
  }
}

// Generate static paths for all posts using slugs
export async function generateStaticParams() {
  const slugs = await getAllPostSlugs()
  
  return slugs.map((slug) => ({
    slug: slug,
  }))
}

// Generate metadata for SEO using slug with Strapi SEO data
export async function generateMetadata({ params }: PostPageProps) {
  const [post, globalSeo] = await Promise.all([
    getPostBySlug(params.slug),
    getGlobalSeoSettings()
  ])
  
  if (!post) {
    return {
      title: 'Post Not Found',
      robots: { index: false, follow: false }
    }
  }
  
  const seoData = buildSeoMetadata(post, globalSeo, `/posts/${post.slug}`)
  
  return {
    title: seoData.title,
    description: seoData.description,
    keywords: seoData.keywords?.join(', '),
    openGraph: {
      ...seoData.openGraph,
      publishedTime: post.publishedAt,
      modifiedTime: post.updatedAt,
      authors: post.author ? [post.author.username] : [],
      section: post.categories?.[0]?.name,
      tags: post.tags,
      url: `/posts/${post.slug}`,
      siteName: globalSeo?.siteName,
      images: seoData.openGraph.images.map(url => ({
        url,
        width: 1200,
        height: 630,
        alt: post.seo?.ogImage?.alternativeText || post.featuredImage?.alternativeText || post.title,
      })),
    },
    twitter: {
      card: 'summary_large_image',
      ...seoData.twitter,
      creator: globalSeo?.twitterHandle,
    },
    alternates: {
      canonical: seoData.canonical,
    },
    robots: seoData.robots,
  }
}

export default async function PostPage({ params }: PostPageProps) {
  const post = await getPostBySlug(params.slug)
  
  if (!post) {
    notFound()
  }
  
  // Increment view count (non-blocking)
  incrementPostViews(params.slug).catch(console.error)
  
  const publishedDate = new Date(post.publishedAt).toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
  
  return (
    <article className="max-w-4xl mx-auto px-4 py-8">
      <header className="mb-8">
        <div className="mb-4">
          {post.categories && post.categories.length > 0 && (
            <div className="flex flex-wrap gap-2 mb-4">
              {post.categories.map((category) => (
                <span
                  key={category.id}
                  className="px-3 py-1 bg-blue-100 text-blue-800 text-sm rounded-full"
                >
                  {category.name}
                </span>
              ))}
            </div>
          )}
        </div>
        
        <h1 className="text-4xl font-bold mb-4 leading-tight">
          {post.title}
        </h1>
        
        <div className="flex items-center text-gray-600 mb-6 space-x-4">
          <time dateTime={post.publishedAt}>
            Published: {publishedDate}
          </time>
          {post.author && (
            <span>By: {post.author.username}</span>
          )}
          <span>{post.viewCount} views</span>
        </div>
        
        {post.featuredImage && (
          <div className="relative w-full h-64 md:h-96 mb-6 rounded-lg overflow-hidden">
            <Image
              src={post.featuredImage.url}
              alt={post.featuredImage.alternativeText || post.title}
              fill
              className="object-cover"
              priority
            />
            {post.featuredImage.caption && (
              <p className="text-sm text-gray-500 mt-2 text-center italic">
                {post.featuredImage.caption}
              </p>
            )}
          </div>
        )}
        
        {post.excerpt && (
          <div className="text-lg text-gray-700 mb-6 font-medium leading-relaxed">
            {post.excerpt}
          </div>
        )}
      </header>
      
      <div className="prose prose-lg max-w-none">
        <ReactMarkdown
          components={{
            img: ({ src, alt }) => (
              <Image
                src={src || ''}
                alt={alt || ''}
                width={800}
                height={400}
                className="rounded-lg"
              />
            ),
          }}
        >
          {post.content}
        </ReactMarkdown>
      </div>
      
      {post.tags && post.tags.length > 0 && (
        <footer className="mt-8 pt-8 border-t">
          <div className="flex flex-wrap gap-2">
            <span className="text-gray-600 font-medium">Tags:</span>
            {post.tags.map((tag) => (
              <span
                key={tag}
                className="px-2 py-1 bg-gray-100 text-gray-700 text-sm rounded"
              >
                #{tag}
              </span>
            ))}
          </div>
        </footer>
      )}
      
      {/* Schema.org structured data */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{
          __html: JSON.stringify({
            "@context": "https://schema.org",
            "@type": "BlogPosting",
            "headline": post.title,
            "description": post.excerpt || post.content.substring(0, 160),
            "image": post.featuredImage?.url,
            "datePublished": post.publishedAt,
            "dateModified": post.updatedAt,
            "author": {
              "@type": "Person",
              "name": post.author?.username || "Anonymous"
            },
            "url": `/posts/${post.slug}`,
            "mainEntityOfPage": {
              "@type": "WebPage",
              "@id": `/posts/${post.slug}`
            }
          })
        }}
      />
    </article>
  )
}

// Enable ISR (Incremental Static Regeneration) with slug-based revalidation
export const revalidate = 3600 // Revalidate every hour
```

### Posts Listing Page

**File: `app/posts/page.tsx`**
```typescript
import Link from 'next/link'
import Image from 'next/image'
import { getCachedPosts } from '@/lib/strapi-cached'

interface PostsPageProps {
  searchParams: {
    page?: string
  }
}

export default async function PostsPage({ searchParams }: PostsPageProps) {
  const currentPage = parseInt(searchParams.page || '1')
  const pageSize = 12
  const postsResponse = await getCachedPosts(currentPage, pageSize)
  
  const posts = postsResponse.data
  const pagination = postsResponse.meta.pagination
  
  return (
    <div className="max-w-6xl mx-auto px-4 py-8">
      <header className="mb-8">
        <h1 className="text-4xl font-bold mb-4">Blog Posts</h1>
        <p className="text-gray-600">
          {pagination && `Showing ${posts.length} of ${pagination.total} posts`}
        </p>
      </header>
      
      {posts.length === 0 ? (
        <div className="text-center py-12">
          <h2 className="text-2xl font-semibold text-gray-600 mb-2">No posts found</h2>
          <p className="text-gray-500">Check back later for new content!</p>
        </div>
      ) : (
        <>
          <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-6 mb-8">
            {posts.map((post) => (
              <article key={post.slug} className="border rounded-lg overflow-hidden shadow-sm hover:shadow-md transition-shadow">
                {post.featuredImage && (
                  <div className="relative w-full h-48">
                    <Image
                      src={post.featuredImage.url}
                      alt={post.featuredImage.alternativeText || post.title}
                      fill
                      className="object-cover"
                    />
                  </div>
                )}
                
                <div className="p-6">
                  {post.categories && post.categories.length > 0 && (
                    <div className="flex flex-wrap gap-1 mb-2">
                      {post.categories.slice(0, 2).map((category) => (
                        <span
                          key={category.id}
                          className="px-2 py-1 bg-blue-100 text-blue-800 text-xs rounded"
                        >
                          {category.name}
                        </span>
                      ))}
                    </div>
                  )}
                  
                  <h2 className="text-xl font-semibold mb-2 leading-tight">
                    <Link 
                      href={`/posts/${post.slug}`}
                      className="hover:text-blue-600 transition-colors"
                    >
                      {post.title}
                    </Link>
                  </h2>
                  
                  {post.excerpt && (
                    <p className="text-gray-600 mb-4 line-clamp-3">
                      {post.excerpt}
                    </p>
                  )}
                  
                  <div className="flex items-center justify-between text-sm text-gray-500">
                    <time dateTime={post.publishedAt}>
                      {new Date(post.publishedAt).toLocaleDateString('en-US', {
                        year: 'numeric',
                        month: 'short',
                        day: 'numeric'
                      })}
                    </time>
                    <span>{post.viewCount} views</span>
                  </div>
                  
                  {post.tags && post.tags.length > 0 && (
                    <div className="mt-3 flex flex-wrap gap-1">
                      {post.tags.slice(0, 3).map((tag) => (
                        <span
                          key={tag}
                          className="px-2 py-1 bg-gray-100 text-gray-600 text-xs rounded"
                        >
                          #{tag}
                        </span>
                      ))}
                    </div>
                  )}
                </div>
              </article>
            ))}
          </div>
          
          {/* Pagination */}
          {pagination && pagination.pageCount > 1 && (
            <div className="flex justify-center items-center space-x-2">
              {currentPage > 1 && (
                <Link
                  href={`/posts?page=${currentPage - 1}`}
                  className="px-4 py-2 border rounded-md hover:bg-gray-50 transition-colors"
                >
                  Previous
                </Link>
              )}
              
              <span className="px-4 py-2 text-gray-600">
                Page {currentPage} of {pagination.pageCount}
              </span>
              
              {currentPage < pagination.pageCount && (
                <Link
                  href={`/posts?page=${currentPage + 1}`}
                  className="px-4 py-2 border rounded-md hover:bg-gray-50 transition-colors"
                >
                  Next
                </Link>
              )}
            </div>
          )}
        </>
      )}
    </div>
  )
}

// Add some basic SEO
export const metadata = {
  title: 'Blog Posts | My Blog',
  description: 'Read our latest blog posts and articles',
}
```

## Navigation Component

**File: `components/Navigation.tsx`**
```typescript
'use client'

import Link from 'next/link'
import { useSession, signIn, signOut } from 'next-auth/react'
import { getCachedNavigation } from '@/lib/strapi-cached'
import { useEffect, useState } from 'react'

interface NavigationItem {
  label: string
  url: string
  isExternal: boolean
  order: number
}

export default function Navigation() {
  const { data: session, status } = useSession()
  const [navItems, setNavItems] = useState<NavigationItem[]>([])
  
  useEffect(() => {
    async function loadNavigation() {
      const navigation = await getCachedNavigation()
      if (navigation?.attributes?.items) {
        setNavItems(navigation.attributes.items.sort((a: any, b: any) => a.order - b.order))
      }
    }
    loadNavigation()
  }, [])
  
  return (
    <nav className="bg-white shadow-sm border-b">
      <div className="max-w-7xl mx-auto px-4">
        <div className="flex justify-between items-center h-16">
          <Link href="/" className="text-xl font-bold">
            My Blog
          </Link>
          
          <div className="flex items-center space-x-6">
            {navItems.map((item) => (
              <Link
                key={item.url}
                href={item.url}
                target={item.isExternal ? '_blank' : undefined}
                rel={item.isExternal ? 'noopener noreferrer' : undefined}
                className="text-gray-600 hover:text-gray-900 transition-colors"
              >
                {item.label}
              </Link>
            ))}
            
            {status === 'loading' ? (
              <div className="animate-pulse bg-gray-200 h-8 w-16 rounded"></div>
            ) : session ? (
              <div className="flex items-center space-x-4">
                <span className="text-gray-600">
                  Welcome, {session.user?.email}
                </span>
                <button
                  onClick={() => signOut()}
                  className="bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600 transition-colors"
                >
                  Sign Out
                </button>
              </div>
            ) : (
              <button
                onClick={() => signIn()}
                className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600 transition-colors"
              >
                Sign In
              </button>
            )}
          </div>
        </div>
      </div>
    </nav>
  )
}
```

## Environment Variables

### Frontend (.env.local)
```env
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key-here

NEXT_PUBLIC_STRAPI_API_URL=http://localhost:1337/api
STRAPI_API_URL=http://localhost:1337/api

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

DATABASE_URL=postgresql://username:password@localhost:5432/mydb
```

## Next.js Configuration

### File: `next.config.js`
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'http',
        hostname: 'localhost',
        port: '1337',
        pathname: '/uploads/**',
      },
      {
        protocol: 'https',
        hostname: 'res.cloudinary.com',
        pathname: '/**',
      },
      {
        protocol: 'https',
        hostname: 'your-strapi-domain.com',
        pathname: '/uploads/**',
      },
    ],
  },
  
  // Enable experimental features for better performance
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ['lucide-react'],
  },
  
  // Compression and optimization
  compress: true,
  poweredByHeader: false,
  
  // Redirect configuration
  async redirects() {
    return [
      {
        source: '/blog/:path*',
        destination: '/posts/:path*',
        permanent: true,
      },
    ]
  },
  
  // Rewrite configuration for better SEO
  async rewrites() {
    return [
      {
        source: '/sitemap.xml',
        destination: '/api/sitemap',
      },
      {
        source: '/robots.txt',
        destination: '/api/robots',
      },
    ]
  },
}

module.exports = nextConfig
```