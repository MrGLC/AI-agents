# Project 7: Recipe Finder with Search and Favorites

## Overview
Build a comprehensive recipe discovery application with advanced search, filtering, favorites management, meal planning, and shopping list generation. This project focuses on creating an engaging food app experience similar to Tasty or Allrecipes.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Integrate recipe APIs (Spoonacular, Edamam)
- Implement advanced search with multiple filters
- Create favorites and collections management
- Build meal planning calendar
- Generate shopping lists from recipes
- Handle dietary restrictions and nutrition data
- Create responsive recipe cards and detail views
- Implement recipe scaling and unit conversions
- Add print-friendly recipe views

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Data Fetching**: React Query (TanStack Query)
- **State Management**: Zustand with persistence
- **Routing**: React Router v6
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **API**: Spoonacular API or Edamam Recipe API
- **Storage**: localStorage for favorites and meal plans
- **Build Tool**: Vite
- **Calendar**: React Big Calendar (for meal planning)

## Project Requirements

### 1. Recipe Search Features
- **Search Functionality**
  - Search by recipe name
  - Search by ingredients
  - Autocomplete suggestions
  - Search history
  - Advanced search options

- **Filters**
  - Cuisine type (Italian, Mexican, Asian, etc.)
  - Meal type (breakfast, lunch, dinner, snack)
  - Diet type (vegetarian, vegan, gluten-free, keto)
  - Cook time (< 15 min, 15-30 min, 30-60 min, > 60 min)
  - Difficulty level (easy, medium, hard)
  - Calories range
  - Number of servings
  - Exclude ingredients

- **Sorting Options**
  - Relevance
  - Popularity
  - Prep time
  - Calories
  - Healthiness score

### 2. Recipe Display
- **Recipe Card**
  - Recipe image
  - Title and description
  - Rating and reviews
  - Cook time and servings
  - Difficulty indicator
  - Dietary tags
  - Favorite button
  - Quick view modal

- **Recipe Detail Page**
  - Large recipe image gallery
  - Complete ingredient list
  - Step-by-step instructions
  - Nutrition information
  - Serving size adjuster
  - Cook time breakdown
  - User reviews and ratings
  - Related recipes
  - Print button
  - Share button

### 3. Favorites & Collections
- **Favorites Management**
  - Save/unsave recipes
  - View all favorites
  - Create custom collections
  - Organize by categories
  - Search within favorites
  - Export favorites

- **Collections**
  - Create named collections
  - Add recipes to multiple collections
  - Collection covers
  - Share collections

### 4. Meal Planning
- **Weekly Planner**
  - Calendar view
  - Drag-and-drop recipes to days
  - Meal slots (breakfast, lunch, dinner, snacks)
  - Weekly overview
  - Print meal plan

- **Shopping List**
  - Auto-generate from meal plan
  - Manual additions
  - Group by category
  - Check off items
  - Export/print list
  - Share list

### 5. Additional Features
- Recipe scaling (adjust servings)
- Unit conversion
- Cooking timer
- Notes and substitutions
- Recipe rating and reviews
- Nutritional calculator

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest recipe-finder -- --template react-ts
cd recipe-finder

npm install @tanstack/react-query axios
npm install react-router-dom zustand
npm install lucide-react
npm install react-big-calendar date-fns
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/recipe.ts
export interface Recipe {
  id: number;
  title: string;
  image: string;
  imageType?: string;
  servings: number;
  readyInMinutes: number;
  cookingMinutes?: number;
  preparationMinutes?: number;
  pricePerServing?: number;
  summary: string;
  cuisines: string[];
  dishTypes: string[];
  diets: string[];
  occasions?: string[];
  instructions: string;
  analyzedInstructions: AnalyzedInstruction[];
  extendedIngredients: Ingredient[];
  nutrition?: Nutrition;
  healthScore?: number;
  spoonacularScore?: number;
  veryPopular?: boolean;
  cheap?: boolean;
  veryHealthy?: boolean;
  sustainable?: boolean;
  aggregateLikes?: number;
  creditsText?: string;
}

