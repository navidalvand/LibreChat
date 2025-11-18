# Frontend Architecture

> **React application overview for LibreChat v0.8.1-rc1**

## Overview

LibreChat's frontend is a modern **React 18 single-page application** built with **Vite 6**, featuring a hybrid **JavaScript/TypeScript** codebase with extensive UI components, real-time updates, and sophisticated state management.

## Architecture Summary

```
React 18 App (client/src/)
├── Providers (8 layers)
│   ├── ApiErrorBoundaryProvider
│   ├── QueryClientProvider (TanStack Query)
│   ├── RecoilRoot (legacy state)
│   ├── LiveAnnouncer (a11y)
│   ├── ThemeProvider
│   ├── ToastProvider
│   ├── DndProvider (drag & drop)
│   └── ScreenshotProvider
├── Router (React Router v6)
│   ├── Protected Routes
│   ├── Public Routes
│   └── Nested Layouts
├── Components (24 feature domains)
│   ├── Chat/
│   ├── Agents/
│   ├── Messages/
│   ├── Artifacts/
│   └── ... 20 more
├── State Management (hybrid)
│   ├── Jotai Atoms (primary)
│   ├── Recoil Atoms (legacy)
│   └── TanStack Query (server state)
└── Hooks (158 custom hooks)
```

## Source Code References

**Entry Points:**
- React entry: `/home/user/LibreChat/client/src/main.jsx` (lines 1-19)
- App component: `/home/user/LibreChat/client/src/App.jsx` (lines 1-81)
- Routes: `/home/user/LibreChat/client/src/routes/index.tsx`

**Key Directories:**
- Components: `client/src/components/`
- State: `client/src/store/`
- Hooks: `client/src/hooks/`
- Data provider: `client/src/data-provider/`
- Routes: `client/src/routes/`
- Utilities: `client/src/utils/`
- Types: `client/src/@types/`

## Technology Stack

### Core Framework

**React 18.3.1:**
- Concurrent rendering
- Automatic batching
- Transitions API
- Suspense for data fetching

**Vite 6.0.3:**
- Lightning-fast HMR (Hot Module Replacement)
- Native ESM dev server
- Optimized production builds
- Code splitting with chunking strategy

### State Management

**Jotai (Primary):**
- Atomic state management
- Bottom-up approach
- Minimal boilerplate
- TypeScript-first

**Recoil (Legacy, being phased out):**
- Atom/selector pattern
- Still used in some components
- Gradual migration to Jotai

**TanStack Query (React Query v5):**
- Server state management
- Automatic caching
- Background refetching
- Optimistic updates
- Infinite queries for pagination

### UI Components

**Radix UI:**
- Headless, accessible components
- Used: Dialog, DropdownMenu, Popover, Select, Tabs, Toast

**Headless UI:**
- Unstyled, accessible components
- Used: Menu, Transition

**Tailwind CSS 3.4:**
- Utility-first CSS
- Custom theme configuration
- Dark mode support
- Custom color palettes

**Class Variance Authority (CVA):**
- Component variant styling
- Type-safe variants

### Routing

**React Router DOM v6:**
- Nested routes
- Protected routes
- Lazy loading
- Search params management

### Forms & Validation

**React Hook Form:**
- Form state management
- Validation
- Performance optimization

### Code & Markdown

**React Markdown:**
- Markdown rendering
- Custom components
- Remark/Rehype plugins

**CodeMirror:**
- Code editor
- Syntax highlighting
- Extensions

**KaTeX:**
- Math rendering
- LaTeX support

**Highlight.js:**
- Syntax highlighting for code blocks

### Internationalization

**i18next + react-i18next:**
- 30+ languages
- Dynamic language switching
- Locize integration

### Additional Libraries

- **Framer Motion** - Animations
- **react-dnd** - Drag and drop
- **react-speech-recognition** - Speech input
- **heic-convert** - HEIC image conversion
- **lucide-react** - Icons
- **@tanstack/react-virtual** - List virtualization

## Provider Stack

