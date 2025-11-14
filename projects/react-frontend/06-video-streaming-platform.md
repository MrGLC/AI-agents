# Project 6: Video Streaming Platform UI with Player Controls

## Overview
Build a comprehensive video streaming platform interface with a custom video player, playlist management, video recommendations, and viewing history. This project focuses on media handling, custom player controls, and creating a user experience similar to YouTube or Netflix.

## Difficulty Level
Advanced

## Learning Objectives
- Build custom video player with HTML5 Video API
- Implement advanced player controls (playback speed, quality, subtitles)
- Create responsive video grid layouts
- Handle video state management
- Implement keyboard shortcuts
- Build picture-in-picture mode
- Create video thumbnails and previews
- Manage playlists and watch history
- Optimize video loading and buffering
- Implement theater and fullscreen modes

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Video Player**: HTML5 Video API with custom controls
- **State Management**: Zustand
- **Routing**: React Router v6
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **Storage**: localStorage for history and preferences
- **Build Tool**: Vite
- **Video CDN**: Sample videos from Pexels or Big Buck Bunny

## Project Requirements

### 1. Video Player Features
- **Core Controls**
  - Play/Pause button
  - Volume control with mute
  - Progress bar with scrubbing
  - Time display (current/duration)
  - Fullscreen toggle
  - Settings menu

- **Advanced Controls**
  - Playback speed (0.25x to 2x)
  - Quality selection (360p, 720p, 1080p)
  - Subtitles/closed captions
  - Theater mode
  - Picture-in-picture
  - Auto-play next video
  - Loop video

- **Keyboard Shortcuts**
  - Space: Play/Pause
  - Arrow keys: Skip forward/backward
  - M: Mute/Unmute
  - F: Fullscreen
  - Number keys: Jump to percentage
  - J/L: Skip 10 seconds

- **Player States**
  - Loading/buffering indicator
  - Error handling
  - End screen with recommendations
  - Thumbnail preview on hover
  - Watch time tracking

### 2. Video Browsing
- **Video Grid**
  - Responsive grid layout
  - Video thumbnails
  - Duration overlay
  - Channel avatar
  - Title and metadata
  - View count and upload date
  - Hover effects

- **Video Categories**
  - Trending
  - Recommended
  - Subscriptions
  - Watch Later
  - History
  - Liked Videos

- **Search and Filter**
  - Search videos by title
  - Filter by category
  - Sort options (newest, popular, duration)
  - Advanced filters

### 3. Video Details Page
- Video player
- Title and description
- Channel information
- Like/dislike buttons
- Share button
- Subscribe button
- View count and date
- Tags
- Related videos sidebar
- Comments section placeholder

### 4. Playlist Features
- Create playlists
- Add/remove videos
- Reorder videos
- Play all functionality
- Shuffle mode
- Playlist sharing

### 5. User Features
- Watch history
- Watch later queue
- Liked videos
- Subscribed channels
- User preferences
- Continue watching

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest video-platform -- --template react-ts
cd video-platform

npm install react-router-dom zustand
npm install lucide-react
npm install date-fns
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/video.ts
export interface Video {
  id: string;
  title: string;
  description: string;
  url: string;
  thumbnail: string;
  duration: number;
  views: number;
  likes: number;
  dislikes: number;
  uploadDate: string;
  category: string;
  tags: string[];
  channel: {
    id: string;
    name: string;
    avatar: string;
    subscribers: number;
    verified: boolean;
  };
  qualities: VideoQuality[];
  subtitles?: Subtitle[];
}

export interface VideoQuality {
  label: string;
  url: string;
  resolution: number;
}

export interface Subtitle {
  language: string;
  label: string;
  url: string;
}

export interface Playlist {
  id: string;
  name: string;
  description?: string;
  videos: string[];
  thumbnail?: string;
  createdAt: string;
  updatedAt: string;
}

export interface PlayerState {
  isPlaying: boolean;
  currentTime: number;
  duration: number;
  volume: number;
  isMuted: boolean;
  playbackRate: number;
  quality: string;
  isFullscreen: boolean;
  isTheaterMode: boolean;
  showControls: boolean;
  buffered: number;
}