export interface Ingredient {
  id: number;
  name: string;
  original: string;
  amount: number;
  unit: string;
  measures: {
    us: Measure;
    metric: Measure;
  };
  aisle?: string;
  image?: string;
}

export interface Measure {
  amount: number;
  unitLong: string;
  unitShort: string;
}

export interface AnalyzedInstruction {
  name: string;
  steps: Step[];
}

export interface Step {
  number: number;
  step: string;
  ingredients: { id: number; name: string; image: string }[];
  equipment: { id: number; name: string; image: string }[];
  length?: {
    number: number;
    unit: string;
  };
}

export interface Nutrition {
  nutrients: Nutrient[];
  ingredients: IngredientNutrition[];
}

export interface Nutrient {
  name: string;
  amount: number;
  unit: string;
  percentOfDailyNeeds?: number;
}

export interface IngredientNutrition {
  id: number;
  name: string;
  amount: number;
  unit: string;
  nutrients: Nutrient[];
}

export interface SearchFilters {
  query: string;
  cuisine?: string;
  diet?: string;
  mealType?: string;
  maxReadyTime?: number;
  minCalories?: number;
  maxCalories?: number;
  excludeIngredients?: string[];
  sort?: 'popularity' | 'time' | 'calories' | 'healthiness';
}

export interface RecipeCollection {
  id: string;
  name: string;
  description?: string;
  recipes: number[];
  coverImage?: string;
  createdAt: string;
  updatedAt: string;
}

export interface MealPlanDay {
  date: string;
  meals: {
    breakfast?: Recipe[];
    lunch?: Recipe[];
    dinner?: Recipe[];
    snacks?: Recipe[];
  };
}

export interface ShoppingListItem {
  id: string;
  name: string;
  amount: number;
  unit: string;
  category: string;
  checked: boolean;
  recipeId?: number;
}
```

### Step 3: Create Recipe API Service
```typescript
// src/services/recipeApi.ts
import axios from 'axios';
import type { Recipe, SearchFilters } from '@/types/recipe';

const API_KEY = import.meta.env.VITE_SPOONACULAR_API_KEY;
const BASE_URL = 'https://api.spoonacular.com/recipes';

export const recipeApi = {
  searchRecipes: async (filters: SearchFilters, offset = 0) => {
    const params: any = {
      apiKey: API_KEY,
      query: filters.query,
      number: 12,
      offset,
      addRecipeInformation: true,
      fillIngredients: true,
    };

    if (filters.cuisine) params.cuisine = filters.cuisine;
    if (filters.diet) params.diet = filters.diet;
    if (filters.mealType) params.type = filters.mealType;
    if (filters.maxReadyTime) params.maxReadyTime = filters.maxReadyTime;
    if (filters.minCalories) params.minCalories = filters.minCalories;
    if (filters.maxCalories) params.maxCalories = filters.maxCalories;
    if (filters.excludeIngredients) {
      params.excludeIngredients = filters.excludeIngredients.join(',');
    }
    if (filters.sort) params.sort = filters.sort;

    const response = await axios.get(`${BASE_URL}/complexSearch`, { params });
    return response.data;
  },

  getRecipeById: async (id: number): Promise<Recipe> => {
    const response = await axios.get(`${BASE_URL}/${id}/information`, {
      params: {
        apiKey: API_KEY,
        includeNutrition: true,
      },
    });
    return response.data;
  },

  getRandomRecipes: async (number = 9) => {
    const response = await axios.get(`${BASE_URL}/random`, {
      params: {
        apiKey: API_KEY,
        number,
      },
    });
    return response.data.recipes;
  },

  getSimilarRecipes: async (id: number, number = 6) => {
    const response = await axios.get(`${BASE_URL}/${id}/similar`, {
      params: {
        apiKey: API_KEY,
        number,
      },
    });
    return response.data;
  },

  autocomplete: async (query: string) => {
    const response = await axios.get(`${BASE_URL}/autocomplete`, {
      params: {
        apiKey: API_KEY,
        query,
        number: 5,
      },
    });
    return response.data;
  },
};
```

### Step 4: Create Zustand Store
```typescript
// src/store/recipeStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { RecipeCollection, MealPlanDay, ShoppingListItem } from '@/types/recipe';

