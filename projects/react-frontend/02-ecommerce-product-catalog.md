# Project 2: E-Commerce Product Catalog with Filters and Cart

## Overview
Build a modern e-commerce product catalog with advanced filtering, sorting, shopping cart functionality, and checkout flow. This project focuses on complex state management, URL-based filtering, and creating a seamless shopping experience.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Implement advanced filtering and sorting logic
- Manage shopping cart state with Context API or Zustand
- Create URL-based filter state for shareable links
- Build responsive product grid layouts
- Handle pagination or infinite scroll
- Implement local storage persistence
- Create smooth animations and transitions
- Optimize performance with React.memo and useMemo

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **State Management**: Zustand or Context API + useReducer
- **Routing**: React Router v6 with URL search params
- **Styling**: Tailwind CSS with HeadlessUI
- **API**: FakeStore API or DummyJSON
- **Icons**: Lucide React
- **Animations**: Framer Motion
- **Build Tool**: Vite
- **Storage**: localStorage for cart persistence

## Project Requirements

### 1. Product Catalog Features
- **Product Grid/List View**
  - Responsive grid layout (1/2/3/4 columns)
  - Product cards with image, title, price, rating
  - Quick view modal
  - Add to cart button with quantity selector
  - List/grid view toggle

- **Filtering System**
  - Filter by category
  - Filter by price range (slider)
  - Filter by rating
  - Filter by availability
  - Multiple filter combination
  - Active filters display with remove option
  - Clear all filters button

- **Sorting Options**
  - Sort by price (low to high, high to low)
  - Sort by rating
  - Sort by popularity
  - Sort by newest

- **Search Functionality**
  - Real-time search with debouncing
  - Search highlighting in results
  - Search suggestions

### 2. Shopping Cart
- **Cart Features**
  - Add/remove items
  - Update quantities
  - Display total price with tax
  - Cart icon with item count badge
  - Persistent cart (localStorage)
  - Cart sidebar/drawer
  - Empty cart state

- **Cart Calculations**
  - Subtotal
  - Tax calculation
  - Shipping cost
  - Discount/coupon codes
  - Final total

### 3. Product Details
- Image gallery with thumbnails
- Product specifications
- Stock availability
- Related products
- Add to favorites/wishlist
- Share product

### 4. UI/UX Components
- Loading skeletons
- Empty states
- Error boundaries
- Toast notifications
- Breadcrumb navigation
- Pagination or infinite scroll

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest ecommerce-catalog -- --template react-ts
cd ecommerce-catalog

npm install react-router-dom zustand
npm install framer-motion lucide-react
npm install @headlessui/react
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/product.ts
export interface Product {
  id: number;
  title: string;
  description: string;
  price: number;
  discountPercentage?: number;
  rating: number;
  stock: number;
  brand: string;
  category: string;
  thumbnail: string;
  images: string[];
}

export interface CartItem {
  product: Product;
  quantity: number;
}

export interface Filters {
  categories: string[];
  priceRange: [number, number];
  minRating: number;
  inStock: boolean;
  search: string;
}

export type SortOption = 'price-asc' | 'price-desc' | 'rating' | 'popular';

export interface ProductsResponse {
  products: Product[];
  total: number;
  skip: number;
  limit: number;
}
```

### Step 3: Create Zustand Store for Cart
```typescript
// src/store/cartStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { Product, CartItem } from '../types/product';

interface CartStore {
  items: CartItem[];
  addItem: (product: Product, quantity?: number) => void;
  removeItem: (productId: number) => void;
  updateQuantity: (productId: number, quantity: number) => void;
  clearCart: () => void;
  getTotalItems: () => number;
  getTotalPrice: () => number;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (product, quantity = 1) => {
        set((state) => {
          const existingItem = state.items.find(
            (item) => item.product.id === product.id
          );

          if (existingItem) {
            return {
              items: state.items.map((item) =>
                item.product.id === product.id
                  ? { ...item, quantity: item.quantity + quantity }
                  : item
              ),
            };
          }

          return {
            items: [...state.items, { product, quantity }],
          };
        });
      },

      removeItem: (productId) => {
        set((state) => ({
          items: state.items.filter((item) => item.product.id !== productId),
        }));
      },

      updateQuantity: (productId, quantity) => {
        if (quantity <= 0) {
          get().removeItem(productId);
          return;
        }

        set((state) => ({
          items: state.items.map((item) =>
            item.product.id === productId ? { ...item, quantity } : item
          ),
        }));
      },

      clearCart: () => set({ items: [] }),

