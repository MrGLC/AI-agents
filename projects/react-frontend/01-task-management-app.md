# Project 1: Task Management App with React Query and TypeScript

## Overview
Build a full-featured task management application using React, TypeScript, and React Query for efficient data fetching and state management. This project implements CRUD operations, filtering, sorting, and real-time data synchronization.

## Difficulty Level
Intermediate

## Learning Objectives
- Master TypeScript with React for type-safe component development
- Implement React Query for server state management
- Handle asynchronous operations with loading and error states
- Build reusable, typed components and custom hooks
- Implement optimistic updates and cache management
- Create responsive UI with modern CSS or Tailwind
- Manage complex form state and validation

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Data Fetching**: React Query (TanStack Query) v5
- **Styling**: Tailwind CSS or styled-components
- **Form Handling**: React Hook Form with Zod validation
- **Routing**: React Router v6
- **Icons**: Lucide React or React Icons
- **Date Handling**: date-fns
- **Build Tool**: Vite
- **API**: JSONPlaceholder or custom mock API

## Project Requirements

### 1. Core Features
- **Task CRUD Operations**
  - Create new tasks with title, description, priority, due date
  - Read and display tasks in multiple views
  - Update task details and status
  - Delete tasks with confirmation

- **Task Properties**
  - Title (required)
  - Description (optional)
  - Status: Todo, In Progress, Completed
  - Priority: Low, Medium, High, Urgent
  - Due date
  - Tags/categories
  - Created/updated timestamps

### 2. UI Components
- Task list with different view modes (list, grid, kanban)
- Task creation/edit modal or drawer
- Filter sidebar (by status, priority, tags)
- Search functionality
- Sort controls (by date, priority, title)
- Task detail view
- Loading skeletons
- Empty states

### 3. React Query Integration
- Query tasks with caching
- Mutations for create/update/delete
- Optimistic updates
- Invalidation and refetching strategies
- Error handling and retry logic
- Loading states

### 4. TypeScript Implementation
- Define strict types for tasks, filters, API responses
- Type-safe API functions
- Typed hooks and components
- Enum for task status and priority

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
# Create Vite project with React + TypeScript template
npm create vite@latest task-manager -- --template react-ts
cd task-manager

# Install dependencies
npm install @tanstack/react-query react-router-dom
npm install react-hook-form @hookform/resolvers zod
npm install date-fns lucide-react
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/task.ts
export enum TaskStatus {
  TODO = 'todo',
  IN_PROGRESS = 'in_progress',
  COMPLETED = 'completed'
}

export enum TaskPriority {
  LOW = 'low',
  MEDIUM = 'medium',
  HIGH = 'high',
  URGENT = 'urgent'
}

export interface Task {
  id: string;
  title: string;
  description?: string;
  status: TaskStatus;
  priority: TaskPriority;
  dueDate?: string;
  tags: string[];
  createdAt: string;
  updatedAt: string;
}

export interface CreateTaskDTO {
  title: string;
  description?: string;
  status: TaskStatus;
  priority: TaskPriority;
  dueDate?: string;
  tags: string[];
}

export interface UpdateTaskDTO extends Partial<CreateTaskDTO> {
  id: string;
}

export interface TaskFilters {
  status?: TaskStatus[];
  priority?: TaskPriority[];
  tags?: string[];
  search?: string;
}
```

### Step 3: Create API Client
```typescript
// src/api/tasks.ts
import type { Task, CreateTaskDTO, UpdateTaskDTO } from '../types/task';

const API_BASE = 'http://localhost:3001/api'; // or JSONPlaceholder