interface RecipeStore {
  favorites: number[];
  collections: RecipeCollection[];
  mealPlan: MealPlanDay[];
  shoppingList: ShoppingListItem[];

  toggleFavorite: (recipeId: number) => void;
  isFavorite: (recipeId: number) => boolean;

  createCollection: (name: string, description?: string) => void;
  addToCollection: (collectionId: string, recipeId: number) => void;
  removeFromCollection: (collectionId: string, recipeId: number) => void;
  deleteCollection: (collectionId: string) => void;

  addToMealPlan: (date: string, mealType: string, recipeId: number) => void;
  removeFromMealPlan: (date: string, mealType: string, recipeId: number) => void;
  getMealPlanForDate: (date: string) => MealPlanDay | undefined;

  addToShoppingList: (item: Omit<ShoppingListItem, 'id'>) => void;
  removeFromShoppingList: (itemId: string) => void;
  toggleShoppingItem: (itemId: string) => void;
  clearShoppingList: () => void;
  generateShoppingList: (recipes: any[]) => void;
}

export const useRecipeStore = create<RecipeStore>()(
  persist(
    (set, get) => ({
      favorites: [],
      collections: [],
      mealPlan: [],
      shoppingList: [],

      toggleFavorite: (recipeId) => {
        set((state) => ({
          favorites: state.favorites.includes(recipeId)
            ? state.favorites.filter((id) => id !== recipeId)
            : [...state.favorites, recipeId],
        }));
      },

      isFavorite: (recipeId) => {
        return get().favorites.includes(recipeId);
      },

      createCollection: (name, description) => {
        const newCollection: RecipeCollection = {
          id: Date.now().toString(),
          name,
          description,
          recipes: [],
          createdAt: new Date().toISOString(),
          updatedAt: new Date().toISOString(),
        };
        set((state) => ({
          collections: [...state.collections, newCollection],
        }));
      },

      addToCollection: (collectionId, recipeId) => {
        set((state) => ({
          collections: state.collections.map((col) =>
            col.id === collectionId
              ? {
                  ...col,
                  recipes: col.recipes.includes(recipeId)
                    ? col.recipes
                    : [...col.recipes, recipeId],
                  updatedAt: new Date().toISOString(),
                }
              : col
          ),
        }));
      },

      removeFromCollection: (collectionId, recipeId) => {
        set((state) => ({
          collections: state.collections.map((col) =>
            col.id === collectionId
              ? {
                  ...col,
                  recipes: col.recipes.filter((id) => id !== recipeId),
                  updatedAt: new Date().toISOString(),
                }
              : col
          ),
        }));
      },

      deleteCollection: (collectionId) => {
        set((state) => ({
          collections: state.collections.filter((col) => col.id !== collectionId),
        }));
      },

      addToMealPlan: (date, mealType, recipeId) => {
        // Implementation here
      },

      removeFromMealPlan: (date, mealType, recipeId) => {
        // Implementation here
      },

      getMealPlanForDate: (date) => {
        return get().mealPlan.find((day) => day.date === date);
      },

      addToShoppingList: (item) => {
        set((state) => ({
          shoppingList: [
            ...state.shoppingList,
            { ...item, id: Date.now().toString(), checked: false },
          ],
        }));
      },

      removeFromShoppingList: (itemId) => {
        set((state) => ({
          shoppingList: state.shoppingList.filter((item) => item.id !== itemId),
        }));
      },

      toggleShoppingItem: (itemId) => {
        set((state) => ({
          shoppingList: state.shoppingList.map((item) =>
            item.id === itemId ? { ...item, checked: !item.checked } : item
          ),
        }));
      },

      clearShoppingList: () => {
        set({ shoppingList: [] });
      },

      generateShoppingList: (recipes) => {
        const items: ShoppingListItem[] = [];
        recipes.forEach((recipe) => {
          recipe.extendedIngredients?.forEach((ingredient: any) => {
            items.push({
              id: `${recipe.id}-${ingredient.id}`,
              name: ingredient.name,
              amount: ingredient.amount,
              unit: ingredient.unit,
              category: ingredient.aisle || 'Other',
              checked: false,
              recipeId: recipe.id,
            });
          });
        });
        set({ shoppingList: items });
      },
    }),
    {
      name: 'recipe-storage',
    }
  )
);
```

### Step 5: Create Recipe Card Component
```typescript
// src/components/RecipeCard.tsx
import { Heart, Clock, Users, ChefHat } from 'lucide-react';
import { Link } from 'react-router-dom';
import type { Recipe } from '@/types/recipe';
import { useRecipeStore } from '@/store/recipeStore';