      getTotalItems: () => {
        return get().items.reduce((total, item) => total + item.quantity, 0);
      },

      getTotalPrice: () => {
        return get().items.reduce(
          (total, item) => total + item.product.price * item.quantity,
          0
        );
      },
    }),
    {
      name: 'cart-storage',
    }
  )
);
```

### Step 4: Create Products API Hook
```typescript
// src/hooks/useProducts.ts
import { useState, useEffect, useMemo } from 'react';
import type { Product, Filters, SortOption } from '../types/product';

export const useProducts = (filters: Filters, sortBy: SortOption) => {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchProducts = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch('https://dummyjson.com/products?limit=100');
        const data = await response.json();
        setProducts(data.products);
      } catch (err) {
        setError('Failed to fetch products');
      } finally {
        setLoading(false);
      }
    };

    fetchProducts();
  }, []);

  const filteredAndSortedProducts = useMemo(() => {
    let result = [...products];

    // Apply search filter
    if (filters.search) {
      const searchLower = filters.search.toLowerCase();
      result = result.filter(
        (p) =>
          p.title.toLowerCase().includes(searchLower) ||
          p.description.toLowerCase().includes(searchLower) ||
          p.brand.toLowerCase().includes(searchLower)
      );
    }

    // Apply category filter
    if (filters.categories.length > 0) {
      result = result.filter((p) => filters.categories.includes(p.category));
    }

    // Apply price range filter
    result = result.filter(
      (p) => p.price >= filters.priceRange[0] && p.price <= filters.priceRange[1]
    );

    // Apply rating filter
    result = result.filter((p) => p.rating >= filters.minRating);

    // Apply stock filter
    if (filters.inStock) {
      result = result.filter((p) => p.stock > 0);
    }

    // Apply sorting
    switch (sortBy) {
      case 'price-asc':
        result.sort((a, b) => a.price - b.price);
        break;
      case 'price-desc':
        result.sort((a, b) => b.price - a.price);
        break;
      case 'rating':
        result.sort((a, b) => b.rating - a.rating);
        break;
      case 'popular':
        // Could use stock or rating as popularity metric
        result.sort((a, b) => b.rating - a.rating);
        break;
    }

    return result;
  }, [products, filters, sortBy]);

  return { products: filteredAndSortedProducts, loading, error };
};

export const useCategories = (products: Product[]) => {
  return useMemo(() => {
    const categories = new Set(products.map((p) => p.category));
    return Array.from(categories);
  }, [products]);
};
```

### Step 5: Create Product Card Component
```typescript
// src/components/ProductCard.tsx
import { motion } from 'framer-motion';
import { ShoppingCart, Star, Eye } from 'lucide-react';
import { useState } from 'react';
import type { Product } from '../types/product';
import { useCartStore } from '../store/cartStore';

interface ProductCardProps {
  product: Product;
  onQuickView: (product: Product) => void;
}

