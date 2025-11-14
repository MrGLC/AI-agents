# Project 3: Blog Platform with Next.js and MDX

## Overview
Build a modern, SEO-optimized blog platform using Next.js 14 with App Router, MDX for content authoring, and advanced features like syntax highlighting, reading time estimation, table of contents, and RSS feeds. This project focuses on static site generation, dynamic routing, and content management.

## Difficulty Level
Advanced

## Learning Objectives
- Master Next.js 14 App Router and Server Components
- Implement MDX for rich content authoring
- Configure static site generation (SSG) and incremental static regeneration (ISR)
- Build SEO-optimized pages with metadata API
- Create custom MDX components and plugins
- Implement full-text search
- Generate RSS and sitemap
- Add analytics and reading statistics
- Optimize images with Next.js Image component

## Technical Stack
- **Framework**: Next.js 14 (App Router)
- **Content**: MDX with gray-matter for frontmatter
- **Styling**: Tailwind CSS with Typography plugin
- **Syntax Highlighting**: Shiki or Prism
- **Search**: Flexsearch or Fuse.js
- **Analytics**: Reading time, view counts
- **SEO**: next-sitemap, RSS feed generation
- **Deployment**: Vercel
- **TypeScript**: Full type safety

## Project Requirements

### 1. Blog Features
- **Content Management**
  - MDX-based blog posts with frontmatter
  - Categories and tags
  - Author profiles
  - Featured images
  - Draft/published status
  - Publication dates

- **Post Display**
  - Blog post listing with pagination
  - Individual post pages
  - Related posts
  - Reading time estimation
  - Table of contents (auto-generated)
  - Social sharing buttons
  - Previous/next post navigation

- **Custom MDX Components**
  - Syntax-highlighted code blocks
  - Callouts/alerts
  - YouTube/Twitter embeds
  - Image galleries
  - Interactive demos
  - Mermaid diagrams

### 2. Navigation and Discovery
- Homepage with featured posts
- Category pages
- Tag pages
- Author pages
- Search functionality
- Archive (posts by date)
- RSS feed
- Sitemap

### 3. SEO and Performance
- Dynamic metadata for each post
- Open Graph images
- Twitter cards
- Structured data (JSON-LD)
- Optimized images
- Static generation for fast loading
- Lighthouse score 95+

### 4. UI Components
- Responsive header with navigation
- Footer with links and newsletter signup
- Post card components
- Breadcrumbs
- Loading states
- 404 page
- About page

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
# Create Next.js project
npx create-next-app@latest blog-platform --typescript --tailwind --app --eslint
cd blog-platform

# Install dependencies
npm install @next/mdx @mdx-js/loader @mdx-js/react
npm install gray-matter reading-time
npm install shiki rehype-pretty-code
npm install @tailwindcss/typography
npm install date-fns
npm install flexsearch
npm install rss next-sitemap
npm install -D @types/mdx
```

### Step 2: Configure MDX
```typescript
// next.config.js
import createMDX from '@next/mdx';
import rehypePrettyCode from 'rehype-pretty-code';

/** @type {import('next').NextConfig} */
const nextConfig = {
  pageExtensions: ['js', 'jsx', 'md', 'mdx', 'ts', 'tsx'],
  experimental: {
    mdxRs: true,
  },
};

const withMDX = createMDX({
  extension: /\.mdx?$/,
  options: {
    remarkPlugins: [],
    rehypePlugins: [
      [
        rehypePrettyCode,
        {
          theme: 'github-dark',
          keepBackground: false,
        },
      ],
    ],
  },
});

export default withMDX(nextConfig);
```

### Step 3: Define TypeScript Types
```typescript
// src/types/blog.ts
export interface Post {
  slug: string;
  title: string;
  description: string;
  date: string;
  author: string;
  category: string;
  tags: string[];
  image?: string;
  draft?: boolean;
  readingTime: string;
  content: string;
}

export interface PostMeta {
  slug: string;
  title: string;
  description: string;
  date: string;
  author: string;
  category: string;
  tags: string[];
  image?: string;
  readingTime: string;
}

export interface Author {
  name: string;
  bio: string;
  avatar: string;
  twitter?: string;
  github?: string;
  website?: string;
}

