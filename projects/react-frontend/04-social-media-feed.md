# Project 4: Social Media Feed with Infinite Scroll

## Overview
Build a dynamic social media feed application with infinite scrolling, real-time updates, post interactions (likes, comments, shares), and user profiles. This project emphasizes performance optimization, virtual scrolling, and creating an engaging user experience similar to Twitter/X or Instagram.

## Difficulty Level
Advanced

## Learning Objectives
- Implement infinite scroll with React Query
- Master virtual scrolling for performance
- Handle real-time updates with optimistic UI
- Create complex interaction patterns (like, comment, share)
- Optimize image loading and rendering
- Implement pull-to-refresh functionality
- Build responsive timeline layouts
- Manage complex state with nested data structures
- Implement skeleton loading states

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Data Fetching**: React Query (TanStack Query) with infinite queries
- **Virtual Scrolling**: react-virtual or react-window
- **State Management**: Zustand or Context API
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **Image Handling**: React Lazy Load Image
- **Animations**: Framer Motion
- **Real-time**: WebSocket or polling with React Query
- **Build Tool**: Vite

## Project Requirements

### 1. Feed Features
- **Infinite Scroll**
  - Load posts dynamically as user scrolls
  - Show loading indicators
  - Handle end of feed
  - Pull-to-refresh on mobile
  - Maintain scroll position on navigation

- **Post Types**
  - Text-only posts
  - Posts with single image
  - Posts with multiple images (carousel)
  - Posts with videos
  - Reposted content
  - Quoted posts

- **Post Interactions**
  - Like/unlike with animation
  - Comment system with nested replies
  - Share/repost functionality
  - Bookmark posts
  - Report/hide posts
  - Copy link

### 2. User Features
- **User Profiles**
  - Avatar and cover photo
  - Bio and links
  - Follower/following counts
  - User posts feed
  - User media gallery

- **Authentication State**
  - Logged-in view
  - Guest view (limited features)
  - User menu
  - Settings

### 3. Feed Filters
- For You (algorithm-based)
- Following
- Trending
- Bookmarks
- Mentions

### 4. UI Components
- Post card with all interaction buttons
- Comment thread
- User profile card
- Story/highlights carousel
- Trending sidebar
- Who to follow suggestions
- Loading skeletons
- Empty states
- Error boundaries

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest social-feed -- --template react-ts
cd social-feed

npm install @tanstack/react-query react-router-dom
npm install zustand
npm install framer-motion lucide-react
npm install react-lazy-load-image-component
npm install @tanstack/react-virtual
npm install date-fns
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/post.ts
export interface User {
  id: string;
  username: string;
  displayName: string;
  avatar: string;
  verified: boolean;
  bio?: string;
  followersCount: number;
  followingCount: number;
}

export interface Post {
  id: string;
  author: User;
  content: string;
  images?: string[];
  video?: string;
  createdAt: string;
  likes: number;
  comments: number;
  reposts: number;
  bookmarks: number;
  isLiked: boolean;
  isBookmarked: boolean;
  isReposted: boolean;
  repostedBy?: User;
  quotedPost?: Post;
}

export interface Comment {
  id: string;
  author: User;
  content: string;
  createdAt: string;
  likes: number;
  isLiked: boolean;
  replies: Comment[];
}

export interface PostsResponse {
  posts: Post[];
  nextCursor?: string;
  hasMore: boolean;
}
```

### Step 3: Create Infinite Query Hook
```typescript
// src/hooks/useFeed.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import type { PostsResponse } from '@/types/post';

const POSTS_PER_PAGE = 10;

async function fetchPosts(cursor?: string): Promise<PostsResponse> {
  const params = new URLSearchParams({
    limit: POSTS_PER_PAGE.toString(),
    ...(cursor && { cursor }),
  });

  const response = await fetch(`/api/posts?${params}`);
  if (!response.ok) throw new Error('Failed to fetch posts');
  return response.json();
}

export function useFeed() {
  return useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage) =>
      lastPage.hasMore ? lastPage.nextCursor : undefined,
    initialPageParam: undefined as string | undefined,
    staleTime: 1000 * 60, // 1 minute
  });
}
```

### Step 4: Create Post Interactions Store
```typescript
// src/store/interactionsStore.ts
import { create } from 'zustand';