export const ProductCard: React.FC<ProductCardProps> = ({
  product,
  onQuickView,
}) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  const addItem = useCartStore((state) => state.addItem);

  const handleAddToCart = (e: React.MouseEvent) => {
    e.preventDefault();
    addItem(product);
    // Show toast notification
  };

  const discountedPrice = product.discountPercentage
    ? product.price * (1 - product.discountPercentage / 100)
    : product.price;

  return (
    <motion.div
      layout
      initial={{ opacity: 0, scale: 0.9 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.9 }}
      className="group relative bg-white rounded-lg shadow-sm hover:shadow-xl transition-shadow duration-300 overflow-hidden"
    >
      {/* Product Image */}
      <div className="relative aspect-square overflow-hidden bg-gray-100">
        {!imageLoaded && (
          <div className="absolute inset-0 animate-pulse bg-gray-200" />
        )}
        <img
          src={product.thumbnail}
          alt={product.title}
          className="w-full h-full object-cover group-hover:scale-110 transition-transform duration-300"
          onLoad={() => setImageLoaded(true)}
        />

        {/* Discount Badge */}
        {product.discountPercentage && product.discountPercentage > 0 && (
          <div className="absolute top-2 right-2 bg-red-500 text-white px-2 py-1 rounded-md text-sm font-semibold">
            -{product.discountPercentage}%
          </div>
        )}

        {/* Quick View Button */}
        <button
          onClick={() => onQuickView(product)}
          className="absolute inset-0 bg-black/0 group-hover:bg-black/20 flex items-center justify-center opacity-0 group-hover:opacity-100 transition-all"
        >
          <div className="bg-white rounded-full p-3">
            <Eye className="w-5 h-5" />
          </div>
        </button>

        {/* Stock Badge */}
        {product.stock === 0 && (
          <div className="absolute inset-0 bg-black/50 flex items-center justify-center">
            <span className="bg-white text-red-600 px-4 py-2 rounded-md font-semibold">
              Out of Stock
            </span>
          </div>
        )}
      </div>

      {/* Product Info */}
      <div className="p-4">
        <p className="text-xs text-gray-500 uppercase tracking-wide">
          {product.brand}
        </p>
        <h3 className="font-semibold text-gray-900 mt-1 line-clamp-2 min-h-[2.5rem]">
          {product.title}
        </h3>

        {/* Rating */}
        <div className="flex items-center gap-1 mt-2">
          <Star className="w-4 h-4 fill-yellow-400 text-yellow-400" />
          <span className="text-sm font-medium">{product.rating.toFixed(1)}</span>
          <span className="text-sm text-gray-500">({product.stock} in stock)</span>
        </div>

        {/* Price */}
        <div className="mt-3 flex items-baseline gap-2">
          <span className="text-xl font-bold text-gray-900">
            ${discountedPrice.toFixed(2)}
          </span>
          {product.discountPercentage && product.discountPercentage > 0 && (
            <span className="text-sm text-gray-500 line-through">
              ${product.price.toFixed(2)}
            </span>
          )}
        </div>

        {/* Add to Cart Button */}
        <button
          onClick={handleAddToCart}
          disabled={product.stock === 0}
          className="mt-4 w-full bg-blue-600 text-white py-2 rounded-md hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed flex items-center justify-center gap-2 transition-colors"
        >
          <ShoppingCart className="w-4 h-4" />
          Add to Cart
        </button>
      </div>
    </motion.div>
  );
};
```

### Step 6: Create Filter Sidebar
```typescript
// src/components/FilterSidebar.tsx
import { useState } from 'react';
import { X } from 'lucide-react';
import type { Filters } from '../types/product';

interface FilterSidebarProps {
  filters: Filters;
  onFiltersChange: (filters: Filters) => void;
  categories: string[];
  maxPrice: number;
}

export const FilterSidebar: React.FC<FilterSidebarProps> = ({
  filters,
  onFiltersChange,
  categories,
  maxPrice,
}) => {
  const handleCategoryToggle = (category: string) => {
    const newCategories = filters.categories.includes(category)
      ? filters.categories.filter((c) => c !== category)
      : [...filters.categories, category];

    onFiltersChange({ ...filters, categories: newCategories });
  };

  const handlePriceChange = (index: 0 | 1, value: number) => {
    const newRange: [number, number] = [...filters.priceRange];
    newRange[index] = value;
    onFiltersChange({ ...filters, priceRange: newRange });
  };

  const clearFilters = () => {
    onFiltersChange({
      categories: [],
      priceRange: [0, maxPrice],
      minRating: 0,
      inStock: false,
      search: '',
    });
  };

  const hasActiveFilters =
    filters.categories.length > 0 ||
    filters.priceRange[0] > 0 ||
    filters.priceRange[1] < maxPrice ||
    filters.minRating > 0 ||
    filters.inStock;

  return (
    <div className="bg-white rounded-lg shadow p-6 sticky top-4">
      <div className="flex items-center justify-between mb-6">
        <h2 className="text-lg font-bold">Filters</h2>
        {hasActiveFilters && (
          <button
            onClick={clearFilters}
            className="text-sm text-blue-600 hover:text-blue-800"
          >
            Clear All
          </button>
        )}
      </div>

      {/* Categories */}
      <div className="mb-6">
        <h3 className="font-semibold mb-3">Categories</h3>
        <div className="space-y-2">
          {categories.map((category) => (
            <label key={category} className="flex items-center gap-2 cursor-pointer">
              <input
                type="checkbox"
                checked={filters.categories.includes(category)}
                onChange={() => handleCategoryToggle(category)}
                className="rounded border-gray-300"
              />
              <span className="text-sm capitalize">{category}</span>
            </label>
          ))}
        </div>
      </div>

      {/* Price Range */}
      <div className="mb-6">
        <h3 className="font-semibold mb-3">Price Range</h3>
        <div className="space-y-3">
          <input
            type="range"
            min={0}
            max={maxPrice}
            value={filters.priceRange[1]}
            onChange={(e) => handlePriceChange(1, Number(e.target.value))}
            className="w-full"
          />
          <div className="flex items-center justify-between text-sm">
            <span>${filters.priceRange[0]}</span>
            <span>${filters.priceRange[1]}</span>
          </div>
        </div>
      </div>

      {/* Rating */}
      <div className="mb-6">
        <h3 className="font-semibold mb-3">Minimum Rating</h3>
        <div className="space-y-2">
          {[4, 3, 2, 1].map((rating) => (
            <label key={rating} className="flex items-center gap-2 cursor-pointer">
              <input
                type="radio"
                name="rating"
                checked={filters.minRating === rating}
                onChange={() =>
                  onFiltersChange({ ...filters, minRating: rating })
                }
                className="border-gray-300"
              />
              <span className="text-sm">{rating}+ Stars</span>
            </label>
          ))}
        </div>
      </div>

      {/* In Stock */}
      <div>
        <label className="flex items-center gap-2 cursor-pointer">
          <input
            type="checkbox"
            checked={filters.inStock}
            onChange={(e) =>
              onFiltersChange({ ...filters, inStock: e.target.checked })
            }
            className="rounded border-gray-300"
          />
          <span className="text-sm font-medium">In Stock Only</span>
        </label>
      </div>
    </div>
  );
};
```

### Step 7: Create Shopping Cart Component
```typescript
// src/components/ShoppingCart.tsx
import { Fragment } from 'react';
import { Dialog, Transition } from '@headlessui/react';
import { X, ShoppingBag, Minus, Plus, Trash2 } from 'lucide-react';
import { useCartStore } from '../store/cartStore';