**App.jsx Provider Hierarchy (lines 36-63):**

```javascript
<ApiErrorBoundaryProvider>          // 1. Error boundary
  <QueryClientProvider>              // 2. TanStack Query
    <RecoilRoot>                     // 3. Recoil state
      <LiveAnnouncer>                // 4. Accessibility
        <ThemeProvider>              // 5. Theme/dark mode
          <RadixToast.Provider>      // 6. Toast notifications
            <ToastProvider>          // 7. Custom toast logic
              <DndProvider>          // 8. Drag & drop
                <RouterProvider />   // 9. React Router
              </DndProvider>
            </ToastProvider>
          </RadixToast.Provider>
        </ThemeProvider>
      </LiveAnnouncer>
    </RecoilRoot>
  </QueryClientProvider>
</ApiErrorBoundaryProvider>
```

**Provider Purposes:**

1. **ApiErrorBoundaryProvider:**
   - Catches API errors globally
   - Handles 401 redirects
   - Error UI fallback

2. **QueryClientProvider:**
   - TanStack Query client
   - Server state caching
   - Automatic refetching

3. **RecoilRoot:**
   - Recoil state container (legacy)
   - Being migrated to Jotai

4. **LiveAnnouncer:**
   - Accessibility announcements
   - Screen reader support

5. **ThemeProvider:**
   - Dark/light mode
   - Custom color themes
   - CSS variable management

6. **RadixToast.Provider:**
   - Radix UI toast system
   - Notification viewport

7. **ToastProvider:**
   - Custom toast logic
   - Toast queue management

8. **DndProvider:**
   - React DnD backend
   - HTML5 drag & drop

9. **ScreenshotProvider:**
   - Screenshot capture
   - Image conversion

## Component Organization

### Feature-Based Structure

```
components/
├── Agents/              # Agent marketplace & management
│   ├── AgentCard.tsx
│   ├── AgentGrid.tsx
│   ├── AgentDetail.tsx
│   └── tests/
├── Artifacts/          # Code artifacts & preview
│   ├── Artifact.tsx
│   ├── ArtifactPreview.tsx
│   └── CodeEditor.tsx
├── Chat/               # Main chat interface
│   ├── ChatView.tsx
│   ├── Input/
│   │   ├── ChatForm.tsx
│   │   ├── TextareaAutosize.tsx
│   │   └── ToolCallHandler.tsx
│   ├── Messages/
│   │   ├── Message.tsx
│   │   ├── MessageList.tsx
│   │   └── StreamingMessage.tsx
│   └── Header/
├── Messages/           # Message rendering
│   ├── Content/
│   ├── Feedback/
│   └── Artifacts/
├── Conversations/      # Conversation list
│   ├── ConvoList.tsx
│   ├── ConvoItem.tsx
│   └── SearchBar.tsx
├── SidePanel/         # Left sidebar
│   ├── NewChat.tsx
│   ├── ConvoList.tsx
│   └── Presets.tsx
├── Nav/               # Navigation
│   ├── NavLinks.tsx
│   ├── Settings/
│   └── UserMenu.tsx
├── Prompts/           # Prompt library
├── Files/             # File management
├── Auth/              # Authentication UI
│   ├── Login.tsx
│   ├── Registration.tsx
│   └── PasswordReset.tsx
└── ui/                # Reusable UI components
    ├── Button.tsx
    ├── Dialog.tsx
    ├── Input.tsx
    ├── Select.tsx
    └── ... 50+ components
```

### Component Patterns

**1. Smart vs Presentational:**
```typescript
// Smart component (container)
const ChatView = () => {
  const messages = useMessages();
  const sendMessage = useSendMessage();

  return <ChatContainer messages={messages} onSend={sendMessage} />;
};

// Presentational component
const ChatContainer = ({ messages, onSend }) => (
  <div>
    <MessageList messages={messages} />
    <ChatForm onSubmit={onSend} />
  </div>
);
```