interface InteractionsStore {
  likedPosts: Set<string>;
  bookmarkedPosts: Set<string>;
  toggleLike: (postId: string) => void;
  toggleBookmark: (postId: string) => void;
  isLiked: (postId: string) => boolean;
  isBookmarked: (postId: string) => boolean;
}

export const useInteractionsStore = create<InteractionsStore>((set, get) => ({
  likedPosts: new Set(),
  bookmarkedPosts: new Set(),

  toggleLike: (postId) => {
    set((state) => {
      const newLiked = new Set(state.likedPosts);
      if (newLiked.has(postId)) {
        newLiked.delete(postId);
      } else {
        newLiked.add(postId);
      }
      return { likedPosts: newLiked };
    });
  },

  toggleBookmark: (postId) => {
    set((state) => {
      const newBookmarked = new Set(state.bookmarkedPosts);
      if (newBookmarked.has(postId)) {
        newBookmarked.delete(postId);
      } else {
        newBookmarked.add(postId);
      }
      return { bookmarkedPosts: newBookmarked };
    });
  },

  isLiked: (postId) => get().likedPosts.has(postId),
  isBookmarked: (postId) => get().bookmarkedPosts.has(postId),
}));
```

### Step 5: Create Post Card Component
```typescript
// src/components/PostCard.tsx
import { useState } from 'react';
import { motion } from 'framer-motion';
import {
  Heart,
  MessageCircle,
  Repeat2,
  Bookmark,
  Share,
  MoreHorizontal,
} from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import type { Post } from '@/types/post';
import { useInteractionsStore } from '@/store/interactionsStore';

interface PostCardProps {
  post: Post;
  onLike: (postId: string) => void;
  onComment: (postId: string) => void;
  onRepost: (postId: string) => void;
  onBookmark: (postId: string) => void;
}