export interface Category {
  name: string;
  slug: string;
  description: string;
  count: number;
}
```

### Step 4: Create Post Utilities
```typescript
// src/lib/posts.ts
import fs from 'fs';
import path from 'path';
import matter from 'gray-matter';
import readingTime from 'reading-time';
import type { Post, PostMeta } from '@/types/blog';

const postsDirectory = path.join(process.cwd(), 'content/posts');

export function getPostSlugs(): string[] {
  return fs.readdirSync(postsDirectory).filter((file) => file.endsWith('.mdx'));
}

export function getPostBySlug(slug: string): Post {
  const realSlug = slug.replace(/\.mdx$/, '');
  const fullPath = path.join(postsDirectory, `${realSlug}.mdx`);
  const fileContents = fs.readFileSync(fullPath, 'utf8');
  const { data, content } = matter(fileContents);

  const stats = readingTime(content);

  return {
    slug: realSlug,
    title: data.title,
    description: data.description,
    date: data.date,
    author: data.author,
    category: data.category,
    tags: data.tags || [],
    image: data.image,
    draft: data.draft || false,
    readingTime: stats.text,
    content,
  };
}

export function getAllPosts(): Post[] {
  const slugs = getPostSlugs();
  const posts = slugs
    .map((slug) => getPostBySlug(slug))
    .filter((post) => !post.draft)
    .sort((a, b) => (new Date(b.date).getTime() - new Date(a.date).getTime()));

  return posts;
}

export function getPostsByCategory(category: string): PostMeta[] {
  const allPosts = getAllPosts();
  return allPosts
    .filter((post) => post.category.toLowerCase() === category.toLowerCase())
    .map(({ content, ...meta }) => meta);
}

export function getPostsByTag(tag: string): PostMeta[] {
  const allPosts = getAllPosts();
  return allPosts
    .filter((post) => post.tags.some((t) => t.toLowerCase() === tag.toLowerCase()))
    .map(({ content, ...meta }) => meta);
}

export function getAllCategories(): Category[] {
  const posts = getAllPosts();
  const categoryMap = new Map<string, number>();

  posts.forEach((post) => {
    const count = categoryMap.get(post.category) || 0;
    categoryMap.set(post.category, count + 1);
  });

  return Array.from(categoryMap.entries()).map(([name, count]) => ({
    name,
    slug: name.toLowerCase().replace(/\s+/g, '-'),
    description: `Posts about ${name}`,
    count,
  }));
}

export function getAllTags(): string[] {
  const posts = getAllPosts();
  const tags = new Set<string>();

  posts.forEach((post) => {
    post.tags.forEach((tag) => tags.add(tag));
  });

  return Array.from(tags).sort();
}
```

### Step 5: Create Custom MDX Components
```typescript
// src/components/mdx/MDXComponents.tsx
import Image from 'next/image';
import { ReactNode } from 'react';

interface CalloutProps {
  children: ReactNode;
  type?: 'info' | 'warning' | 'error' | 'success';
}

function Callout({ children, type = 'info' }: CalloutProps) {
  const styles = {
    info: 'bg-blue-50 border-blue-500 text-blue-900',
    warning: 'bg-yellow-50 border-yellow-500 text-yellow-900',
    error: 'bg-red-50 border-red-500 text-red-900',
    success: 'bg-green-50 border-green-500 text-green-900',
  };

  return (
    <div className={`border-l-4 p-4 my-6 ${styles[type]}`}>
      {children}
    </div>
  );
}

interface YouTubeProps {
  id: string;
}

function YouTube({ id }: YouTubeProps) {
  return (
    <div className="relative aspect-video my-8">
      <iframe
        src={`https://www.youtube.com/embed/${id}`}
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
        allowFullScreen
        className="absolute inset-0 w-full h-full rounded-lg"
      />
    </div>
  );
}

function CustomImage(props: any) {
  return (
    <div className="relative my-8">
      <Image
        {...props}
        width={1200}
        height={630}
        className="rounded-lg"
        alt={props.alt}
      />
      {props.caption && (
        <p className="text-center text-sm text-gray-600 mt-2">{props.caption}</p>
      )}
    </div>
  );
}