export const tasksApi = {
  getTasks: async (): Promise<Task[]> => {
    const response = await fetch(`${API_BASE}/tasks`);
    if (!response.ok) throw new Error('Failed to fetch tasks');
    return response.json();
  },

  getTask: async (id: string): Promise<Task> => {
    const response = await fetch(`${API_BASE}/tasks/${id}`);
    if (!response.ok) throw new Error('Failed to fetch task');
    return response.json();
  },

  createTask: async (data: CreateTaskDTO): Promise<Task> => {
    const response = await fetch(`${API_BASE}/tasks`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...data,
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      }),
    });
    if (!response.ok) throw new Error('Failed to create task');
    return response.json();
  },

  updateTask: async ({ id, ...data }: UpdateTaskDTO): Promise<Task> => {
    const response = await fetch(`${API_BASE}/tasks/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...data,
        updatedAt: new Date().toISOString(),
      }),
    });
    if (!response.ok) throw new Error('Failed to update task');
    return response.json();
  },

  deleteTask: async (id: string): Promise<void> => {
    const response = await fetch(`${API_BASE}/tasks/${id}`, {
      method: 'DELETE',
    });
    if (!response.ok) throw new Error('Failed to delete task');
  },
};
```

### Step 4: Setup React Query
```typescript
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      gcTime: 1000 * 60 * 10, // 10 minutes
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
);
```

### Step 5: Create Custom Hooks
```typescript
// src/hooks/useTasks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { tasksApi } from '../api/tasks';
import type { CreateTaskDTO, UpdateTaskDTO, Task } from '../types/task';

const TASKS_QUERY_KEY = ['tasks'];

export const useTasks = () => {
  return useQuery({
    queryKey: TASKS_QUERY_KEY,
    queryFn: tasksApi.getTasks,
  });
};

export const useTask = (id: string) => {
  return useQuery({
    queryKey: [...TASKS_QUERY_KEY, id],
    queryFn: () => tasksApi.getTask(id),
    enabled: !!id,
  });
};

export const useCreateTask = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: tasksApi.createTask,
    onMutate: async (newTask) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: TASKS_QUERY_KEY });

      // Snapshot previous value
      const previousTasks = queryClient.getQueryData<Task[]>(TASKS_QUERY_KEY);

      // Optimistically update
      if (previousTasks) {
        queryClient.setQueryData<Task[]>(TASKS_QUERY_KEY, [
          ...previousTasks,
          { ...newTask, id: 'temp-id', createdAt: new Date().toISOString(), updatedAt: new Date().toISOString() } as Task,
        ]);
      }

      return { previousTasks };
    },
    onError: (err, newTask, context) => {
      // Rollback on error
      if (context?.previousTasks) {
        queryClient.setQueryData(TASKS_QUERY_KEY, context.previousTasks);
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: TASKS_QUERY_KEY });
    },
  });
};

export const useUpdateTask = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: tasksApi.updateTask,
    onMutate: async (updatedTask) => {
      await queryClient.cancelQueries({ queryKey: TASKS_QUERY_KEY });
      const previousTasks = queryClient.getQueryData<Task[]>(TASKS_QUERY_KEY);

      if (previousTasks) {
        queryClient.setQueryData<Task[]>(
          TASKS_QUERY_KEY,
          previousTasks.map((task) =>
            task.id === updatedTask.id ? { ...task, ...updatedTask } : task
          )
        );
      }

      return { previousTasks };
    },
    onError: (err, updatedTask, context) => {
      if (context?.previousTasks) {
        queryClient.setQueryData(TASKS_QUERY_KEY, context.previousTasks);
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: TASKS_QUERY_KEY });
    },
  });
};

export const useDeleteTask = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: tasksApi.deleteTask,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: TASKS_QUERY_KEY });
    },
  });
};
```

### Step 6: Create Task Form Component
```typescript
// src/components/TaskForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { TaskStatus, TaskPriority } from '../types/task';

const taskSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100),
  description: z.string().max(500).optional(),
  status: z.nativeEnum(TaskStatus),
  priority: z.nativeEnum(TaskPriority),
  dueDate: z.string().optional(),
  tags: z.array(z.string()),
});

type TaskFormData = z.infer<typeof taskSchema>;

interface TaskFormProps {
  onSubmit: (data: TaskFormData) => void;
  defaultValues?: Partial<TaskFormData>;
  isLoading?: boolean;
}

export const TaskForm: React.FC<TaskFormProps> = ({
  onSubmit,
  defaultValues,
  isLoading,
}) => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<TaskFormData>({
    resolver: zodResolver(taskSchema),
    defaultValues: defaultValues || {
      status: TaskStatus.TODO,
      priority: TaskPriority.MEDIUM,
      tags: [],
    },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label className="block text-sm font-medium mb-1">Title</label>
        <input
          {...register('title')}
          className="w-full px-3 py-2 border rounded-md"
          placeholder="Task title"
        />
        {errors.title && (
          <p className="text-red-500 text-sm mt-1">{errors.title.message}</p>
        )}
      </div>

      <div>
        <label className="block text-sm font-medium mb-1">Description</label>
        <textarea
          {...register('description')}
          className="w-full px-3 py-2 border rounded-md"
          rows={3}
          placeholder="Task description (optional)"
        />
      </div>

      <div className="grid grid-cols-2 gap-4">
        <div>
          <label className="block text-sm font-medium mb-1">Status</label>
          <select {...register('status')} className="w-full px-3 py-2 border rounded-md">
            <option value={TaskStatus.TODO}>To Do</option>
            <option value={TaskStatus.IN_PROGRESS}>In Progress</option>
            <option value={TaskStatus.COMPLETED}>Completed</option>
          </select>
        </div>

        <div>
          <label className="block text-sm font-medium mb-1">Priority</label>
          <select {...register('priority')} className="w-full px-3 py-2 border rounded-md">
            <option value={TaskPriority.LOW}>Low</option>
            <option value={TaskPriority.MEDIUM}>Medium</option>
            <option value={TaskPriority.HIGH}>High</option>
            <option value={TaskPriority.URGENT}>Urgent</option>
          </select>
        </div>
      </div>

      <div>
        <label className="block text-sm font-medium mb-1">Due Date</label>
        <input
          type="date"
          {...register('dueDate')}
          className="w-full px-3 py-2 border rounded-md"
        />
      </div>

      <button
        type="submit"
        disabled={isLoading}
        className="w-full bg-blue-600 text-white py-2 rounded-md hover:bg-blue-700 disabled:opacity-50"
      >
        {isLoading ? 'Saving...' : 'Save Task'}
      </button>
    </form>
  );
};
```

### Step 7: Create Task List Component
```typescript
// src/components/TaskList.tsx
import { useTasks, useUpdateTask, useDeleteTask } from '../hooks/useTasks';
import { Task } from '../types/task';
import { format } from 'date-fns';
import { Trash2, Edit, CheckCircle } from 'lucide-react';

export const TaskList: React.FC = () => {
  const { data: tasks, isLoading, error } = useTasks();
  const updateTask = useUpdateTask();
  const deleteTask = useDeleteTask();

  if (isLoading) {
    return <div>Loading tasks...</div>;
  }

  if (error) {
    return <div>Error loading tasks: {error.message}</div>;
  }

  if (!tasks || tasks.length === 0) {
    return <div>No tasks found. Create your first task!</div>;
  }

  const handleToggleComplete = (task: Task) => {
    updateTask.mutate({
      id: task.id,
      status: task.status === 'completed' ? 'todo' : 'completed',
    });
  };

  return (
    <div className="space-y-3">
      {tasks.map((task) => (
        <div
          key={task.id}
          className="border rounded-lg p-4 hover:shadow-md transition-shadow"
        >
          <div className="flex items-start justify-between">
            <div className="flex-1">
              <div className="flex items-center gap-2">
                <button
                  onClick={() => handleToggleComplete(task)}
                  className="text-gray-400 hover:text-green-600"
                >
                  <CheckCircle
                    className={task.status === 'completed' ? 'fill-green-600 text-white' : ''}
                  />
                </button>
                <h3
                  className={`font-semibold ${
                    task.status === 'completed' ? 'line-through text-gray-400' : ''
                  }`}
                >
                  {task.title}
                </h3>
                <span
                  className={`text-xs px-2 py-1 rounded ${
                    task.priority === 'urgent'
                      ? 'bg-red-100 text-red-800'
                      : task.priority === 'high'
                      ? 'bg-orange-100 text-orange-800'
                      : task.priority === 'medium'
                      ? 'bg-yellow-100 text-yellow-800'
                      : 'bg-gray-100 text-gray-800'
                  }`}
                >
                  {task.priority}
                </span>
              </div>
              {task.description && (
                <p className="text-gray-600 mt-2">{task.description}</p>
              )}
              {task.dueDate && (
                <p className="text-sm text-gray-500 mt-2">
                  Due: {format(new Date(task.dueDate), 'MMM dd, yyyy')}
                </p>
              )}
            </div>
            <div className="flex gap-2">
              <button className="text-blue-600 hover:text-blue-800">
                <Edit size={18} />
              </button>
              <button
                onClick={() => deleteTask.mutate(task.id)}
                className="text-red-600 hover:text-red-800"
              >
                <Trash2 size={18} />
              </button>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
};
```

## Expected Outputs

1. **Functional Task Management App** with:
   - Clean, responsive UI
   - Smooth task creation and editing
   - Real-time updates with optimistic UI
   - Proper loading and error states

2. **Type Safety**:
   - No TypeScript errors
   - Full type coverage for components and functions
   - Intelligent autocomplete in IDE

3. **Performance**:
   - Fast data fetching with caching
   - Optimistic updates for instant feedback
   - Minimal re-renders

4. **User Experience**:
   - Intuitive navigation
   - Visual feedback for all actions
   - Form validation with helpful error messages

## Bonus Challenges

- [ ] Add task categories/projects for organization
- [ ] Implement drag-and-drop for reordering tasks
- [ ] Add due date notifications/reminders
- [ ] Create a kanban board view with drag-and-drop between columns
- [ ] Add task time tracking functionality
- [ ] Implement task search with debouncing
- [ ] Add dark mode toggle
- [ ] Create task templates for recurring tasks
- [ ] Add subtasks/checklist items within tasks
- [ ] Implement data persistence with localStorage as fallback
- [ ] Add task statistics dashboard
- [ ] Export tasks to CSV/JSON
- [ ] Add collaboration features (assign tasks, comments)

## Resources

- [React Query Documentation](https://tanstack.com/query/latest)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [React Hook Form](https://react-hook-form.com/)
- [Zod Validation](https://zod.dev/)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)

## Success Criteria

- All CRUD operations work correctly
- TypeScript types are properly defined with no `any` types
- React Query manages all server state
- Optimistic updates provide instant feedback
- Error handling is implemented for all operations
- UI is responsive and accessible
- Code follows React and TypeScript best practices
- Loading states are displayed appropriately
- Forms validate input before submission
- App handles edge cases (empty states, errors, slow network)
