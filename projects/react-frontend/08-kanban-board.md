# Project 8: Kanban Board with Drag-and-Drop

## Overview
Build a powerful Kanban board application with drag-and-drop functionality, similar to Trello or Jira. This project focuses on implementing complex drag-and-drop interactions, real-time state updates, board management, and collaborative features.

## Difficulty Level
Advanced

## Learning Objectives
- Implement drag-and-drop with @dnd-kit or react-beautiful-dnd
- Manage complex nested state structures
- Handle optimistic UI updates
- Create custom hooks for board logic
- Implement keyboard accessibility for drag-and-drop
- Build responsive board layouts
- Handle collision detection and drop zones
- Create smooth animations and transitions
- Implement undo/redo functionality

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Drag & Drop**: @dnd-kit/core (modern, accessible)
- **State Management**: Zustand with immer middleware
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **Animations**: Framer Motion
- **Storage**: localStorage for persistence
- **Build Tool**: Vite
- **Date Handling**: date-fns

## Project Requirements

### 1. Board Features
- **Board Management**
  - Create multiple boards
  - Rename/delete boards
  - Board templates
  - Board backgrounds/themes
  - Archive boards
  - Board search

- **Column Management**
  - Add/remove columns
  - Rename columns
  - Reorder columns via drag-and-drop
  - Set column limits (WIP limits)
  - Collapse/expand columns
  - Column colors

### 2. Card Features
- **Card Management**
  - Create cards with quick add
  - Edit card details
  - Delete cards
  - Duplicate cards
  - Move cards between columns
  - Reorder cards within columns
  - Archive cards

- **Card Details**
  - Title and description
  - Labels/tags with colors
  - Due dates with overdue indicators
  - Priority levels
  - Assignees with avatars
  - Attachments
  - Checklists
  - Comments
  - Activity log

- **Card Actions**
  - Drag-and-drop movement
  - Quick edit
  - Copy card link
  - Move to another board
  - Set cover image

### 3. Drag-and-Drop Interactions
- **Card Dragging**
  - Smooth drag animations
  - Visual feedback during drag
  - Auto-scroll when near edges
  - Drop indicators
  - Keyboard navigation support

- **Column Dragging**
  - Reorder columns
  - Visual placeholders
  - Smooth transitions

- **Multi-select**
  - Select multiple cards
  - Bulk operations
  - Move multiple cards together

### 4. Filtering and Search
- Search cards across board
- Filter by labels
- Filter by assignee
- Filter by due date
- Filter by priority
- Clear filters

### 5. UI Components
- Board header with actions
- Column with card list
- Card component with all details
- Card detail modal
- Label manager
- Member manager
- Board settings panel
- Context menus

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest kanban-board -- --template react-ts
cd kanban-board

npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
npm install zustand immer
npm install framer-motion lucide-react
npm install date-fns
npm install react-router-dom
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/board.ts
export interface Board {
  id: string;
  title: string;
  description?: string;
  background?: string;
  columns: Column[];
  labels: Label[];
  members: Member[];
  createdAt: string;
  updatedAt: string;
}

export interface Column {
  id: string;
  title: string;
  color?: string;
  cards: Card[];
  wipLimit?: number;
  collapsed: boolean;
  order: number;
}

export interface Card {
  id: string;
  title: string;
  description?: string;
  columnId: string;
  order: number;
  labels: string[];
  members: string[];
  dueDate?: string;
  priority?: 'low' | 'medium' | 'high' | 'urgent';
  checklist?: ChecklistItem[];
  attachments?: Attachment[];
  comments?: Comment[];
  coverImage?: string;
  archived: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface ChecklistItem {
  id: string;
  text: string;
  completed: boolean;
  order: number;
}

export interface Attachment {
  id: string;
  name: string;
  url: string;
  type: string;
  size: number;
  uploadedAt: string;
}

export interface Comment {
  id: string;
  author: string;
  text: string;
  createdAt: string;
  updatedAt?: string;
}

export interface Label {
  id: string;
  name: string;
  color: string;
}

export interface Member {
  id: string;
  name: string;
  avatar?: string;
  email: string;
}

export interface BoardFilters {
  search: string;
  labels: string[];
  members: string[];
  dueDate?: 'overdue' | 'today' | 'week' | 'month';
  priority?: string[];
}
```

### Step 3: Create Zustand Store with Immer
```typescript
// src/store/boardStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';
import type { Board, Column, Card, Label, Member } from '@/types/board';

interface BoardState {
  boards: Board[];
  activeBoard: string | null;
  filters: BoardFilters;