export const MDXComponents = {
  img: CustomImage,
  Image: CustomImage,
  Callout,
  YouTube,
  pre: ({ children, ...props }: any) => (
    <pre className="overflow-x-auto rounded-lg p-4 my-6" {...props}>
      {children}
    </pre>
  ),
  code: ({ children, ...props }: any) => (
    <code className="bg-gray-100 rounded px-1 py-0.5 text-sm" {...props}>
      {children}
    </code>
  ),
};
```

### Step 6: Create Blog Post Page
```typescript
// src/app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation';
import { getPostBySlug, getAllPosts } from '@/lib/posts';
import { MDXRemote } from 'next-mdx-remote/rsc';
import { MDXComponents } from '@/components/mdx/MDXComponents';
import { format } from 'date-fns';
import Image from 'next/image';
import type { Metadata } from 'next';

interface Props {
  params: { slug: string };
}

export async function generateStaticParams() {
  const posts = getAllPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = getPostBySlug(params.slug);

  if (!post) {
    return {};
  }

  return {
    title: post.title,
    description: post.description,
    authors: [{ name: post.author }],
    openGraph: {
      title: post.title,
      description: post.description,
      type: 'article',
      publishedTime: post.date,
      images: post.image ? [post.image] : [],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.description,
      images: post.image ? [post.image] : [],
    },
  };
}

export default function BlogPost({ params }: Props) {
  const post = getPostBySlug(params.slug);

  if (!post) {
    notFound();
  }

  return (
    <article className="max-w-4xl mx-auto px-4 py-12">
      {/* Header */}
      <header className="mb-8">
        <div className="flex items-center gap-2 text-sm text-gray-600 mb-4">
          <time dateTime={post.date}>
            {format(new Date(post.date), 'MMMM dd, yyyy')}
          </time>
          <span>•</span>
          <span>{post.readingTime}</span>
        </div>

        <h1 className="text-4xl md:text-5xl font-bold mb-4">{post.title}</h1>

        <p className="text-xl text-gray-600 mb-6">{post.description}</p>

        <div className="flex items-center gap-4">
          <div className="flex items-center gap-2">
            <div className="w-10 h-10 rounded-full bg-gray-200" />
            <div>
              <p className="font-medium">{post.author}</p>
            </div>
          </div>
        </div>

        <div className="flex gap-2 mt-4">
          {post.tags.map((tag) => (
            <span
              key={tag}
              className="px-3 py-1 bg-gray-100 text-gray-700 rounded-full text-sm"
            >
              #{tag}
            </span>
          ))}
        </div>
      </header>

      {/* Featured Image */}
      {post.image && (
        <div className="relative aspect-video mb-8 rounded-lg overflow-hidden">
          <Image
            src={post.image}
            alt={post.title}
            fill
            className="object-cover"
          />
        </div>
      )}

      {/* Content */}
      <div className="prose prose-lg max-w-none">
        <MDXRemote source={post.content} components={MDXComponents} />
      </div>

      {/* Footer */}
      <footer className="mt-12 pt-8 border-t">
        <div className="flex gap-4">
          <button className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700">
            Share on Twitter
          </button>
          <button className="px-4 py-2 bg-gray-100 text-gray-700 rounded-md hover:bg-gray-200">
            Copy Link
          </button>
        </div>
      </footer>
    </article>
  );
}
```

### Step 7: Create Blog Listing Page
```typescript
// src/app/blog/page.tsx
import Link from 'next/link';
import Image from 'next/image';
import { getAllPosts } from '@/lib/posts';
import { format } from 'date-fns';

export const metadata = {
  title: 'Blog',
  description: 'Read our latest articles and tutorials',
};