interface RecipeCardProps {
  recipe: Recipe;
}

export const RecipeCard: React.FC<RecipeCardProps> = ({ recipe }) => {
  const { isFavorite, toggleFavorite } = useRecipeStore();
  const favorite = isFavorite(recipe.id);

  return (
    <div className="group relative bg-white rounded-xl shadow-sm hover:shadow-xl transition-shadow overflow-hidden">
      {/* Image */}
      <Link to={`/recipe/${recipe.id}`} className="block relative aspect-[4/3]">
        <img
          src={recipe.image}
          alt={recipe.title}
          className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
        />

        {/* Overlay badges */}
        <div className="absolute top-3 left-3 flex gap-2">
          {recipe.veryHealthy && (
            <span className="bg-green-500 text-white text-xs px-2 py-1 rounded-full font-semibold">
              Healthy
            </span>
          )}
          {recipe.veryPopular && (
            <span className="bg-orange-500 text-white text-xs px-2 py-1 rounded-full font-semibold">
              Popular
            </span>
          )}
        </div>

        {/* Favorite button */}
        <button
          onClick={(e) => {
            e.preventDefault();
            toggleFavorite(recipe.id);
          }}
          className="absolute top-3 right-3 bg-white/90 backdrop-blur-sm p-2 rounded-full hover:bg-white transition-colors"
        >
          <Heart
            className={`w-5 h-5 ${
              favorite ? 'fill-red-500 text-red-500' : 'text-gray-600'
            }`}
          />
        </button>
      </Link>

      {/* Content */}
      <div className="p-4">
        <Link to={`/recipe/${recipe.id}`}>
          <h3 className="font-bold text-lg line-clamp-2 group-hover:text-blue-600 transition-colors mb-2">
            {recipe.title}
          </h3>
        </Link>

        {/* Diet tags */}
        {recipe.diets.length > 0 && (
          <div className="flex gap-1 mb-3 flex-wrap">
            {recipe.diets.slice(0, 2).map((diet) => (
              <span
                key={diet}
                className="text-xs bg-blue-50 text-blue-700 px-2 py-1 rounded-full capitalize"
              >
                {diet}
              </span>
            ))}
          </div>
        )}

        {/* Meta info */}
        <div className="flex items-center justify-between text-sm text-gray-600">
          <div className="flex items-center gap-1">
            <Clock className="w-4 h-4" />
            <span>{recipe.readyInMinutes} min</span>
          </div>

          <div className="flex items-center gap-1">
            <Users className="w-4 h-4" />
            <span>{recipe.servings} servings</span>
          </div>

          {recipe.healthScore && (
            <div className="flex items-center gap-1">
              <ChefHat className="w-4 h-4" />
              <span>{recipe.healthScore}%</span>
            </div>
          )}
        </div>

        {/* Cuisines */}
        {recipe.cuisines.length > 0 && (
          <p className="text-xs text-gray-500 mt-2 capitalize">
            {recipe.cuisines.join(', ')}
          </p>
        )}
      </div>
    </div>
  );
};
```

### Step 6: Create Search Page
```typescript
// src/pages/SearchPage.tsx
import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Search, Filter, X } from 'lucide-react';
import { recipeApi } from '@/services/recipeApi';
import { RecipeCard } from '@/components/RecipeCard';
import type { SearchFilters } from '@/types/recipe';