  // Board actions
  createBoard: (title: string) => void;
  deleteBoard: (boardId: string) => void;
  updateBoard: (boardId: string, updates: Partial<Board>) => void;
  setActiveBoard: (boardId: string) => void;

  // Column actions
  addColumn: (boardId: string, title: string) => void;
  updateColumn: (boardId: string, columnId: string, updates: Partial<Column>) => void;
  deleteColumn: (boardId: string, columnId: string) => void;
  moveColumn: (boardId: string, fromIndex: number, toIndex: number) => void;

  // Card actions
  addCard: (boardId: string, columnId: string, title: string) => void;
  updateCard: (boardId: string, cardId: string, updates: Partial<Card>) => void;
  deleteCard: (boardId: string, cardId: string) => void;
  moveCard: (
    boardId: string,
    cardId: string,
    fromColumnId: string,
    toColumnId: string,
    newOrder: number
  ) => void;

  // Label actions
  addLabel: (boardId: string, name: string, color: string) => void;
  updateLabel: (boardId: string, labelId: string, updates: Partial<Label>) => void;
  deleteLabel: (boardId: string, labelId: string) => void;

  // Filter actions
  setFilters: (filters: Partial<BoardFilters>) => void;
  clearFilters: () => void;

  // Helper getters
  getBoard: (boardId: string) => Board | undefined;
  getCard: (boardId: string, cardId: string) => Card | undefined;
}

export const useBoardStore = create<BoardState>()(
  persist(
    immer((set, get) => ({
      boards: [],
      activeBoard: null,
      filters: {
        search: '',
        labels: [],
        members: [],
      },

      createBoard: (title) => {
        set((state) => {
          const newBoard: Board = {
            id: Date.now().toString(),
            title,
            columns: [],
            labels: [],
            members: [],
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString(),
          };
          state.boards.push(newBoard);
        });
      },

      deleteBoard: (boardId) => {
        set((state) => {
          state.boards = state.boards.filter((b) => b.id !== boardId);
          if (state.activeBoard === boardId) {
            state.activeBoard = null;
          }
        });
      },

      updateBoard: (boardId, updates) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            Object.assign(board, updates, {
              updatedAt: new Date().toISOString(),
            });
          }
        });
      },

      setActiveBoard: (boardId) => {
        set({ activeBoard: boardId });
      },

      addColumn: (boardId, title) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const newColumn: Column = {
              id: Date.now().toString(),
              title,
              cards: [],
              collapsed: false,
              order: board.columns.length,
            };
            board.columns.push(newColumn);
          }
        });
      },

      updateColumn: (boardId, columnId, updates) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const column = board.columns.find((c) => c.id === columnId);
            if (column) {
              Object.assign(column, updates);
            }
          }
        });
      },

      deleteColumn: (boardId, columnId) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            board.columns = board.columns.filter((c) => c.id !== columnId);
          }
        });
      },

      moveColumn: (boardId, fromIndex, toIndex) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const [removed] = board.columns.splice(fromIndex, 1);
            board.columns.splice(toIndex, 0, removed);
            // Update order
            board.columns.forEach((col, idx) => {
              col.order = idx;
            });
          }
        });
      },

      addCard: (boardId, columnId, title) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const column = board.columns.find((c) => c.id === columnId);
            if (column) {
              const newCard: Card = {
                id: Date.now().toString(),
                title,
                columnId,
                order: column.cards.length,
                labels: [],
                members: [],
                archived: false,
                createdAt: new Date().toISOString(),
                updatedAt: new Date().toISOString(),
              };
              column.cards.push(newCard);
            }
          }
        });
      },

      updateCard: (boardId, cardId, updates) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            for (const column of board.columns) {
              const card = column.cards.find((c) => c.id === cardId);
              if (card) {
                Object.assign(card, updates, {
                  updatedAt: new Date().toISOString(),
                });
                break;
              }
            }
          }
        });
      },

      deleteCard: (boardId, cardId) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            for (const column of board.columns) {
              column.cards = column.cards.filter((c) => c.id !== cardId);
            }
          }
        });
      },

      moveCard: (boardId, cardId, fromColumnId, toColumnId, newOrder) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const fromColumn = board.columns.find((c) => c.id === fromColumnId);
            const toColumn = board.columns.find((c) => c.id === toColumnId);

            if (fromColumn && toColumn) {
              const cardIndex = fromColumn.cards.findIndex((c) => c.id === cardId);
              if (cardIndex !== -1) {
                const [card] = fromColumn.cards.splice(cardIndex, 1);
                card.columnId = toColumnId;
                toColumn.cards.splice(newOrder, 0, card);

                // Update order for all cards
                fromColumn.cards.forEach((c, idx) => {
                  c.order = idx;
                });
                toColumn.cards.forEach((c, idx) => {
                  c.order = idx;
                });
              }
            }
          }
        });
      },

      addLabel: (boardId, name, color) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            board.labels.push({
              id: Date.now().toString(),
              name,
              color,
            });
          }
        });
      },

      updateLabel: (boardId, labelId, updates) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            const label = board.labels.find((l) => l.id === labelId);
            if (label) {
              Object.assign(label, updates);
            }
          }
        });
      },

      deleteLabel: (boardId, labelId) => {
        set((state) => {
          const board = state.boards.find((b) => b.id === boardId);
          if (board) {
            board.labels = board.labels.filter((l) => l.id !== labelId);
          }
        });
      },

      setFilters: (filters) => {
        set((state) => {
          Object.assign(state.filters, filters);
        });
      },

      clearFilters: () => {
        set({
          filters: {
            search: '',
            labels: [],
            members: [],
          },
        });
      },

      getBoard: (boardId) => {
        return get().boards.find((b) => b.id === boardId);
      },

      getCard: (boardId, cardId) => {
        const board = get().boards.find((b) => b.id === boardId);
        if (board) {
          for (const column of board.columns) {
            const card = column.cards.find((c) => c.id === cardId);
            if (card) return card;
          }
        }
        return undefined;
      },
    })),
    {
      name: 'kanban-storage',
    }
  )
);
```

### Step 4: Create Draggable Card Component
```typescript
// src/components/KanbanCard.tsx
import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { Calendar, User, CheckSquare, Paperclip, MessageSquare, MoreHorizontal } from 'lucide-react';
import { format, isPast, isToday } from 'date-fns';
import type { Card, Label } from '@/types/board';

