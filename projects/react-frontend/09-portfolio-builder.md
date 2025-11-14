# Project 9: Portfolio Builder with Live Preview

## Overview
Build an interactive portfolio builder that allows users to create, customize, and preview their personal portfolio websites in real-time. This project focuses on dynamic content editing, live preview synchronization, template management, and export functionality similar to Wix or Webflow for portfolios.

## Difficulty Level
Advanced

## Learning Objectives
- Implement real-time preview synchronization
- Build drag-and-drop page builder
- Create customizable component system
- Handle dynamic theming and styling
- Implement template system
- Build form builders with validation
- Create export/deployment functionality
- Manage complex nested state
- Implement iframe communication
- Build responsive preview modes

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **State Management**: Zustand with persistence
- **Styling**: Tailwind CSS with dynamic theming
- **Drag & Drop**: @dnd-kit/core
- **Code Editor**: Monaco Editor (VS Code editor)
- **Color Picker**: react-colorful
- **Icons**: Lucide React
- **Export**: html-to-image or html2canvas
- **Build Tool**: Vite
- **Preview**: Sandpack or custom iframe

## Project Requirements

### 1. Portfolio Builder Features
- **Page Management**
  - Create multiple pages (Home, About, Projects, Contact)
  - Reorder pages
  - Set home page
  - Delete pages
  - Duplicate pages

- **Section Builder**
  - Hero section
  - About section
  - Skills section
  - Projects/Portfolio gallery
  - Experience/Timeline
  - Testimonials
  - Contact form
  - Custom sections

- **Component Library**
  - Headings (H1-H6)
  - Paragraphs
  - Images
  - Buttons
  - Links
  - Icons
  - Dividers
  - Spacers
  - Cards
  - Grids

### 2. Customization Features
- **Theme Customization**
  - Color scheme picker
  - Font family selection
  - Font size controls
  - Spacing controls
  - Border radius
  - Shadow intensity
  - Background patterns

- **Component Editing**
  - Text editing inline
  - Image upload/URL
  - Link destinations
  - Button styles
  - Alignment options
  - Visibility toggles
  - Animation effects

- **Layout Controls**
  - Container width
  - Section spacing
  - Grid columns
  - Flex alignment
  - Responsive breakpoints

### 3. Content Management
- **Personal Information**
  - Name and title
  - Profile photo
  - Bio/description
  - Contact information
  - Social media links
  - Resume/CV upload

- **Projects**
  - Project title
  - Description
  - Technologies used
  - Images/screenshots
  - Live demo link
  - GitHub repository
  - Featured projects

- **Skills**
  - Skill name
  - Proficiency level
  - Categories
  - Icons

- **Experience**
  - Job title
  - Company
  - Duration
  - Description
  - Technologies

### 4. Live Preview
- **Preview Modes**
  - Desktop view
  - Tablet view
  - Mobile view
  - Full-screen preview
  - Side-by-side editing

- **Preview Features**
  - Real-time updates
  - Interactive elements
  - Scroll synchronization
  - Device frames
  - Responsive testing

### 5. Export & Share
- Export as HTML/CSS/JS
- Download as ZIP
- Copy code to clipboard
- Deploy to Netlify/Vercel
- Share preview link
- Generate PDF resume

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest portfolio-builder -- --template react-ts
cd portfolio-builder

npm install zustand @dnd-kit/core @dnd-kit/sortable
npm install @monaco-editor/react
npm install react-colorful
npm install lucide-react
npm install html-to-image
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/portfolio.ts
export interface Portfolio {
  id: string;
  name: string;
  pages: Page[];
  theme: Theme;
  personalInfo: PersonalInfo;
  projects: Project[];
  skills: Skill[];
  experience: Experience[];
  createdAt: string;
  updatedAt: string;
}

export interface Page {
  id: string;
  title: string;
  slug: string;
  sections: Section[];
  isHome: boolean;
  order: number;
}

export interface Section {
  id: string;
  type: SectionType;
  components: Component[];
  settings: SectionSettings;
  order: number;
}

export type SectionType =
  | 'hero'
  | 'about'
  | 'skills'
  | 'projects'
  | 'experience'
  | 'testimonials'
  | 'contact'
  | 'custom';