export const PostCard: React.FC<PostCardProps> = ({
  post,
  onLike,
  onComment,
  onRepost,
  onBookmark,
}) => {
  const [imageIndex, setImageIndex] = useState(0);
  const { isLiked, isBookmarked } = useInteractionsStore();

  const liked = isLiked(post.id);
  const bookmarked = isBookmarked(post.id);

  return (
    <article className="border-b border-gray-200 p-4 hover:bg-gray-50 transition-colors">
      {/* Repost indicator */}
      {post.repostedBy && (
        <div className="flex items-center gap-2 text-sm text-gray-500 mb-2">
          <Repeat2 className="w-4 h-4" />
          <span>{post.repostedBy.displayName} reposted</span>
        </div>
      )}

      <div className="flex gap-3">
        {/* Avatar */}
        <div className="flex-shrink-0">
          <img
            src={post.author.avatar}
            alt={post.author.displayName}
            className="w-12 h-12 rounded-full"
          />
        </div>

        <div className="flex-1 min-w-0">
          {/* Header */}
          <div className="flex items-start justify-between">
            <div className="flex items-center gap-2">
              <span className="font-bold hover:underline cursor-pointer">
                {post.author.displayName}
              </span>
              {post.author.verified && (
                <svg className="w-4 h-4 text-blue-500" viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M12 2L9.19 8.63L2 9.24l5.46 4.73L5.82 21L12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2z"
                  />
                </svg>
              )}
              <span className="text-gray-500">@{post.author.username}</span>
              <span className="text-gray-500">·</span>
              <span className="text-gray-500 text-sm">
                {formatDistanceToNow(new Date(post.createdAt), {
                  addSuffix: true,
                })}
              </span>
            </div>
            <button className="text-gray-500 hover:text-blue-500">
              <MoreHorizontal className="w-5 h-5" />
            </button>
          </div>

          {/* Content */}
          <p className="mt-2 text-gray-900 whitespace-pre-wrap">
            {post.content}
          </p>

          {/* Images */}
          {post.images && post.images.length > 0 && (
            <div className="mt-3 rounded-2xl overflow-hidden border border-gray-200">
              {post.images.length === 1 ? (
                <img
                  src={post.images[0]}
                  alt="Post image"
                  className="w-full max-h-96 object-cover"
                />
              ) : (
                <div className="relative">
                  <img
                    src={post.images[imageIndex]}
                    alt={`Post image ${imageIndex + 1}`}
                    className="w-full max-h-96 object-cover"
                  />
                  {post.images.length > 1 && (
                    <div className="absolute bottom-2 right-2 bg-black/70 text-white px-2 py-1 rounded text-sm">
                      {imageIndex + 1} / {post.images.length}
                    </div>
                  )}
                  {/* Navigation dots */}
                  <div className="absolute bottom-2 left-1/2 -translate-x-1/2 flex gap-2">
                    {post.images.map((_, idx) => (
                      <button
                        key={idx}
                        onClick={() => setImageIndex(idx)}
                        className={`w-2 h-2 rounded-full ${
                          idx === imageIndex ? 'bg-white' : 'bg-white/50'
                        }`}
                      />
                    ))}
                  </div>
                </div>
              )}
            </div>
          )}

          {/* Quoted post */}
          {post.quotedPost && (
            <div className="mt-3 border border-gray-200 rounded-2xl p-3">
              <div className="flex items-center gap-2 mb-1">
                <img
                  src={post.quotedPost.author.avatar}
                  alt={post.quotedPost.author.displayName}
                  className="w-5 h-5 rounded-full"
                />
                <span className="font-bold text-sm">
                  {post.quotedPost.author.displayName}
                </span>
                <span className="text-gray-500 text-sm">
                  @{post.quotedPost.author.username}
                </span>
              </div>
              <p className="text-sm text-gray-900 line-clamp-3">
                {post.quotedPost.content}
              </p>
            </div>
          )}

          {/* Interaction Buttons */}
          <div className="flex items-center justify-between mt-3 max-w-md">
            {/* Comment */}
            <button
              onClick={() => onComment(post.id)}
              className="flex items-center gap-2 text-gray-500 hover:text-blue-500 group"
            >
              <div className="p-2 rounded-full group-hover:bg-blue-50">
                <MessageCircle className="w-5 h-5" />
              </div>
              <span className="text-sm">{post.comments}</span>
            </button>

            {/* Repost */}
            <button
              onClick={() => onRepost(post.id)}
              className="flex items-center gap-2 text-gray-500 hover:text-green-500 group"
            >
              <div className="p-2 rounded-full group-hover:bg-green-50">
                <Repeat2 className="w-5 h-5" />
              </div>
              <span className="text-sm">{post.reposts}</span>
            </button>

            {/* Like */}
            <button
              onClick={() => onLike(post.id)}
              className={`flex items-center gap-2 group ${
                liked ? 'text-red-500' : 'text-gray-500 hover:text-red-500'
              }`}
            >
              <motion.div
                className="p-2 rounded-full group-hover:bg-red-50"
                whileTap={{ scale: 0.8 }}
              >
                <Heart
                  className={`w-5 h-5 ${liked ? 'fill-current' : ''}`}
                />
              </motion.div>
              <span className="text-sm">
                {post.likes + (liked ? 1 : 0)}
              </span>
            </button>

            {/* Bookmark */}
            <button
              onClick={() => onBookmark(post.id)}
              className={`group ${
                bookmarked
                  ? 'text-blue-500'
                  : 'text-gray-500 hover:text-blue-500'
              }`}
            >
              <div className="p-2 rounded-full group-hover:bg-blue-50">
                <Bookmark
                  className={`w-5 h-5 ${bookmarked ? 'fill-current' : ''}`}
                />
              </div>
            </button>

            {/* Share */}
            <button className="text-gray-500 hover:text-blue-500 group">
              <div className="p-2 rounded-full group-hover:bg-blue-50">
                <Share className="w-5 h-5" />
              </div>
            </button>
          </div>
        </div>
      </div>
    </article>
  );
};
```

### Step 6: Create Feed Component with Infinite Scroll
```typescript
// src/components/Feed.tsx
import { useEffect, useRef } from 'react';
import { useFeed } from '@/hooks/useFeed';
import { PostCard } from './PostCard';
import { useInteractionsStore } from '@/store/interactionsStore';
import { Loader2 } from 'lucide-react';