interface ShoppingCartProps {
  isOpen: boolean;
  onClose: () => void;
}

export const ShoppingCart: React.FC<ShoppingCartProps> = ({
  isOpen,
  onClose,
}) => {
  const { items, updateQuantity, removeItem, getTotalPrice, clearCart } =
    useCartStore();

  const subtotal = getTotalPrice();
  const tax = subtotal * 0.1; // 10% tax
  const shipping = subtotal > 50 ? 0 : 5.99;
  const total = subtotal + tax + shipping;

  return (
    <Transition.Root show={isOpen} as={Fragment}>
      <Dialog as="div" className="relative z-50" onClose={onClose}>
        <Transition.Child
          as={Fragment}
          enter="ease-in-out duration-300"
          enterFrom="opacity-0"
          enterTo="opacity-100"
          leave="ease-in-out duration-300"
          leaveFrom="opacity-100"
          leaveTo="opacity-0"
        >
          <div className="fixed inset-0 bg-black bg-opacity-50 transition-opacity" />
        </Transition.Child>

        <div className="fixed inset-0 overflow-hidden">
          <div className="absolute inset-0 overflow-hidden">
            <div className="pointer-events-none fixed inset-y-0 right-0 flex max-w-full pl-10">
              <Transition.Child
                as={Fragment}
                enter="transform transition ease-in-out duration-300"
                enterFrom="translate-x-full"
                enterTo="translate-x-0"
                leave="transform transition ease-in-out duration-300"
                leaveFrom="translate-x-0"
                leaveTo="translate-x-full"
              >
                <Dialog.Panel className="pointer-events-auto w-screen max-w-md">
                  <div className="flex h-full flex-col bg-white shadow-xl">
                    {/* Header */}
                    <div className="flex items-center justify-between px-6 py-4 border-b">
                      <div className="flex items-center gap-2">
                        <ShoppingBag className="w-5 h-5" />
                        <Dialog.Title className="text-lg font-semibold">
                          Shopping Cart ({items.length})
                        </Dialog.Title>
                      </div>
                      <button onClick={onClose} className="text-gray-400 hover:text-gray-600">
                        <X className="w-5 h-5" />
                      </button>
                    </div>

                    {/* Cart Items */}
                    <div className="flex-1 overflow-y-auto px-6 py-4">
                      {items.length === 0 ? (
                        <div className="flex flex-col items-center justify-center h-full text-gray-500">
                          <ShoppingBag className="w-16 h-16 mb-4 opacity-20" />
                          <p>Your cart is empty</p>
                        </div>
                      ) : (
                        <div className="space-y-4">
                          {items.map((item) => (
                            <div
                              key={item.product.id}
                              className="flex gap-4 pb-4 border-b"
                            >
                              <img
                                src={item.product.thumbnail}
                                alt={item.product.title}
                                className="w-20 h-20 object-cover rounded"
                              />
                              <div className="flex-1">
                                <h3 className="font-medium text-sm">
                                  {item.product.title}
                                </h3>
                                <p className="text-sm text-gray-500 mt-1">
                                  ${item.product.price.toFixed(2)}
                                </p>
                                <div className="flex items-center gap-2 mt-2">
                                  <button
                                    onClick={() =>
                                      updateQuantity(
                                        item.product.id,
                                        item.quantity - 1
                                      )
                                    }
                                    className="p-1 border rounded hover:bg-gray-100"
                                  >
                                    <Minus className="w-3 h-3" />
                                  </button>
                                  <span className="w-8 text-center text-sm">
                                    {item.quantity}
                                  </span>
                                  <button
                                    onClick={() =>
                                      updateQuantity(
                                        item.product.id,
                                        item.quantity + 1
                                      )
                                    }
                                    className="p-1 border rounded hover:bg-gray-100"
                                  >
                                    <Plus className="w-3 h-3" />
                                  </button>
                                  <button
                                    onClick={() => removeItem(item.product.id)}
                                    className="ml-auto text-red-500 hover:text-red-700"
                                  >
                                    <Trash2 className="w-4 h-4" />
                                  </button>
                                </div>
                              </div>
                              <div className="text-right">
                                <p className="font-semibold">
                                  ${(item.product.price * item.quantity).toFixed(2)}
                                </p>
                              </div>
                            </div>
                          ))}
                        </div>
                      )}
                    </div>

                    {/* Footer */}
                    {items.length > 0 && (
                      <div className="border-t px-6 py-4 space-y-3">
                        <div className="flex justify-between text-sm">
                          <span>Subtotal</span>
                          <span>${subtotal.toFixed(2)}</span>
                        </div>
                        <div className="flex justify-between text-sm">
                          <span>Tax (10%)</span>
                          <span>${tax.toFixed(2)}</span>
                        </div>
                        <div className="flex justify-between text-sm">
                          <span>Shipping</span>
                          <span>{shipping === 0 ? 'FREE' : `$${shipping.toFixed(2)}`}</span>
                        </div>
                        <div className="flex justify-between text-lg font-bold border-t pt-3">
                          <span>Total</span>
                          <span>${total.toFixed(2)}</span>
                        </div>
                        <button className="w-full bg-blue-600 text-white py-3 rounded-md hover:bg-blue-700 font-semibold">
                          Proceed to Checkout
                        </button>
                      </div>
                    )}
                  </div>
                </Dialog.Panel>
              </Transition.Child>
            </div>
          </div>
        </div>
      </Dialog>
    </Transition.Root>
  );
};
```

## Expected Outputs

1. **Fully Functional E-Commerce Catalog** with:
   - Responsive product grid
   - Working filters and sorting
   - Persistent shopping cart
   - Smooth animations

2. **Advanced Features**:
   - URL-based filter state (shareable links)
   - Real-time search with debouncing
   - Optimistic UI updates
   - Loading skeletons

3. **Shopping Experience**:
   - Quick add to cart
   - Cart sidebar with animations
   - Quantity management
   - Price calculations with tax and shipping

4. **Performance**:
   - Optimized re-renders with React.memo
   - Efficient filtering with useMemo
   - Image lazy loading
   - Smooth 60fps animations

## Bonus Challenges

- [ ] Add product comparison feature
- [ ] Implement wishlist/favorites with persistence
- [ ] Add product reviews and ratings system
- [ ] Create order history page
- [ ] Add discount coupon system
- [ ] Implement product variants (size, color)
- [ ] Add recently viewed products
- [ ] Create product recommendations
- [ ] Add image zoom on hover
- [ ] Implement virtual scrolling for large lists
- [ ] Add PWA support for offline cart
- [ ] Create checkout flow with form validation
- [ ] Add payment integration (Stripe test mode)
- [ ] Implement product quick filters (price ranges as chips)

## Resources

- [Zustand Documentation](https://docs.pmnd.rs/zustand/getting-started/introduction)
- [Framer Motion](https://www.framer.com/motion/)
- [HeadlessUI](https://headlessui.com/)
- [DummyJSON API](https://dummyjson.com/)
- [FakeStore API](https://fakestoreapi.com/)
- [React Router URL Params](https://reactrouter.com/en/main/hooks/use-search-params)

## Success Criteria

- Products display correctly in grid layout
- All filters work individually and in combination
- Sorting changes product order correctly
- Cart persists across page refreshes
- Cart calculations are accurate
- URL updates when filters change
- Search results are instant with debouncing
- Animations are smooth without jank
- UI is responsive on mobile, tablet, and desktop
- Loading states are shown appropriately
- Empty states guide users effectively
- No TypeScript errors or warnings