export interface SectionSettings {
  backgroundColor?: string;
  backgroundImage?: string;
  padding?: string;
  margin?: string;
  textAlign?: 'left' | 'center' | 'right';
  containerWidth?: 'full' | 'container' | 'narrow';
}

export interface Component {
  id: string;
  type: ComponentType;
  content: any;
  styles?: ComponentStyles;
  order: number;
}

export type ComponentType =
  | 'heading'
  | 'text'
  | 'image'
  | 'button'
  | 'link'
  | 'icon'
  | 'divider'
  | 'spacer'
  | 'card'
  | 'grid';

export interface ComponentStyles {
  color?: string;
  fontSize?: string;
  fontWeight?: string;
  textAlign?: 'left' | 'center' | 'right';
  margin?: string;
  padding?: string;
  borderRadius?: string;
  backgroundColor?: string;
  border?: string;
  boxShadow?: string;
}

export interface Theme {
  name: string;
  colors: {
    primary: string;
    secondary: string;
    accent: string;
    background: string;
    text: string;
    muted: string;
  };
  fonts: {
    heading: string;
    body: string;
  };
  borderRadius: string;
  spacing: {
    section: string;
    component: string;
  };
}

export interface PersonalInfo {
  name: string;
  title: string;
  email: string;
  phone?: string;
  location?: string;
  bio: string;
  avatar?: string;
  resume?: string;
  socialLinks: SocialLink[];
}

export interface SocialLink {
  platform: string;
  url: string;
  icon: string;
}

export interface Project {
  id: string;
  title: string;
  description: string;
  image?: string;
  images?: string[];
  technologies: string[];
  liveUrl?: string;
  githubUrl?: string;
  featured: boolean;
  order: number;
}

export interface Skill {
  id: string;
  name: string;
  level: number;
  category: string;
  icon?: string;
}

export interface Experience {
  id: string;
  title: string;
  company: string;
  startDate: string;
  endDate?: string;
  current: boolean;
  description: string;
  technologies: string[];
}

export type PreviewMode = 'desktop' | 'tablet' | 'mobile';
```

### Step 3: Create Portfolio Store
```typescript
// src/store/portfolioStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';
import type { Portfolio, Page, Section, Component, Theme, PreviewMode } from '@/types/portfolio';

interface PortfolioState {
  portfolio: Portfolio | null;
  currentPage: string | null;
  selectedSection: string | null;
  selectedComponent: string | null;
  previewMode: PreviewMode;
  isPreviewMode: boolean;

  // Portfolio actions
  createPortfolio: (name: string) => void;
  updateTheme: (theme: Partial<Theme>) => void;
  updatePersonalInfo: (info: Partial<PersonalInfo>) => void;

  // Page actions
  addPage: (title: string) => void;
  updatePage: (pageId: string, updates: Partial<Page>) => void;
  deletePage: (pageId: string) => void;
  setCurrentPage: (pageId: string) => void;

  // Section actions
  addSection: (pageId: string, type: SectionType) => void;
  updateSection: (sectionId: string, updates: Partial<Section>) => void;
  deleteSection: (sectionId: string) => void;
  moveSection: (sectionId: string, direction: 'up' | 'down') => void;

  // Component actions
  addComponent: (sectionId: string, type: ComponentType) => void;
  updateComponent: (componentId: string, updates: Partial<Component>) => void;
  deleteComponent: (componentId: string) => void;

  // Selection
  selectSection: (sectionId: string | null) => void;
  selectComponent: (componentId: string | null) => void;

  // Preview
  setPreviewMode: (mode: PreviewMode) => void;
  togglePreviewMode: () => void;

  // Projects
  addProject: (project: Omit<Project, 'id' | 'order'>) => void;
  updateProject: (projectId: string, updates: Partial<Project>) => void;
  deleteProject: (projectId: string) => void;

  // Skills
  addSkill: (skill: Omit<Skill, 'id'>) => void;
  updateSkill: (skillId: string, updates: Partial<Skill>) => void;
  deleteSkill: (skillId: string) => void;
}

const defaultTheme: Theme = {
  name: 'Default',
  colors: {
    primary: '#3b82f6',
    secondary: '#8b5cf6',
    accent: '#f59e0b',
    background: '#ffffff',
    text: '#1f2937',
    muted: '#6b7280',
  },
  fonts: {
    heading: 'Inter, sans-serif',
    body: 'Inter, sans-serif',
  },
  borderRadius: '0.5rem',
  spacing: {
    section: '4rem',
    component: '1rem',
  },
};