**2. Compound Components:**
```typescript
<Dialog>
  <Dialog.Trigger>Open</Dialog.Trigger>
  <Dialog.Content>
    <Dialog.Title>Title</Dialog.Title>
    <Dialog.Description>Description</Dialog.Description>
  </Dialog.Content>
</Dialog>
```

**3. Render Props:**
```typescript
<Virtuoso
  data={messages}
  itemContent={(index, message) => (
    <Message key={message.id} {...message} />
  )}
/>
```

## Custom Hooks

**158+ Custom Hooks** organized by category:

### Data Fetching (30+ hooks)
- `useGetConversations()`
- `useGetMessages()`
- `useGetAgents()`
- `useGetModels()`

### Mutations (20+ hooks)
- `useSendMessage()`
- `useUpdateMessage()`
- `useCreateAgent()`
- `useUploadFile()`

### State Hooks (40+ hooks)
- `useConversation()`
- `useMessages()`
- `useAgents()`
- `useSettings()`

### UI Hooks (30+ hooks)
- `useModal()`
- `useToast()`
- `useToggle()`
- `useDebounce()`

### Auth Hooks (10+ hooks)
- `useAuth()`
- `useAuthRedirect()`
- `useLogout()`

### WebSocket/SSE Hooks
- `useSSE()` - Server-Sent Events
- `useMessageStream()` - AI response streaming

## Data Provider Layer

**Location:** `client/src/data-provider/`

**Purpose:** Abstraction layer between UI and API

**Structure:**
```typescript
// Query hooks (TanStack Query)
export const useGetConversations = (params) => {
  return useQuery({
    queryKey: ['conversations', params],
    queryFn: () => dataService.getConversations(params),
    staleTime: 5 * 60 * 1000 // 5 minutes
  });
};

// Mutation hooks
export const useSendMessage = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data) => dataService.sendMessage(data),
    onSuccess: () => {
      queryClient.invalidateQueries(['messages']);
    }
  });
};

// Data service (API calls)
const dataService = {
  getConversations: (params) => {
    return axios.get('/api/convos', { params });
  },
  sendMessage: (data) => {
    return axios.post(`/api/messages/${data.conversationId}`, data);
  }
};
```

## Styling Approach

### Tailwind CSS

**Configuration:** `client/tailwind.config.js`

**Custom Theme:**
```javascript
theme: {
  extend: {
    colors: {
      'text-primary': 'rgb(var(--text-primary) / <alpha-value>)',
      'surface-primary': 'rgb(var(--surface-primary) / <alpha-value>)',
      // ... 50+ custom colors
    },
    spacing: {
      '18': '4.5rem',
      '88': '22rem'
    }
  }
}
```

**Usage:**
```typescript
<div className="flex items-center gap-2 rounded-lg bg-surface-primary p-4 text-text-primary">
  <Button variant="primary" size="md">
    Send
  </Button>
</div>
```

### CSS Variables (Theme System)

**Dark/Light Mode:**
```css
:root {
  --text-primary: 11 11 11;      /* Light mode */
  --surface-primary: 255 255 255;
}

.dark {
  --text-primary: 236 236 241;   /* Dark mode */
  --surface-primary: 33 33 33;
}
```

### Class Variance Authority (CVA)

**Button Variants:**
```typescript
const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium',
  {
    variants: {
      variant: {
        primary: 'bg-green-500 text-white hover:bg-green-600',
        secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
        ghost: 'hover:bg-gray-100'
      },
      size: {
        sm: 'h-8 px-3',
        md: 'h-10 px-4',
        lg: 'h-12 px-6'
      }
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md'
    }
  }
);
```

## Build Configuration

### Vite Configuration

**File:** `client/vite.config.ts`

**Key Features:**
- React plugin with Fast Refresh
- PWA support (VitePWA plugin)
- Path aliases (`~/`, `@/`)
- Code splitting strategy
- Environment variable injection