export default function BlogPage() {
  const posts = getAllPosts();

  return (
    <div className="max-w-7xl mx-auto px-4 py-12">
      <header className="mb-12">
        <h1 className="text-4xl md:text-5xl font-bold mb-4">Blog</h1>
        <p className="text-xl text-gray-600">
          Articles, tutorials, and thoughts on web development
        </p>
      </header>

      <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-8">
        {posts.map((post) => (
          <Link
            key={post.slug}
            href={`/blog/${post.slug}`}
            className="group"
          >
            <article className="border rounded-lg overflow-hidden hover:shadow-lg transition-shadow">
              {post.image && (
                <div className="relative aspect-video">
                  <Image
                    src={post.image}
                    alt={post.title}
                    fill
                    className="object-cover group-hover:scale-105 transition-transform"
                  />
                </div>
              )}

              <div className="p-6">
                <div className="flex items-center gap-2 text-sm text-gray-600 mb-2">
                  <time dateTime={post.date}>
                    {format(new Date(post.date), 'MMM dd, yyyy')}
                  </time>
                  <span>•</span>
                  <span>{post.readingTime}</span>
                </div>

                <h2 className="text-xl font-bold mb-2 group-hover:text-blue-600 transition-colors">
                  {post.title}
                </h2>

                <p className="text-gray-600 mb-4 line-clamp-2">
                  {post.description}
                </p>

                <div className="flex items-center justify-between">
                  <span className="text-sm text-gray-500">{post.category}</span>
                  <span className="text-blue-600 text-sm font-medium">
                    Read more →
                  </span>
                </div>
              </div>
            </article>
          </Link>
        ))}
      </div>
    </div>
  );
}
```

### Step 8: Create Example Blog Post
```mdx
---
title: "Getting Started with Next.js 14 and MDX"
description: "Learn how to build a modern blog platform using Next.js 14 App Router and MDX for content authoring"
date: "2024-01-15"
author: "John Doe"
category: "Tutorial"
tags: ["nextjs", "mdx", "react", "tutorial"]
image: "/images/blog/nextjs-mdx.jpg"
---

# Getting Started with Next.js 14 and MDX

Next.js 14 brings powerful features for building content-rich websites. Combined with MDX, you can create engaging blog posts with interactive components.

## What is MDX?

MDX allows you to use JSX in your markdown content. This means you can import and use React components directly in your blog posts!

<Callout type="info">
MDX is a superset of Markdown that lets you write JSX directly in your content files.
</Callout>

## Code Examples

Here's a simple React component:

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
}

export function Button({ label, onClick }: ButtonProps) {
  return (
    <button onClick={onClick} className="px-4 py-2 bg-blue-600 text-white rounded">
      {label}
    </button>
  );
}
```

## Embedding Media

You can easily embed YouTube videos:

<YouTube id="dQw4w9WgXcQ" />

## Conclusion

Next.js 14 with MDX provides an excellent foundation for building modern, performant blogs with rich content.
```

## Expected Outputs

1. **Fully Functional Blog Platform** with:
   - Static generated pages for optimal performance
   - MDX-powered content with custom components
   - SEO-optimized metadata
   - Responsive design

2. **Content Features**:
   - Syntax-highlighted code blocks
   - Custom callouts and alerts
   - Embedded media (YouTube, images)
   - Reading time estimation
   - Author and date information

3. **Navigation**:
   - Category and tag pages
   - Search functionality
   - Related posts
   - Archive

4. **Performance**:
   - Lighthouse score 95+
   - Fast page loads with SSG
   - Optimized images
   - Minimal JavaScript

## Bonus Challenges

- [ ] Add comment system (Giscus or Utterances)
- [ ] Implement newsletter subscription
- [ ] Add view counter with database
- [ ] Create admin panel for content management
- [ ] Add full-text search with Algolia
- [ ] Implement i18n for multiple languages
- [ ] Add light/dark mode toggle
- [ ] Create series/collection feature
- [ ] Add content analytics dashboard
- [ ] Implement bookmarking feature
- [ ] Add estimated reading progress bar
- [ ] Create email digest of latest posts
- [ ] Add content recommendation engine
- [ ] Implement lazy loading for images

## Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [MDX Documentation](https://mdxjs.com/)
- [Tailwind Typography](https://tailwindcss.com/docs/typography-plugin)
- [rehype-pretty-code](https://rehype-pretty-code.netlify.app/)
- [Next Sitemap](https://github.com/iamvishnusankar/next-sitemap)
- [RSS Feed Generation](https://www.npmjs.com/package/rss)

## Success Criteria

- Blog posts load instantly with SSG
- All MDX custom components render correctly
- Syntax highlighting works for all languages
- SEO metadata is complete and accurate
- Images are optimized and lazy-loaded
- Site is fully responsive
- RSS feed is valid and accessible
- Sitemap is generated automatically
- TypeScript has no errors
- Lighthouse performance score is 95+
- Content is easily discoverable through categories, tags, and search
- Reading experience is smooth and engaging