export const usePortfolioStore = create<PortfolioState>()(
  persist(
    immer((set, get) => ({
      portfolio: null,
      currentPage: null,
      selectedSection: null,
      selectedComponent: null,
      previewMode: 'desktop',
      isPreviewMode: false,

      createPortfolio: (name) => {
        const newPortfolio: Portfolio = {
          id: Date.now().toString(),
          name,
          pages: [
            {
              id: 'home',
              title: 'Home',
              slug: 'home',
              sections: [],
              isHome: true,
              order: 0,
            },
          ],
          theme: defaultTheme,
          personalInfo: {
            name: '',
            title: '',
            email: '',
            bio: '',
            socialLinks: [],
          },
          projects: [],
          skills: [],
          experience: [],
          createdAt: new Date().toISOString(),
          updatedAt: new Date().toISOString(),
        };
        set({ portfolio: newPortfolio, currentPage: 'home' });
      },

      updateTheme: (themeUpdates) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.theme = {
              ...state.portfolio.theme,
              ...themeUpdates,
            };
            state.portfolio.updatedAt = new Date().toISOString();
          }
        });
      },

      updatePersonalInfo: (info) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.personalInfo = {
              ...state.portfolio.personalInfo,
              ...info,
            };
            state.portfolio.updatedAt = new Date().toISOString();
          }
        });
      },

      addPage: (title) => {
        set((state) => {
          if (state.portfolio) {
            const newPage: Page = {
              id: Date.now().toString(),
              title,
              slug: title.toLowerCase().replace(/\s+/g, '-'),
              sections: [],
              isHome: false,
              order: state.portfolio.pages.length,
            };
            state.portfolio.pages.push(newPage);
            state.currentPage = newPage.id;
          }
        });
      },

      updatePage: (pageId, updates) => {
        set((state) => {
          if (state.portfolio) {
            const page = state.portfolio.pages.find((p) => p.id === pageId);
            if (page) {
              Object.assign(page, updates);
            }
          }
        });
      },

      deletePage: (pageId) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.pages = state.portfolio.pages.filter(
              (p) => p.id !== pageId
            );
          }
        });
      },

      setCurrentPage: (pageId) => {
        set({ currentPage: pageId });
      },

      addSection: (pageId, type) => {
        set((state) => {
          if (state.portfolio) {
            const page = state.portfolio.pages.find((p) => p.id === pageId);
            if (page) {
              const newSection: Section = {
                id: Date.now().toString(),
                type,
                components: [],
                settings: {},
                order: page.sections.length,
              };
              page.sections.push(newSection);
            }
          }
        });
      },

      updateSection: (sectionId, updates) => {
        set((state) => {
          if (state.portfolio) {
            for (const page of state.portfolio.pages) {
              const section = page.sections.find((s) => s.id === sectionId);
              if (section) {
                Object.assign(section, updates);
                break;
              }
            }
          }
        });
      },

      deleteSection: (sectionId) => {
        set((state) => {
          if (state.portfolio) {
            for (const page of state.portfolio.pages) {
              page.sections = page.sections.filter((s) => s.id !== sectionId);
            }
          }
        });
      },

      moveSection: (sectionId, direction) => {
        // Implementation for moving sections up/down
      },

      addComponent: (sectionId, type) => {
        set((state) => {
          if (state.portfolio) {
            for (const page of state.portfolio.pages) {
              const section = page.sections.find((s) => s.id === sectionId);
              if (section) {
                const newComponent: Component = {
                  id: Date.now().toString(),
                  type,
                  content: getDefaultContent(type),
                  order: section.components.length,
                };
                section.components.push(newComponent);
                break;
              }
            }
          }
        });
      },

      updateComponent: (componentId, updates) => {
        set((state) => {
          if (state.portfolio) {
            for (const page of state.portfolio.pages) {
              for (const section of page.sections) {
                const component = section.components.find(
                  (c) => c.id === componentId
                );
                if (component) {
                  Object.assign(component, updates);
                  return;
                }
              }
            }
          }
        });
      },

      deleteComponent: (componentId) => {
        set((state) => {
          if (state.portfolio) {
            for (const page of state.portfolio.pages) {
              for (const section of page.sections) {
                section.components = section.components.filter(
                  (c) => c.id !== componentId
                );
              }
            }
          }
        });
      },

      selectSection: (sectionId) => {
        set({ selectedSection: sectionId, selectedComponent: null });
      },

      selectComponent: (componentId) => {
        set({ selectedComponent: componentId });
      },

      setPreviewMode: (mode) => {
        set({ previewMode: mode });
      },

      togglePreviewMode: () => {
        set((state) => ({ isPreviewMode: !state.isPreviewMode }));
      },

      addProject: (project) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.projects.push({
              ...project,
              id: Date.now().toString(),
              order: state.portfolio.projects.length,
            });
          }
        });
      },

      updateProject: (projectId, updates) => {
        set((state) => {
          if (state.portfolio) {
            const project = state.portfolio.projects.find(
              (p) => p.id === projectId
            );
            if (project) {
              Object.assign(project, updates);
            }
          }
        });
      },

      deleteProject: (projectId) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.projects = state.portfolio.projects.filter(
              (p) => p.id !== projectId
            );
          }
        });
      },

      addSkill: (skill) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.skills.push({
              ...skill,
              id: Date.now().toString(),
            });
          }
        });
      },

      updateSkill: (skillId, updates) => {
        set((state) => {
          if (state.portfolio) {
            const skill = state.portfolio.skills.find((s) => s.id === skillId);
            if (skill) {
              Object.assign(skill, updates);
            }
          }
        });
      },

      deleteSkill: (skillId) => {
        set((state) => {
          if (state.portfolio) {
            state.portfolio.skills = state.portfolio.skills.filter(
              (s) => s.id !== skillId
            );
          }
        });
      },
    })),
    {
      name: 'portfolio-storage',
    }
  )
);