**Chunk Strategy:**
```javascript
manualChunks: (id) => {
  if (id.includes('node_modules')) {
    if (id.includes('react') || id.includes('react-dom')) {
      return 'react-vendor';
    }
    if (id.includes('@radix-ui')) {
      return 'radix-vendor';
    }
    if (id.includes('@tanstack')) {
      return 'tanstack-vendor';
    }
    return 'vendor';
  }
}
```

### Environment Variables

**File:** `client/.env.development`

**Common Variables:**
```
VITE_API_URL=http://localhost:3080
VITE_WS_URL=ws://localhost:3080
VITE_SHOW_DEVTOOLS=true
```

**Usage:**
```typescript
const apiUrl = import.meta.env.VITE_API_URL;
```

## Performance Optimizations

### 1. Code Splitting

**Route-based:**
```typescript
const Chat = lazy(() => import('./routes/Chat'));
const Agents = lazy(() => import('./routes/Agents'));

<Route path="/chat" element={<Suspense><Chat /></Suspense>} />
```

**Component-based:**
```typescript
const HeavyComponent = lazy(() => import('./HeavyComponent'));
```

### 2. Virtualization

**Long Lists:**
```typescript
import { Virtuoso } from '@tanstack/react-virtual';

<Virtuoso
  data={messages}
  itemContent={(index, message) => <Message {...message} />}
  overscan={10}
/>
```

### 3. Memoization

**React.memo:**
```typescript
export const Message = React.memo(({ message }) => {
  return <div>{message.text}</div>;
});
```

**useMemo/useCallback:**
```typescript
const processedData = useMemo(() => {
  return expensiveOperation(data);
}, [data]);

const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

### 4. Image Optimization

**Lazy Loading:**
```typescript
<img src={url} loading="lazy" />
```

**HEIC Conversion:**
```typescript
import heicConvert from 'heic-convert';

const convertHEIC = async (file) => {
  const arrayBuffer = await file.arrayBuffer();
  const jpegBuffer = await heicConvert({
    buffer: arrayBuffer,
    format: 'JPEG'
  });
  return new Blob([jpegBuffer], { type: 'image/jpeg' });
};
```

## Progressive Web App (PWA)

**VitePWA Plugin:**
- Service worker registration
- Offline support
- App manifest
- Install prompts

**Manifest:**
```json
{
  "name": "LibreChat",
  "short_name": "LibreChat",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192" },
    { "src": "/icon-512.png", "sizes": "512x512" }
  ],
  "theme_color": "#10a37f",
  "display": "standalone"
}
```

## Accessibility (a11y)

### Features

1. **Keyboard Navigation:**
   - All interactive elements focusable
   - Skip links
   - Focus trapping in modals

2. **Screen Reader Support:**
   - ARIA labels
   - Live regions
   - Semantic HTML

3. **LiveAnnouncer:**
```typescript
const { announcePolite, announceAssertive } = useLiveAnnouncer();

announcePolite('Message sent');
announceAssertive('Error occurred');
```

4. **axe-core Testing:**
   - Automated a11y tests
   - E2E accessibility checks

## Testing

### Unit Tests (Jest)

**Location:** `client/test/`

**Test Pattern:**
```typescript
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders children', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });
});
```

### E2E Tests (Playwright)

**Location:** `e2e/specs/`

**Test Pattern:**
```typescript
test('should send a message', async ({ page }) => {
  await page.goto('/chat/new');
  await page.fill('[data-testid="chat-input"]', 'Hello');
  await page.click('[data-testid="send-button"]');
  await expect(page.locator('.message')).toContainText('Hello');
});
```

## Known Issues & TODOs

1. **Recoil → Jotai Migration:**
   - Ongoing migration from Recoil to Jotai
   - Some components still use Recoil

2. **TypeScript Coverage:**
   - Mixed JS/TS codebase
   - Gradual TypeScript adoption

3. **Bundle Size:**
   - Large vendor bundles
   - Consider lazy loading more libraries

## Related Documentation

- [State Management](state-management.md) - Detailed state management guide
- [Component Architecture](component-architecture.md) - Component patterns
- [Routing](routing.md) - React Router configuration
- [API Integration](api-integration.md) - Data provider layer

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