export interface WatchHistory {
  videoId: string;
  watchedAt: string;
  progress: number;
}
```

### Step 3: Create Video Player Component
```typescript
// src/components/VideoPlayer.tsx
import { useRef, useState, useEffect } from 'react';
import {
  Play,
  Pause,
  Volume2,
  VolumeX,
  Maximize,
  Settings,
  Subtitles,
  PictureInPicture,
} from 'lucide-react';
import type { Video, PlayerState } from '@/types/video';

interface VideoPlayerProps {
  video: Video;
  onEnded?: () => void;
  onTimeUpdate?: (time: number) => void;
}

export const VideoPlayer: React.FC<VideoPlayerProps> = ({
  video,
  onEnded,
  onTimeUpdate,
}) => {
  const videoRef = useRef<HTMLVideoElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  const progressRef = useRef<HTMLDivElement>(null);
  const controlsTimeoutRef = useRef<NodeJS.Timeout>();

  const [playerState, setPlayerState] = useState<PlayerState>({
    isPlaying: false,
    currentTime: 0,
    duration: 0,
    volume: 1,
    isMuted: false,
    playbackRate: 1,
    quality: '720p',
    isFullscreen: false,
    isTheaterMode: false,
    showControls: true,
    buffered: 0,
  });

  const [showSettings, setShowSettings] = useState(false);

  // Play/Pause
  const togglePlay = () => {
    if (videoRef.current) {
      if (playerState.isPlaying) {
        videoRef.current.pause();
      } else {
        videoRef.current.play();
      }
      setPlayerState((prev) => ({ ...prev, isPlaying: !prev.isPlaying }));
    }
  };

  // Volume
  const handleVolumeChange = (value: number) => {
    if (videoRef.current) {
      videoRef.current.volume = value;
      setPlayerState((prev) => ({
        ...prev,
        volume: value,
        isMuted: value === 0,
      }));
    }
  };

  const toggleMute = () => {
    if (videoRef.current) {
      const newMuted = !playerState.isMuted;
      videoRef.current.muted = newMuted;
      setPlayerState((prev) => ({ ...prev, isMuted: newMuted }));
    }
  };

  // Progress
  const handleTimeUpdate = () => {
    if (videoRef.current) {
      setPlayerState((prev) => ({
        ...prev,
        currentTime: videoRef.current!.currentTime,
      }));
      onTimeUpdate?.(videoRef.current.currentTime);
    }
  };

  const handleLoadedMetadata = () => {
    if (videoRef.current) {
      setPlayerState((prev) => ({
        ...prev,
        duration: videoRef.current!.duration,
      }));
    }
  };

  const handleProgressClick = (e: React.MouseEvent<HTMLDivElement>) => {
    if (progressRef.current && videoRef.current) {
      const rect = progressRef.current.getBoundingClientRect();
      const pos = (e.clientX - rect.left) / rect.width;
      videoRef.current.currentTime = pos * playerState.duration;
    }
  };

  // Fullscreen
  const toggleFullscreen = () => {
    if (!document.fullscreenElement) {
      containerRef.current?.requestFullscreen();
      setPlayerState((prev) => ({ ...prev, isFullscreen: true }));
    } else {
      document.exitFullscreen();
      setPlayerState((prev) => ({ ...prev, isFullscreen: false }));
    }
  };

  // Playback Rate
  const changePlaybackRate = (rate: number) => {
    if (videoRef.current) {
      videoRef.current.playbackRate = rate;
      setPlayerState((prev) => ({ ...prev, playbackRate: rate }));
    }
  };

  // Picture in Picture
  const togglePiP = async () => {
    if (videoRef.current) {
      try {
        if (document.pictureInPictureElement) {
          await document.exitPictureInPicture();
        } else {
          await videoRef.current.requestPictureInPicture();
        }
      } catch (error) {
        console.error('PiP error:', error);
      }
    }
  };

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyPress = (e: KeyboardEvent) => {
      switch (e.key) {
        case ' ':
          e.preventDefault();
          togglePlay();
          break;
        case 'ArrowLeft':
          if (videoRef.current) {
            videoRef.current.currentTime -= 5;
          }
          break;
        case 'ArrowRight':
          if (videoRef.current) {
            videoRef.current.currentTime += 5;
          }
          break;
        case 'ArrowUp':
          e.preventDefault();
          handleVolumeChange(Math.min(1, playerState.volume + 0.1));
          break;
        case 'ArrowDown':
          e.preventDefault();
          handleVolumeChange(Math.max(0, playerState.volume - 0.1));
          break;
        case 'm':
          toggleMute();
          break;
        case 'f':
          toggleFullscreen();
          break;
      }
    };

    window.addEventListener('keydown', handleKeyPress);
    return () => window.removeEventListener('keydown', handleKeyPress);
  }, [playerState.volume, playerState.isPlaying]);

  // Auto-hide controls
  const handleMouseMove = () => {
    setPlayerState((prev) => ({ ...prev, showControls: true }));

    if (controlsTimeoutRef.current) {
      clearTimeout(controlsTimeoutRef.current);
    }

    if (playerState.isPlaying) {
      controlsTimeoutRef.current = setTimeout(() => {
        setPlayerState((prev) => ({ ...prev, showControls: false }));
      }, 3000);
    }
  };

  const formatTime = (seconds: number): string => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = Math.floor(seconds % 60);

    if (h > 0) {
      return `${h}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
    }
    return `${m}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <div
      ref={containerRef}
      className="relative bg-black group"
      onMouseMove={handleMouseMove}
      onMouseLeave={() =>
        playerState.isPlaying &&
        setPlayerState((prev) => ({ ...prev, showControls: false }))
      }
    >
      {/* Video Element */}
      <video
        ref={videoRef}
        src={video.url}
        className="w-full aspect-video"
        onTimeUpdate={handleTimeUpdate}
        onLoadedMetadata={handleLoadedMetadata}
        onEnded={onEnded}
        onClick={togglePlay}
      />

      {/* Controls Overlay */}
      <div
        className={`absolute inset-0 bg-gradient-to-t from-black/80 via-transparent to-transparent transition-opacity ${
          playerState.showControls ? 'opacity-100' : 'opacity-0'
        }`}
      >
        {/* Center Play Button */}
        {!playerState.isPlaying && (
          <button
            onClick={togglePlay}
            className="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-blue-600 rounded-full p-6 hover:bg-blue-700 transition-colors"
          >
            <Play className="w-12 h-12 text-white" fill="white" />
          </button>
        )}

        {/* Bottom Controls */}
        <div className="absolute bottom-0 left-0 right-0 p-4 space-y-2">
          {/* Progress Bar */}
          <div
            ref={progressRef}
            className="h-1 bg-gray-600 rounded-full cursor-pointer group/progress"
            onClick={handleProgressClick}
          >
            <div
              className="h-full bg-blue-600 rounded-full relative group-hover/progress:h-1.5 transition-all"
              style={{
                width: `${(playerState.currentTime / playerState.duration) * 100}%`,
              }}
            >
              <div className="absolute right-0 top-1/2 -translate-y-1/2 w-3 h-3 bg-blue-600 rounded-full opacity-0 group-hover/progress:opacity-100" />
            </div>
          </div>

          {/* Control Buttons */}
          <div className="flex items-center justify-between text-white">
            <div className="flex items-center gap-4">
              {/* Play/Pause */}
              <button onClick={togglePlay} className="hover:text-blue-400">
                {playerState.isPlaying ? (
                  <Pause className="w-6 h-6" />
                ) : (
                  <Play className="w-6 h-6" />
                )}
              </button>

              {/* Volume */}
              <div className="flex items-center gap-2 group/volume">
                <button onClick={toggleMute} className="hover:text-blue-400">
                  {playerState.isMuted || playerState.volume === 0 ? (
                    <VolumeX className="w-6 h-6" />
                  ) : (
                    <Volume2 className="w-6 h-6" />
                  )}
                </button>
                <input
                  type="range"
                  min="0"
                  max="1"
                  step="0.1"
                  value={playerState.isMuted ? 0 : playerState.volume}
                  onChange={(e) => handleVolumeChange(parseFloat(e.target.value))}
                  className="w-0 group-hover/volume:w-20 transition-all"
                />
              </div>

              {/* Time */}
              <span className="text-sm">
                {formatTime(playerState.currentTime)} /{' '}
                {formatTime(playerState.duration)}
              </span>
            </div>

            <div className="flex items-center gap-4">
              {/* Playback Speed */}
              <div className="relative">
                <button
                  onClick={() => setShowSettings(!showSettings)}
                  className="hover:text-blue-400 text-sm"
                >
                  {playerState.playbackRate}x
                </button>
                {showSettings && (
                  <div className="absolute bottom-full right-0 mb-2 bg-gray-900 rounded-lg p-2 min-w-[120px]">
                    {[0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2].map((rate) => (
                      <button
                        key={rate}
                        onClick={() => {
                          changePlaybackRate(rate);
                          setShowSettings(false);
                        }}
                        className={`block w-full text-left px-3 py-2 hover:bg-gray-800 rounded ${
                          playerState.playbackRate === rate ? 'text-blue-400' : ''
                        }`}
                      >
                        {rate}x
                      </button>
                    ))}
                  </div>
                )}
              </div>

              {/* Settings */}
              <button className="hover:text-blue-400">
                <Settings className="w-6 h-6" />
              </button>

              {/* Subtitles */}
              {video.subtitles && (
                <button className="hover:text-blue-400">
                  <Subtitles className="w-6 h-6" />
                </button>
              )}

              {/* Picture in Picture */}
              <button onClick={togglePiP} className="hover:text-blue-400">
                <PictureInPicture className="w-6 h-6" />
              </button>

              {/* Fullscreen */}
              <button onClick={toggleFullscreen} className="hover:text-blue-400">
                <Maximize className="w-6 h-6" />
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};
```

### Step 4: Create Video Card Component
```typescript
// src/components/VideoCard.tsx
import { formatDistanceToNow } from 'date-fns';
import { MoreVertical, Clock } from 'lucide-react';
import { Link } from 'react-router-dom';
import type { Video } from '@/types/video';