function getDefaultContent(type: ComponentType): any {
  switch (type) {
    case 'heading':
      return { text: 'Heading', level: 2 };
    case 'text':
      return { text: 'Enter your text here...' };
    case 'image':
      return { src: '', alt: '' };
    case 'button':
      return { text: 'Button', url: '#' };
    default:
      return {};
  }
}
```

### Step 4: Create Live Preview Component
```typescript
// src/components/LivePreview.tsx
import { useEffect, useRef } from 'react';
import { usePortfolioStore } from '@/store/portfolioStore';
import { Monitor, Tablet, Smartphone } from 'lucide-react';

export const LivePreview: React.FC = () => {
  const { portfolio, previewMode, setPreviewMode, currentPage } =
    usePortfolioStore();
  const iframeRef = useRef<HTMLIFrameElement>(null);

  const previewWidths = {
    desktop: '100%',
    tablet: '768px',
    mobile: '375px',
  };

  useEffect(() => {
    if (iframeRef.current && portfolio) {
      const html = generateHTML(portfolio, currentPage || '');
      const iframeDoc = iframeRef.current.contentDocument;
      if (iframeDoc) {
        iframeDoc.open();
        iframeDoc.write(html);
        iframeDoc.close();
      }
    }
  }, [portfolio, currentPage]);

  return (
    <div className="flex-1 bg-gray-100 flex flex-col">
      {/* Preview Toolbar */}
      <div className="bg-white border-b px-4 py-3 flex items-center justify-between">
        <div className="flex items-center gap-2">
          <button
            onClick={() => setPreviewMode('desktop')}
            className={`p-2 rounded ${
              previewMode === 'desktop'
                ? 'bg-blue-100 text-blue-600'
                : 'hover:bg-gray-100'
            }`}
          >
            <Monitor className="w-5 h-5" />
          </button>
          <button
            onClick={() => setPreviewMode('tablet')}
            className={`p-2 rounded ${
              previewMode === 'tablet'
                ? 'bg-blue-100 text-blue-600'
                : 'hover:bg-gray-100'
            }`}
          >
            <Tablet className="w-5 h-5" />
          </button>
          <button
            onClick={() => setPreviewMode('mobile')}
            className={`p-2 rounded ${
              previewMode === 'mobile'
                ? 'bg-blue-100 text-blue-600'
                : 'hover:bg-gray-100'
            }`}
          >
            <Smartphone className="w-5 h-5" />
          </button>
        </div>

        <div className="text-sm text-gray-600">
          {previewMode.charAt(0).toUpperCase() + previewMode.slice(1)} View
        </div>
      </div>

      {/* Preview Frame */}
      <div className="flex-1 flex items-center justify-center p-8 overflow-auto">
        <div
          className="bg-white shadow-2xl transition-all duration-300"
          style={{
            width: previewWidths[previewMode],
            height: '100%',
            maxWidth: '100%',
          }}
        >
          <iframe
            ref={iframeRef}
            title="Portfolio Preview"
            className="w-full h-full"
            sandbox="allow-scripts"
          />
        </div>
      </div>
    </div>
  );
};