export const SearchPage: React.FC = () => {
  const [filters, setFilters] = useState<SearchFilters>({
    query: '',
  });
  const [showFilters, setShowFilters] = useState(false);

  const { data, isLoading, error } = useQuery({
    queryKey: ['recipes', 'search', filters],
    queryFn: () => recipeApi.searchRecipes(filters),
    enabled: filters.query.length > 0,
  });

  const cuisines = [
    'Italian', 'Mexican', 'Asian', 'American', 'Mediterranean',
    'Indian', 'Chinese', 'Japanese', 'Thai', 'French'
  ];

  const diets = [
    'Vegetarian', 'Vegan', 'Gluten Free', 'Ketogenic',
    'Paleo', 'Dairy Free', 'Whole 30'
  ];

  const mealTypes = ['Breakfast', 'Lunch', 'Dinner', 'Snack', 'Dessert'];

  return (
    <div className="max-w-7xl mx-auto px-4 py-8">
      {/* Search Header */}
      <div className="mb-8">
        <h1 className="text-4xl font-bold mb-6">Find Your Perfect Recipe</h1>

        {/* Search Bar */}
        <div className="flex gap-4">
          <div className="flex-1 relative">
            <Search className="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400 w-5 h-5" />
            <input
              type="text"
              placeholder="Search recipes..."
              value={filters.query}
              onChange={(e) => setFilters({ ...filters, query: e.target.value })}
              className="w-full pl-12 pr-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
          </div>
          <button
            onClick={() => setShowFilters(!showFilters)}
            className="flex items-center gap-2 px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            <Filter className="w-5 h-5" />
            Filters
          </button>
        </div>

        {/* Active Filters */}
        <div className="flex gap-2 mt-4 flex-wrap">
          {filters.cuisine && (
            <span className="flex items-center gap-2 bg-blue-100 text-blue-800 px-3 py-1 rounded-full text-sm">
              {filters.cuisine}
              <button onClick={() => setFilters({ ...filters, cuisine: undefined })}>
                <X className="w-3 h-3" />
              </button>
            </span>
          )}
          {filters.diet && (
            <span className="flex items-center gap-2 bg-green-100 text-green-800 px-3 py-1 rounded-full text-sm">
              {filters.diet}
              <button onClick={() => setFilters({ ...filters, diet: undefined })}>
                <X className="w-3 h-3" />
              </button>
            </span>
          )}
          {filters.mealType && (
            <span className="flex items-center gap-2 bg-purple-100 text-purple-800 px-3 py-1 rounded-full text-sm">
              {filters.mealType}
              <button onClick={() => setFilters({ ...filters, mealType: undefined })}>
                <X className="w-3 h-3" />
              </button>
            </span>
          )}
        </div>
      </div>

      <div className="grid lg:grid-cols-4 gap-8">
        {/* Filters Sidebar */}
        {showFilters && (
          <div className="lg:col-span-1 bg-white rounded-xl shadow-sm p-6 h-fit">
            <h3 className="font-bold text-lg mb-4">Filters</h3>

            {/* Cuisine */}
            <div className="mb-6">
              <h4 className="font-semibold mb-2">Cuisine</h4>
              <div className="space-y-2">
                {cuisines.map((cuisine) => (
                  <label key={cuisine} className="flex items-center gap-2">
                    <input
                      type="radio"
                      name="cuisine"
                      checked={filters.cuisine === cuisine}
                      onChange={() => setFilters({ ...filters, cuisine })}
                    />
                    <span className="text-sm">{cuisine}</span>
                  </label>
                ))}
              </div>
            </div>

            {/* Diet */}
            <div className="mb-6">
              <h4 className="font-semibold mb-2">Diet</h4>
              <div className="space-y-2">
                {diets.map((diet) => (
                  <label key={diet} className="flex items-center gap-2">
                    <input
                      type="radio"
                      name="diet"
                      checked={filters.diet === diet}
                      onChange={() => setFilters({ ...filters, diet })}
                    />
                    <span className="text-sm">{diet}</span>
                  </label>
                ))}
              </div>
            </div>

            {/* Meal Type */}
            <div className="mb-6">
              <h4 className="font-semibold mb-2">Meal Type</h4>
              <div className="space-y-2">
                {mealTypes.map((type) => (
                  <label key={type} className="flex items-center gap-2">
                    <input
                      type="radio"
                      name="mealType"
                      checked={filters.mealType === type}
                      onChange={() => setFilters({ ...filters, mealType: type })}
                    />
                    <span className="text-sm">{type}</span>
                  </label>
                ))}
              </div>
            </div>

            {/* Max Cook Time */}
            <div className="mb-6">
              <h4 className="font-semibold mb-2">Max Cook Time</h4>
              <input
                type="range"
                min="15"
                max="120"
                step="15"
                value={filters.maxReadyTime || 120}
                onChange={(e) =>
                  setFilters({ ...filters, maxReadyTime: parseInt(e.target.value) })
                }
                className="w-full"
              />
              <div className="text-sm text-gray-600 mt-1">
                {filters.maxReadyTime || 120} minutes
              </div>
            </div>

            <button
              onClick={() =>
                setFilters({ query: filters.query })
              }
              className="w-full py-2 text-sm text-gray-600 hover:text-gray-900"
            >
              Clear all filters
            </button>
          </div>
        )}

        {/* Results */}
        <div className={showFilters ? 'lg:col-span-3' : 'lg:col-span-4'}>
          {isLoading && (
            <div className="text-center py-12">
              <div className="inline-block w-8 h-8 border-4 border-blue-600 border-t-transparent rounded-full animate-spin" />
            </div>
          )}

          {error && (
            <div className="text-center py-12 text-red-500">
              Error loading recipes. Please try again.
            </div>
          )}

          {data && data.results.length === 0 && (
            <div className="text-center py-12 text-gray-500">
              No recipes found. Try different search terms or filters.
            </div>
          )}

          {data && data.results.length > 0 && (
            <>
              <p className="text-gray-600 mb-4">
                Found {data.totalResults} recipes
              </p>
              <div className="grid md:grid-cols-2 lg:grid-cols-3 gap-6">
                {data.results.map((recipe: any) => (
                  <RecipeCard key={recipe.id} recipe={recipe} />
                ))}
              </div>
            </>
          )}
        </div>
      </div>
    </div>
  );
};
```

## Expected Outputs

1. **Complete Recipe Discovery App** with:
   - Advanced search with autocomplete
   - Multiple filter options
   - Recipe cards with rich information
   - Detailed recipe views

2. **Favorites & Collections**:
   - Save favorite recipes
   - Create custom collections
   - Organize recipes by category
   - Persistent storage

3. **Meal Planning**:
   - Weekly meal planner
   - Drag-and-drop interface
   - Shopping list generation
   - Print functionality

4. **User Experience**:
   - Responsive design
   - Fast search results
   - Smooth animations
   - Intuitive navigation

## Bonus Challenges

- [ ] Add recipe rating and review system
- [ ] Implement recipe scaling with unit conversion
- [ ] Add cooking timer with notifications
- [ ] Create custom recipe addition
- [ ] Add ingredient substitution suggestions
- [ ] Implement barcode scanner for ingredients
- [ ] Add nutrition tracking
- [ ] Create grocery store integration
- [ ] Add voice-guided cooking mode
- [ ] Implement recipe sharing via link
- [ ] Add meal prep calculator
- [ ] Create dietary goal tracking
- [ ] Add wine/beverage pairing suggestions
- [ ] Implement leftover recipe suggestions

## Resources

- [Spoonacular API](https://spoonacular.com/food-api)
- [Edamam Recipe API](https://developer.edamam.com/)
- [React Big Calendar](https://jquense.github.io/react-big-calendar/)
- [React Query Documentation](https://tanstack.com/query/latest)
- [Zustand Persist Middleware](https://docs.pmnd.rs/zustand/integrations/persisting-store-data)

## Success Criteria

- Search returns relevant recipes
- Filters work individually and in combination
- Favorites persist across sessions
- Collections can be created and managed
- Meal planner is functional and intuitive
- Shopping list generates correctly
- Recipe details display all information
- Images load properly
- Mobile responsive on all screens
- Loading states are informative
- Error handling covers edge cases
- TypeScript types are comprehensive
- API rate limits are respected