interface VideoCardProps {
  video: Video;
}

export const VideoCard: React.FC<VideoCardProps> = ({ video }) => {
  const formatViews = (views: number): string => {
    if (views >= 1000000) {
      return `${(views / 1000000).toFixed(1)}M`;
    }
    if (views >= 1000) {
      return `${(views / 1000).toFixed(1)}K`;
    }
    return views.toString();
  };

  const formatDuration = (seconds: number): string => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = Math.floor(seconds % 60);

    if (h > 0) {
      return `${h}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
    }
    return `${m}:${s.toString().padStart(2, '0')}`;
  };

  return (
    <Link to={`/video/${video.id}`} className="group">
      <div className="flex flex-col">
        {/* Thumbnail */}
        <div className="relative aspect-video bg-gray-200 rounded-lg overflow-hidden">
          <img
            src={video.thumbnail}
            alt={video.title}
            className="w-full h-full object-cover group-hover:scale-105 transition-transform"
          />
          <div className="absolute bottom-2 right-2 bg-black/80 text-white px-2 py-1 rounded text-xs font-semibold">
            {formatDuration(video.duration)}
          </div>
        </div>

        {/* Info */}
        <div className="flex gap-3 mt-3">
          <img
            src={video.channel.avatar}
            alt={video.channel.name}
            className="w-9 h-9 rounded-full flex-shrink-0"
          />

          <div className="flex-1 min-w-0">
            <h3 className="font-semibold line-clamp-2 group-hover:text-blue-600">
              {video.title}
            </h3>

            <div className="flex items-center gap-1 mt-1 text-sm text-gray-600">
              <span>{video.channel.name}</span>
              {video.channel.verified && (
                <svg className="w-3 h-3" viewBox="0 0 24 24">
                  <path
                    fill="currentColor"
                    d="M12 2L9.19 8.63L2 9.24l5.46 4.73L5.82 21L12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2z"
                  />
                </svg>
              )}
            </div>

            <div className="text-sm text-gray-600 mt-1">
              <span>{formatViews(video.views)} views</span>
              <span className="mx-1">•</span>
              <span>
                {formatDistanceToNow(new Date(video.uploadDate), {
                  addSuffix: true,
                })}
              </span>
            </div>
          </div>

          <button className="text-gray-600 hover:text-gray-900 flex-shrink-0">
            <MoreVertical className="w-5 h-5" />
          </button>
        </div>
      </div>
    </Link>
  );
};
```

### Step 5: Create Video Page
```typescript
// src/pages/VideoPage.tsx
import { useParams } from 'react-router-dom';
import { useState } from 'react';
import { ThumbsUp, ThumbsDown, Share2, Save, MoreHorizontal } from 'lucide-react';
import { VideoPlayer } from '@/components/VideoPlayer';
import { VideoCard } from '@/components/VideoCard';
import { format } from 'date-fns';

// Mock data - replace with actual API call
const mockVideo = {
  id: '1',
  title: 'Building a Video Player in React',
  description: 'Learn how to build a custom video player with React...',
  url: 'https://example.com/video.mp4',
  thumbnail: 'https://picsum.photos/1280/720',
  duration: 600,
  views: 125000,
  likes: 5200,
  dislikes: 150,
  uploadDate: '2024-01-15',
  category: 'Tutorial',
  tags: ['react', 'javascript', 'tutorial'],
  channel: {
    id: 'ch1',
    name: 'Code Academy',
    avatar: 'https://picsum.photos/200',
    subscribers: 250000,
    verified: true,
  },
  qualities: [
    { label: '1080p', url: '', resolution: 1080 },
    { label: '720p', url: '', resolution: 720 },
  ],
};

const mockRelatedVideos = Array(8).fill(mockVideo).map((v, i) => ({
  ...v,
  id: `related-${i}`,
  title: `Related Video ${i + 1}`,
}));

export const VideoPage: React.FC = () => {
  const { id } = useParams();
  const [isSubscribed, setIsSubscribed] = useState(false);
  const [isLiked, setIsLiked] = useState(false);
  const [isDisliked, setIsDisliked] = useState(false);
  const [showFullDescription, setShowFullDescription] = useState(false);

  const formatNumber = (num: number): string => {
    if (num >= 1000000) return `${(num / 1000000).toFixed(1)}M`;
    if (num >= 1000) return `${(num / 1000).toFixed(1)}K`;
    return num.toString();
  };

  return (
    <div className="max-w-[1800px] mx-auto px-4 py-6">
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Main Content */}
        <div className="lg:col-span-2">
          {/* Video Player */}
          <VideoPlayer video={mockVideo} />

          {/* Video Info */}
          <div className="mt-4">
            <h1 className="text-2xl font-bold">{mockVideo.title}</h1>

            {/* Stats and Actions */}
            <div className="flex items-center justify-between mt-4 pb-4 border-b">
              <div className="flex items-center gap-4">
                <img
                  src={mockVideo.channel.avatar}
                  alt={mockVideo.channel.name}
                  className="w-12 h-12 rounded-full"
                />
                <div>
                  <div className="flex items-center gap-2">
                    <h3 className="font-semibold">{mockVideo.channel.name}</h3>
                    {mockVideo.channel.verified && (
                      <svg className="w-4 h-4" viewBox="0 0 24 24">
                        <path
                          fill="currentColor"
                          d="M12 2L9.19 8.63L2 9.24l5.46 4.73L5.82 21L12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2z"
                        />
                      </svg>
                    )}
                  </div>
                  <p className="text-sm text-gray-600">
                    {formatNumber(mockVideo.channel.subscribers)} subscribers
                  </p>
                </div>
                <button
                  onClick={() => setIsSubscribed(!isSubscribed)}
                  className={`ml-4 px-6 py-2 rounded-full font-semibold ${
                    isSubscribed
                      ? 'bg-gray-200 text-gray-900'
                      : 'bg-red-600 text-white hover:bg-red-700'
                  }`}
                >
                  {isSubscribed ? 'Subscribed' : 'Subscribe'}
                </button>
              </div>

              <div className="flex items-center gap-2">
                <div className="flex bg-gray-100 rounded-full">
                  <button
                    onClick={() => {
                      setIsLiked(!isLiked);
                      if (isDisliked) setIsDisliked(false);
                    }}
                    className={`flex items-center gap-2 px-4 py-2 rounded-l-full ${
                      isLiked ? 'text-blue-600' : ''
                    }`}
                  >
                    <ThumbsUp className="w-5 h-5" fill={isLiked ? 'currentColor' : 'none'} />
                    <span>{formatNumber(mockVideo.likes + (isLiked ? 1 : 0))}</span>
                  </button>
                  <div className="w-px bg-gray-300" />
                  <button
                    onClick={() => {
                      setIsDisliked(!isDisliked);
                      if (isLiked) setIsLiked(false);
                    }}
                    className={`px-4 py-2 rounded-r-full ${
                      isDisliked ? 'text-blue-600' : ''
                    }`}
                  >
                    <ThumbsDown className="w-5 h-5" fill={isDisliked ? 'currentColor' : 'none'} />
                  </button>
                </div>

                <button className="flex items-center gap-2 bg-gray-100 px-4 py-2 rounded-full hover:bg-gray-200">
                  <Share2 className="w-5 h-5" />
                  <span>Share</span>
                </button>

                <button className="flex items-center gap-2 bg-gray-100 px-4 py-2 rounded-full hover:bg-gray-200">
                  <Save className="w-5 h-5" />
                  <span>Save</span>
                </button>

                <button className="bg-gray-100 p-2 rounded-full hover:bg-gray-200">
                  <MoreHorizontal className="w-5 h-5" />
                </button>
              </div>
            </div>

            {/* Description */}
            <div className="mt-4 bg-gray-100 rounded-lg p-4">
              <div className="flex items-center gap-4 text-sm font-semibold mb-2">
                <span>{formatNumber(mockVideo.views)} views</span>
                <span>{format(new Date(mockVideo.uploadDate), 'MMM dd, yyyy')}</span>
                {mockVideo.tags.map((tag) => (
                  <span key={tag} className="text-blue-600">
                    #{tag}
                  </span>
                ))}
              </div>
              <p className={showFullDescription ? '' : 'line-clamp-3'}>
                {mockVideo.description}
              </p>
              <button
                onClick={() => setShowFullDescription(!showFullDescription)}
                className="text-sm font-semibold mt-2"
              >
                {showFullDescription ? 'Show less' : 'Show more'}
              </button>
            </div>
          </div>
        </div>

        {/* Sidebar - Related Videos */}
        <div className="space-y-3">
          {mockRelatedVideos.map((video) => (
            <VideoCard key={video.id} video={video} />
          ))}
        </div>
      </div>
    </div>
  );
};
```

## Expected Outputs

1. **Fully Functional Video Player** with:
   - Custom controls with modern UI
   - Keyboard shortcuts
   - Picture-in-picture support
   - Fullscreen and theater modes
   - Playback speed control

2. **Video Browsing**:
   - Responsive video grid
   - Video thumbnails with duration
   - Channel information
   - View counts and dates

3. **Video Details**:
   - Complete video page
   - Like/dislike functionality
   - Subscribe button
   - Share and save options
   - Related videos

4. **User Experience**:
   - Smooth playback
   - Auto-hiding controls
   - Progress tracking
   - Watch history
   - Responsive design

## Bonus Challenges

- [ ] Add comments section with replies
- [ ] Implement video upload functionality
- [ ] Add live streaming support
- [ ] Create channel pages
- [ ] Add video analytics dashboard
- [ ] Implement video chapters/timestamps
- [ ] Add mini-player (sticky player)
- [ ] Create video playlist editor
- [ ] Add subtitle editor
- [ ] Implement video trimming/editing
- [ ] Add casting support (Chromecast)
- [ ] Create watch party feature
- [ ] Add video quality auto-switching
- [ ] Implement content recommendations algorithm

## Resources

- [HTML5 Video API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement)
- [Picture-in-Picture API](https://developer.mozilla.org/en-US/docs/Web/API/Picture-in-Picture_API)
- [Fullscreen API](https://developer.mozilla.org/en-US/docs/Web/API/Fullscreen_API)
- [Media Session API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Session_API)
- [Web VTT Subtitles](https://developer.mozilla.org/en-US/docs/Web/API/WebVTT_API)

## Success Criteria

- Video plays smoothly without buffering issues
- All player controls work correctly
- Keyboard shortcuts function as expected
- Fullscreen and PiP modes work properly
- Progress bar accurately shows position
- Volume controls respond smoothly
- Settings menu displays correctly
- Video grid is responsive
- Watch history is tracked accurately
- UI is intuitive and accessible
- TypeScript provides full type coverage
- Performance is optimized (60fps playback)