function generateHTML(portfolio: Portfolio, pageId: string): string {
  const page = portfolio.pages.find((p) => p.id === pageId);
  if (!page) return '';

  const { theme, personalInfo } = portfolio;

  return `
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>${personalInfo.name} - ${page.title}</title>
      <script src="https://cdn.tailwindcss.com"></script>
      <style>
        :root {
          --color-primary: ${theme.colors.primary};
          --color-secondary: ${theme.colors.secondary};
          --color-accent: ${theme.colors.accent};
          --color-background: ${theme.colors.background};
          --color-text: ${theme.colors.text};
        }
        body {
          font-family: ${theme.fonts.body};
          color: var(--color-text);
          background-color: var(--color-background);
        }
        h1, h2, h3, h4, h5, h6 {
          font-family: ${theme.fonts.heading};
        }
      </style>
    </head>
    <body>
      ${renderPage(page, portfolio)}
    </body>
    </html>
  `;
}

function renderPage(page: Page, portfolio: Portfolio): string {
  return page.sections.map((section) => renderSection(section, portfolio)).join('');
}

function renderSection(section: Section, portfolio: Portfolio): string {
  const { settings } = section;
  const style = `
    background-color: ${settings.backgroundColor || 'transparent'};
    padding: ${settings.padding || '4rem 0'};
    text-align: ${settings.textAlign || 'left'};
  `;

  const containerClass =
    settings.containerWidth === 'full'
      ? 'w-full'
      : settings.containerWidth === 'narrow'
      ? 'max-w-4xl mx-auto px-4'
      : 'max-w-7xl mx-auto px-4';

  return `
    <section style="${style}">
      <div class="${containerClass}">
        ${section.components.map((comp) => renderComponent(comp)).join('')}
      </div>
    </section>
  `;
}

function renderComponent(component: Component): string {
  switch (component.type) {
    case 'heading':
      const level = component.content.level || 2;
      return `<h${level} class="text-${level === 1 ? '5xl' : level === 2 ? '4xl' : '3xl'} font-bold mb-4">${
        component.content.text
      }</h${level}>`;

    case 'text':
      return `<p class="text-lg mb-4">${component.content.text}</p>`;

    case 'image':
      return `<img src="${component.content.src}" alt="${component.content.alt}" class="w-full rounded-lg mb-4" />`;

    case 'button':
      return `<a href="${component.content.url}" class="inline-block bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700">${component.content.text}</a>`;

    default:
      return '';
  }
}
```

### Step 5: Create Editor Sidebar
```typescript
// src/components/EditorSidebar.tsx
import { useState } from 'react';
import { Plus, Settings, Palette, FileText, Briefcase } from 'lucide-react';
import { usePortfolioStore } from '@/store/portfolioStore';