export const Feed: React.FC = () => {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    error,
  } = useFeed();

  const { toggleLike, toggleBookmark } = useInteractionsStore();
  const observerTarget = useRef<HTMLDivElement>(null);

  // Infinite scroll with Intersection Observer
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.5 }
    );

    const currentTarget = observerTarget.current;
    if (currentTarget) {
      observer.observe(currentTarget);
    }

    return () => {
      if (currentTarget) {
        observer.unobserve(currentTarget);
      }
    };
  }, [fetchNextPage, hasNextPage, isFetchingNextPage]);

  if (isLoading) {
    return (
      <div className="flex justify-center py-8">
        <Loader2 className="w-8 h-8 animate-spin text-blue-500" />
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center py-8 text-red-500">
        Error loading feed. Please try again.
      </div>
    );
  }

  const posts = data?.pages.flatMap((page) => page.posts) ?? [];

  return (
    <div className="max-w-2xl mx-auto">
      {posts.map((post) => (
        <PostCard
          key={post.id}
          post={post}
          onLike={toggleLike}
          onComment={(id) => console.log('Comment on', id)}
          onRepost={(id) => console.log('Repost', id)}
          onBookmark={toggleBookmark}
        />
      ))}

      {/* Infinite scroll trigger */}
      <div ref={observerTarget} className="py-4 flex justify-center">
        {isFetchingNextPage && (
          <Loader2 className="w-6 h-6 animate-spin text-blue-500" />
        )}
        {!hasNextPage && posts.length > 0 && (
          <p className="text-gray-500">You've reached the end!</p>
        )}
      </div>
    </div>
  );
};
```

### Step 7: Create Loading Skeleton
```typescript
// src/components/PostSkeleton.tsx
export const PostSkeleton: React.FC = () => {
  return (
    <div className="border-b border-gray-200 p-4 animate-pulse">
      <div className="flex gap-3">
        <div className="w-12 h-12 bg-gray-200 rounded-full" />
        <div className="flex-1">
          <div className="h-4 bg-gray-200 rounded w-1/3 mb-2" />
          <div className="h-4 bg-gray-200 rounded w-full mb-2" />
          <div className="h-4 bg-gray-200 rounded w-2/3 mb-3" />
          <div className="h-48 bg-gray-200 rounded-lg mb-3" />
          <div className="flex gap-8">
            <div className="h-4 bg-gray-200 rounded w-12" />
            <div className="h-4 bg-gray-200 rounded w-12" />
            <div className="h-4 bg-gray-200 rounded w-12" />
          </div>
        </div>
      </div>
    </div>
  );
};
```

## Expected Outputs

1. **Smooth Infinite Scroll** with:
   - Automatic loading as user scrolls
   - Loading indicators
   - Error handling
   - End-of-feed message

2. **Interactive Posts**:
   - Like animation with optimistic updates
   - Comment system
   - Repost functionality
   - Bookmark feature
   - Share options

3. **Performance**:
   - Smooth scrolling at 60fps
   - Optimized image loading
   - Minimal re-renders
   - Fast initial load

4. **User Experience**:
   - Loading skeletons
   - Pull-to-refresh
   - Maintained scroll position
   - Responsive design

## Bonus Challenges

- [ ] Implement virtual scrolling for better performance
- [ ] Add pull-to-refresh on mobile
- [ ] Create post composer with image upload
- [ ] Add video support with custom player
- [ ] Implement hashtag and mention parsing
- [ ] Add real-time updates with WebSocket
- [ ] Create trending topics sidebar
- [ ] Add advanced filtering options
- [ ] Implement user blocking/muting
- [ ] Add direct messaging feature
- [ ] Create notifications system
- [ ] Add keyboard shortcuts
- [ ] Implement progressive image loading
- [ ] Add accessibility features (ARIA labels, keyboard navigation)

## Resources

- [React Query Infinite Queries](https://tanstack.com/query/latest/docs/react/guides/infinite-queries)
- [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
- [Framer Motion](https://www.framer.com/motion/)
- [React Virtual](https://tanstack.com/virtual/latest)
- [Optimistic Updates Guide](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)

## Success Criteria

- Infinite scroll loads posts smoothly
- No layout shift during loading
- Interactions update instantly (optimistic UI)
- Images load progressively
- Scroll position is maintained
- Feed performs well with 1000+ posts
- Loading states are clear and helpful
- Error states are handled gracefully
- Mobile experience is smooth
- Accessibility standards are met
- TypeScript provides full type safety
- Code is well-organized and maintainable