interface KanbanCardProps {
  card: Card;
  labels: Label[];
  onEdit: () => void;
}

export const KanbanCard: React.FC<KanbanCardProps> = ({ card, labels, onEdit }) => {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id: card.id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
  };

  const cardLabels = labels.filter((l) => card.labels.includes(l.id));
  const isOverdue = card.dueDate && isPast(new Date(card.dueDate)) && !isToday(new Date(card.dueDate));
  const isDueToday = card.dueDate && isToday(new Date(card.dueDate));

  const completedChecklist = card.checklist?.filter((item) => item.completed).length || 0;
  const totalChecklist = card.checklist?.length || 0;

  return (
    <div
      ref={setNodeRef}
      style={style}
      {...attributes}
      {...listeners}
      onClick={onEdit}
      className="bg-white rounded-lg shadow-sm hover:shadow-md transition-shadow p-3 mb-3 cursor-pointer group"
    >
      {/* Cover Image */}
      {card.coverImage && (
        <img
          src={card.coverImage}
          alt=""
          className="w-full h-32 object-cover rounded-md mb-3 -mx-3 -mt-3"
        />
      )}

      {/* Labels */}
      {cardLabels.length > 0 && (
        <div className="flex gap-1 mb-2 flex-wrap">
          {cardLabels.map((label) => (
            <span
              key={label.id}
              className="px-2 py-1 rounded text-xs font-semibold text-white"
              style={{ backgroundColor: label.color }}
            >
              {label.name}
            </span>
          ))}
        </div>
      )}

      {/* Title */}
      <h4 className="font-medium text-gray-900 mb-2">{card.title}</h4>

      {/* Metadata */}
      <div className="flex items-center gap-3 text-sm text-gray-600">
        {/* Due Date */}
        {card.dueDate && (
          <div
            className={`flex items-center gap-1 px-2 py-1 rounded ${
              isOverdue
                ? 'bg-red-100 text-red-700'
                : isDueToday
                ? 'bg-yellow-100 text-yellow-700'
                : 'bg-gray-100'
            }`}
          >
            <Calendar className="w-3 h-3" />
            <span className="text-xs">
              {format(new Date(card.dueDate), 'MMM dd')}
            </span>
          </div>
        )}

        {/* Checklist */}
        {totalChecklist > 0 && (
          <div
            className={`flex items-center gap-1 ${
              completedChecklist === totalChecklist ? 'text-green-600' : ''
            }`}
          >
            <CheckSquare className="w-3 h-3" />
            <span className="text-xs">
              {completedChecklist}/{totalChecklist}
            </span>
          </div>
        )}

        {/* Attachments */}
        {card.attachments && card.attachments.length > 0 && (
          <div className="flex items-center gap-1">
            <Paperclip className="w-3 h-3" />
            <span className="text-xs">{card.attachments.length}</span>
          </div>
        )}

        {/* Comments */}
        {card.comments && card.comments.length > 0 && (
          <div className="flex items-center gap-1">
            <MessageSquare className="w-3 h-3" />
            <span className="text-xs">{card.comments.length}</span>
          </div>
        )}

        {/* Members */}
        {card.members.length > 0 && (
          <div className="flex -space-x-2 ml-auto">
            {card.members.slice(0, 3).map((memberId) => (
              <div
                key={memberId}
                className="w-6 h-6 rounded-full bg-blue-500 border-2 border-white flex items-center justify-center text-white text-xs font-semibold"
              >
                {memberId.substring(0, 2).toUpperCase()}
              </div>
            ))}
            {card.members.length > 3 && (
              <div className="w-6 h-6 rounded-full bg-gray-300 border-2 border-white flex items-center justify-center text-xs font-semibold">
                +{card.members.length - 3}
              </div>
            )}
          </div>
        )}
      </div>

      {/* Priority Indicator */}
      {card.priority && (
        <div className="mt-2">
          <span
            className={`text-xs px-2 py-1 rounded ${
              card.priority === 'urgent'
                ? 'bg-red-100 text-red-700'
                : card.priority === 'high'
                ? 'bg-orange-100 text-orange-700'
                : card.priority === 'medium'
                ? 'bg-yellow-100 text-yellow-700'
                : 'bg-blue-100 text-blue-700'
            }`}
          >
            {card.priority.charAt(0).toUpperCase() + card.priority.slice(1)} Priority
          </span>
        </div>
      )}
    </div>
  );
};
```

### Step 5: Create Column Component
```typescript
// src/components/KanbanColumn.tsx
import { useDroppable } from '@dnd-kit/core';
import { SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable';
import { Plus, MoreHorizontal, ChevronDown, ChevronUp } from 'lucide-react';
import { useState } from 'react';
import { KanbanCard } from './KanbanCard';
import type { Column, Card, Label } from '@/types/board';

interface KanbanColumnProps {
  column: Column;
  cards: Card[];
  labels: Label[];
  onAddCard: (columnId: string, title: string) => void;
  onEditCard: (card: Card) => void;
  onToggleCollapse: (columnId: string) => void;
}

export const KanbanColumn: React.FC<KanbanColumnProps> = ({
  column,
  cards,
  labels,
  onAddCard,
  onEditCard,
  onToggleCollapse,
}) => {
  const [isAddingCard, setIsAddingCard] = useState(false);
  const [newCardTitle, setNewCardTitle] = useState('');

  const { setNodeRef } = useDroppable({
    id: column.id,
  });

  const handleAddCard = () => {
    if (newCardTitle.trim()) {
      onAddCard(column.id, newCardTitle.trim());
      setNewCardTitle('');
      setIsAddingCard(false);
    }
  };

  const isAtLimit = column.wipLimit && cards.length >= column.wipLimit;

  return (
    <div className="bg-gray-100 rounded-lg p-3 w-80 flex-shrink-0">
      {/* Column Header */}
      <div className="flex items-center justify-between mb-3">
        <div className="flex items-center gap-2">
          <button
            onClick={() => onToggleCollapse(column.id)}
            className="text-gray-600 hover:text-gray-900"
          >
            {column.collapsed ? (
              <ChevronDown className="w-4 h-4" />
            ) : (
              <ChevronUp className="w-4 h-4" />
            )}
          </button>
          <h3 className="font-semibold text-gray-900">
            {column.title}
          </h3>
          <span className="text-sm text-gray-500">
            {cards.length}
            {column.wipLimit && ` / ${column.wipLimit}`}
          </span>
          {isAtLimit && (
            <span className="text-xs bg-red-100 text-red-700 px-2 py-0.5 rounded">
              Limit reached
            </span>
          )}
        </div>
        <div className="flex items-center gap-1">
          <button className="p-1 hover:bg-gray-200 rounded">
            <MoreHorizontal className="w-4 h-4 text-gray-600" />
          </button>
        </div>
      </div>

      {/* Cards */}
      {!column.collapsed && (
        <>
          <div ref={setNodeRef} className="min-h-[100px]">
            <SortableContext
              items={cards.map((c) => c.id)}
              strategy={verticalListSortingStrategy}
            >
              {cards.map((card) => (
                <KanbanCard
                  key={card.id}
                  card={card}
                  labels={labels}
                  onEdit={() => onEditCard(card)}
                />
              ))}
            </SortableContext>
          </div>

          {/* Add Card */}
          {isAddingCard ? (
            <div className="mt-2">
              <textarea
                autoFocus
                value={newCardTitle}
                onChange={(e) => setNewCardTitle(e.target.value)}
                onKeyDown={(e) => {
                  if (e.key === 'Enter' && !e.shiftKey) {
                    e.preventDefault();
                    handleAddCard();
                  }
                  if (e.key === 'Escape') {
                    setIsAddingCard(false);
                    setNewCardTitle('');
                  }
                }}
                placeholder="Enter card title..."
                className="w-full p-2 border border-gray-300 rounded-md resize-none"
                rows={3}
              />
              <div className="flex gap-2 mt-2">
                <button
                  onClick={handleAddCard}
                  className="px-3 py-1 bg-blue-600 text-white rounded-md hover:bg-blue-700 text-sm"
                >
                  Add
                </button>
                <button
                  onClick={() => {
                    setIsAddingCard(false);
                    setNewCardTitle('');
                  }}
                  className="px-3 py-1 text-gray-600 hover:text-gray-900 text-sm"
                >
                  Cancel
                </button>
              </div>
            </div>
          ) : (
            <button
              onClick={() => setIsAddingCard(true)}
              disabled={isAtLimit}
              className="w-full flex items-center gap-2 text-gray-600 hover:text-gray-900 hover:bg-gray-200 rounded-md p-2 mt-2 disabled:opacity-50 disabled:cursor-not-allowed"
            >
              <Plus className="w-4 h-4" />
              <span>Add a card</span>
            </button>
          )}
        </>
      )}
    </div>
  );
};
```

### Step 6: Create Board View with DnD Context
```typescript
// src/pages/BoardPage.tsx
import { useState } from 'react';
import {
  DndContext,
  DragEndEvent,
  DragOverEvent,
  DragOverlay,
  DragStartEvent,
  PointerSensor,
  useSensor,
  useSensors,
  closestCorners,
} from '@dnd-kit/core';
import { arrayMove, SortableContext, horizontalListSortingStrategy } from '@dnd-kit/sortable';
import { Plus } from 'lucide-react';
import { KanbanColumn } from '@/components/KanbanColumn';
import { useBoardStore } from '@/store/boardStore';
import type { Card } from '@/types/board';

export const BoardPage: React.FC = () => {
  const { activeBoard, getBoard, addColumn, addCard, moveCard, updateColumn } =
    useBoardStore();

  const board = activeBoard ? getBoard(activeBoard) : null;
  const [activeCard, setActiveCard] = useState<Card | null>(null);

  const sensors = useSensors(
    useSensor(PointerSensor, {
      activationConstraint: {
        distance: 8,
      },
    })
  );

  const handleDragStart = (event: DragStartEvent) => {
    const { active } = event;
    if (board) {
      const card = board.columns
        .flatMap((col) => col.cards)
        .find((c) => c.id === active.id);
      setActiveCard(card || null);
    }
  };

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event;
    setActiveCard(null);

    if (!over || !board) return;

    const activeCard = board.columns
      .flatMap((col) => col.cards)
      .find((c) => c.id === active.id);

    if (!activeCard) return;

    const overColumn = board.columns.find(
      (col) =>
        col.id === over.id || col.cards.some((c) => c.id === over.id)
    );

    if (!overColumn) return;

    const overCard = overColumn.cards.find((c) => c.id === over.id);
    const newOrder = overCard ? overCard.order : overColumn.cards.length;

    moveCard(
      board.id,
      activeCard.id,
      activeCard.columnId,
      overColumn.id,
      newOrder
    );
  };

  if (!board) {
    return <div>No board selected</div>;
  }

  return (
    <div className="h-screen flex flex-col bg-gradient-to-br from-blue-50 to-indigo-50">
      {/* Board Header */}
      <header className="bg-white shadow-sm px-6 py-4">
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-bold text-gray-900">{board.title}</h1>
            {board.description && (
              <p className="text-gray-600 mt-1">{board.description}</p>
            )}
          </div>
          <div className="flex items-center gap-4">
            <button className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700">
              Share
            </button>
          </div>
        </div>
      </header>

      {/* Board Content */}
      <div className="flex-1 overflow-x-auto p-6">
        <DndContext
          sensors={sensors}
          collisionDetection={closestCorners}
          onDragStart={handleDragStart}
          onDragEnd={handleDragEnd}
        >
          <div className="flex gap-4">
            <SortableContext
              items={board.columns.map((c) => c.id)}
              strategy={horizontalListSortingStrategy}
            >
              {board.columns.map((column) => (
                <KanbanColumn
                  key={column.id}
                  column={column}
                  cards={column.cards}
                  labels={board.labels}
                  onAddCard={addCard.bind(null, board.id)}
                  onEditCard={(card) => console.log('Edit card', card)}
                  onToggleCollapse={(columnId) =>
                    updateColumn(board.id, columnId, {
                      collapsed: !column.collapsed,
                    })
                  }
                />
              ))}
            </SortableContext>

            {/* Add Column */}
            <button
              onClick={() => {
                const title = prompt('Enter column title:');
                if (title) addColumn(board.id, title);
              }}
              className="bg-white/50 hover:bg-white/80 rounded-lg p-3 w-80 flex-shrink-0 flex items-center justify-center gap-2 text-gray-700 hover:text-gray-900 transition-colors"
            >
              <Plus className="w-5 h-5" />
              <span>Add Column</span>
            </button>
          </div>

          <DragOverlay>
            {activeCard && (
              <div className="bg-white rounded-lg shadow-lg p-3 w-80 rotate-3">
                <h4 className="font-medium">{activeCard.title}</h4>
              </div>
            )}
          </DragOverlay>
        </DndContext>
      </div>
    </div>
  );
};
```

## Expected Outputs

1. **Fully Functional Kanban Board** with:
   - Smooth drag-and-drop interactions
   - Multiple boards support
   - Column management
   - Card CRUD operations

2. **Rich Card Features**:
   - Labels and tags
   - Due dates with indicators
   - Checklists
   - Members
   - Attachments
   - Comments

3. **User Experience**:
   - Intuitive drag-and-drop
   - Keyboard accessibility
   - Visual feedback
   - Responsive design

4. **Performance**:
   - Smooth animations at 60fps
   - Optimistic UI updates
   - Persistent state
   - Fast interactions

## Bonus Challenges

- [ ] Add card templates
- [ ] Implement board templates (Scrum, Kanban, etc.)
- [ ] Add custom fields to cards
- [ ] Create automation rules (move card when checklist complete)
- [ ] Add card dependencies
- [ ] Implement calendar view
- [ ] Add time tracking
- [ ] Create activity feed
- [ ] Add board analytics
- [ ] Implement real-time collaboration with WebSocket
- [ ] Add card voting/reactions
- [ ] Create sprint planning features
- [ ] Add burndown charts
- [ ] Implement card linking

## Resources

- [@dnd-kit Documentation](https://docs.dndkit.com/)
- [Zustand with Immer](https://docs.pmnd.rs/zustand/integrations/immer-middleware)
- [Drag and Drop Accessibility](https://www.w3.org/WAI/ARIA/apg/patterns/drag-and-drop/)
- [Framer Motion](https://www.framer.com/motion/)

## Success Criteria

- Cards drag smoothly between columns
- Columns can be reordered
- State persists across page refreshes
- Keyboard navigation works
- All CRUD operations function correctly
- UI is responsive on all screen sizes
- Loading states are clear
- Animations are smooth (60fps)
- TypeScript provides full type safety
- Accessibility standards are met
- No layout shifts during drag
- Drop zones are clearly indicated
- Multi-board management works