export const EditorSidebar: React.FC = () => {
  const [activeTab, setActiveTab] = useState<'sections' | 'theme' | 'content'>('sections');
  const { addSection, currentPage } = usePortfolioStore();

  const sectionTypes = [
    { type: 'hero', name: 'Hero', icon: '🎯' },
    { type: 'about', name: 'About', icon: '👤' },
    { type: 'skills', name: 'Skills', icon: '⚡' },
    { type: 'projects', name: 'Projects', icon: '💼' },
    { type: 'experience', name: 'Experience', icon: '📋' },
    { type: 'contact', name: 'Contact', icon: '📧' },
  ];

  return (
    <div className="w-80 bg-white border-r flex flex-col">
      {/* Tabs */}
      <div className="flex border-b">
        <button
          onClick={() => setActiveTab('sections')}
          className={`flex-1 py-3 px-4 text-sm font-medium ${
            activeTab === 'sections'
              ? 'border-b-2 border-blue-600 text-blue-600'
              : 'text-gray-600 hover:text-gray-900'
          }`}
        >
          <Plus className="w-4 h-4 inline mr-2" />
          Sections
        </button>
        <button
          onClick={() => setActiveTab('theme')}
          className={`flex-1 py-3 px-4 text-sm font-medium ${
            activeTab === 'theme'
              ? 'border-b-2 border-blue-600 text-blue-600'
              : 'text-gray-600 hover:text-gray-900'
          }`}
        >
          <Palette className="w-4 h-4 inline mr-2" />
          Theme
        </button>
        <button
          onClick={() => setActiveTab('content')}
          className={`flex-1 py-3 px-4 text-sm font-medium ${
            activeTab === 'content'
              ? 'border-b-2 border-blue-600 text-blue-600'
              : 'text-gray-600 hover:text-gray-900'
          }`}
        >
          <FileText className="w-4 h-4 inline mr-2" />
          Content
        </button>
      </div>

      {/* Content */}
      <div className="flex-1 overflow-y-auto p-4">
        {activeTab === 'sections' && (
          <div className="space-y-2">
            <h3 className="font-semibold mb-3">Add Section</h3>
            {sectionTypes.map((section) => (
              <button
                key={section.type}
                onClick={() => currentPage && addSection(currentPage, section.type as any)}
                className="w-full text-left p-3 border rounded-lg hover:bg-gray-50 flex items-center gap-3"
              >
                <span className="text-2xl">{section.icon}</span>
                <span className="font-medium">{section.name}</span>
              </button>
            ))}
          </div>
        )}

        {activeTab === 'theme' && (
          <div className="space-y-4">
            <h3 className="font-semibold mb-3">Theme Settings</h3>
            <ThemeEditor />
          </div>
        )}

        {activeTab === 'content' && (
          <div className="space-y-4">
            <h3 className="font-semibold mb-3">Content Manager</h3>
            <ContentManager />
          </div>
        )}
      </div>
    </div>
  );
};

function ThemeEditor() {
  const { portfolio, updateTheme } = usePortfolioStore();
  if (!portfolio) return null;

  return (
    <div className="space-y-4">
      <div>
        <label className="block text-sm font-medium mb-2">Primary Color</label>
        <input
          type="color"
          value={portfolio.theme.colors.primary}
          onChange={(e) =>
            updateTheme({
              colors: { ...portfolio.theme.colors, primary: e.target.value },
            })
          }
          className="w-full h-10 rounded cursor-pointer"
        />
      </div>
      {/* Add more theme controls */}
    </div>
  );
}

function ContentManager() {
  return <div>Content management UI here</div>;
}
```

## Expected Outputs

1. **Fully Functional Portfolio Builder** with:
   - Drag-and-drop page builder
   - Live preview with multiple device modes
   - Real-time editing
   - Template system

2. **Customization**:
   - Theme editor with color picker
   - Typography controls
   - Spacing adjustments
   - Component styling

3. **Content Management**:
   - Personal information editor
   - Project management
   - Skills editor
   - Experience timeline

4. **Export & Share**:
   - HTML/CSS export
   - Deployment options
   - Preview sharing
   - Code viewing

## Bonus Challenges

- [ ] Add more section templates
- [ ] Implement undo/redo functionality
- [ ] Add animation presets
- [ ] Create SEO editor
- [ ] Add form builder with integrations
- [ ] Implement A/B testing
- [ ] Add analytics integration
- [ ] Create custom domain mapping
- [ ] Add multilingual support
- [ ] Implement version history
- [ ] Add collaboration features
- [ ] Create component marketplace
- [ ] Add AI-powered content suggestions
- [ ] Implement automatic responsive optimization

## Resources

- [Monaco Editor](https://microsoft.github.io/monaco-editor/)
- [@dnd-kit Documentation](https://docs.dndkit.com/)
- [react-colorful](https://github.com/omgovich/react-colorful)
- [html-to-image](https://github.com/bubkoo/html-to-image)
- [Sandpack](https://sandpack.codesandbox.io/)

## Success Criteria

- Live preview updates in real-time
- All sections can be added and customized
- Theme changes apply immediately
- Export generates valid HTML/CSS
- Preview works on all device sizes
- Drag-and-drop is smooth and intuitive
- Content persists across sessions
- No performance issues with complex portfolios
- TypeScript provides full type safety
- Accessibility standards are met
- Code is well-organized and maintainable
- UI is intuitive and user-friendly
