# State Management

> **State management architecture for LibreChat v0.8.1-rc1**

## Overview

LibreChat implements a **hybrid state management architecture** combining three complementary solutions:

1. **Jotai** - Primary client state (atomic, bottom-up)
2. **Recoil** - Legacy client state (being migrated to Jotai)
3. **TanStack Query** - Server state (caching, sync, mutations)

## Source Code References

**State Files:**
- Jotai/Recoil atoms: `/home/user/LibreChat/client/src/store/` (20 files)
- Store index: `client/src/store/index.ts` (lines 1-34)
- TanStack Query hooks: `client/src/data-provider/queries.ts`

## Architecture Pattern

```
┌─────────────────────────────────────────────┐
│         React Component Tree                │
└─────────────────────────────────────────────┘
           │
     ┌─────┴─────┐
     │           │
┌────▼────┐ ┌───▼────────┐
│  Jotai  │ │  Recoil    │ ← Client State (UI, temp data)
│ Atoms   │ │  Atoms     │
└─────────┘ └────────────┘
           │
      ┌────▼──────┐
      │  TanStack │ ← Server State (API data)
      │   Query   │
      └───────────┘
           │
      ┌────▼──────┐
      │    API    │
      └───────────┘
```

## Jotai (Primary State)

**Why Jotai:**
- Minimal boilerplate
- TypeScript-first
- Atomic updates (no unnecessary re-renders)
- Better DevTools support
- Easier testing

### Atom Modules

**Location:** `client/src/store/`

| Module | File | Purpose |
|--------|------|---------|
| Agents | `agents.ts` | Ephemeral agent state per conversation |
| Artifacts | `artifacts.ts` | Code artifact state |
| Endpoints | `endpoints.ts` | AI endpoint configurations |
| Settings | `settings.ts` | User preferences |
| MCP | `mcp.ts` | Model Context Protocol state |
| Prompts | `prompts.ts` | Prompt library state |
| User | `user.ts` | User profile state |
| Toast | `toast.ts` | Toast notification queue |
| Search | `search.ts` | Search UI state |
| Text | `text.ts` | Text input state |
| FontSize | `fontSize.ts` | Accessibility font size |

### Atom Pattern (Jotai)

**File:** `client/src/store/agents.ts` (lines 1-100)

**Atom Definition:**
```typescript
import { atomFamily } from 'recoil'; // Note: Using Recoil's atomFamily pattern
import type { TEphemeralAgent } from 'librechat-data-provider';

// Atom family: one atom per conversation ID
export const ephemeralAgentByConvoId = atomFamily<TEphemeralAgent | null, string>({
  key: 'ephemeralAgentByConvoId',
  default: null,
  effects: [
    ({ onSet, node }) => {
      onSet(async (newValue) => {
        const conversationId = node.key.split('__')[1]?.replaceAll('"', '');
        logger.log('agents', 'Setting ephemeral agent:', { conversationId, newValue });
      });
    },
  ],
});
```

**Usage Hook:**
```typescript
export function useUpdateEphemeralAgent() {
  const updateEphemeralAgent = useRecoilCallback(
    ({ set }) =>
      (convoId: string, agent: TEphemeralAgent | null) => {
        set(ephemeralAgentByConvoId(convoId), agent);
      },
    [],
  );

  return updateEphemeralAgent;
}
```

**Component Usage:**
```typescript
import { useUpdateEphemeralAgent } from '~/store/agents';

const AgentSelector = ({ conversationId }) => {
  const updateAgent = useUpdateEphemeralAgent();

  const handleSelect = (agent) => {
    updateAgent(conversationId, agent);
  };

  return <AgentList onSelect={handleSelect} />;
};
```

### Jotai Best Practices

**1. Atom Families for Dynamic Keys:**
```typescript
// ✅ Good - Separate atom per conversation
const messagesByConvoId = atomFamily({
  key: 'messages',
  default: (convoId) => []
});

// ❌ Avoid - Single atom for all conversations
const allMessages = atom({
  key: 'allMessages',
  default: {}
});
```

**2. Derived State with Selectors:**
```typescript
const messageCountByConvoId = selectorFamily({
  key: 'messageCount',
  get: (convoId) => ({ get }) => {
    const messages = get(messagesByConvoId(convoId));
    return messages.length;
  }
});
```

**3. Effects for Side Effects:**
```typescript
const settingsAtom = atom({
  key: 'settings',
  default: {},
  effects: [
    ({ onSet }) => {
      onSet((newValue) => {
        localStorage.setItem('settings', JSON.stringify(newValue));
      });
    }
  ]
});
```

## Recoil (Legacy)

**Status:** Being migrated to Jotai

**Still Used In:**
- Some older components
- Families module (`client/src/store/families.ts`)
- Conversation state (partially)

**Migration Strategy:**
1. New features use Jotai
2. Gradual Recoil → Jotai migration
3. Recoil removed once migration complete

## TanStack Query (Server State)

**Purpose:** API data fetching, caching, synchronization

### Configuration

**File:** `client/src/App.jsx` (lines 19-27)

```javascript
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) => {
      if (error?.response?.status === 401) {
        setError(error); // Redirect to login
      }
    },
  }),
});
```

### Query Hooks

**File:** `client/src/data-provider/queries.ts`

**Pattern:**
```typescript
// 1. Query hook (fetching)
export const useGetConversations = (params) => {
  return useQuery({
    queryKey: ['conversations', params],
    queryFn: () => dataService.getConversations(params),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    refetchOnWindowFocus: true
  });
};

// 2. Infinite query (pagination)
export const useGetInfiniteMessages = (conversationId) => {
  return useInfiniteQuery({
    queryKey: ['messages', conversationId],
    queryFn: ({ pageParam }) =>
      dataService.getMessages(conversationId, { cursor: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    staleTime: 30 * 1000 // 30 seconds
  });
};

// 3. Mutation hook (updating)
export const useSendMessage = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data) => dataService.sendMessage(data),
    onMutate: async (newMessage) => {
      // Optimistic update
      await queryClient.cancelQueries(['messages']);
      const previousMessages = queryClient.getQueryData(['messages']);
      queryClient.setQueryData(['messages'], (old) => [...old, newMessage]);
      return { previousMessages };
    },
    onError: (err, newMessage, context) => {
      // Rollback on error
      queryClient.setQueryData(['messages'], context.previousMessages);
    },
    onSuccess: () => {
      // Refetch after success
      queryClient.invalidateQueries(['messages']);
    }
  });
};
```

### Component Usage

```typescript
const ConversationList = () => {
  const { data, isLoading, error, refetch } = useGetConversations({
    limit: 25,
    isArchived: false
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;

  return (
    <div>
      {data.conversations.map(convo => (
        <ConvoItem key={convo.conversationId} {...convo} />
      ))}
      <button onClick={() => refetch()}>Refresh</button>
    </div>
  );
};
```

### Query Key Strategy

**Hierarchical Keys:**
```typescript
// Base key
['conversations']

// With params
['conversations', { limit: 25, isArchived: false }]

// Nested resources
['conversations', conversationId, 'messages']

// With filters
['conversations', conversationId, 'messages', { cursor: 'abc123' }]
```

**Invalidation:**
```typescript
// Invalidate all conversations
queryClient.invalidateQueries(['conversations']);

// Invalidate specific conversation
queryClient.invalidateQueries(['conversations', conversationId]);

// Invalidate messages for conversation
queryClient.invalidateQueries(['conversations', conversationId, 'messages']);
```

### Caching Strategy

**Configuration:**
```typescript
{
  staleTime: 5 * 60 * 1000,    // Data fresh for 5 minutes
  cacheTime: 10 * 60 * 1000,   // Cache persists for 10 minutes
  refetchOnWindowFocus: true,  // Refetch when window regains focus
  refetchOnMount: true,        // Refetch on component mount
  retry: 3,                    // Retry failed requests 3 times
}
```

**Garbage Collection:**
- Unused queries removed after `cacheTime`
- Background refetch for stale data
- Manual cache clearing: `queryClient.clear()`

## State Persistence

### LocalStorage

**Settings Persistence:**
```typescript
const settingsAtom = atom({
  key: 'settings',
  default: loadSettings(),
  effects: [
    ({ onSet }) => {
      onSet((newValue) => {
        localStorage.setItem('librechat_settings', JSON.stringify(newValue));
      });
    }
  ]
});

function loadSettings() {
  const stored = localStorage.getItem('librechat_settings');
  return stored ? JSON.parse(stored) : defaultSettings;
}
```

**Persisted State:**
- Theme preferences
- Font size
- Language selection
- UI layout preferences

### SessionStorage

**Temporary State:**
- Draft messages
- Form data
- Tab state

## DevTools

### React Query DevTools

**File:** `client/src/App.jsx` (line 54)

```javascript
<ReactQueryDevtools initialIsOpen={false} position="top-right" />
```

**Features:**
- Query inspection
- Cache visualization
- Refetch triggers
- Timeline view

### Recoil DevTools

**Browser Extension:**
- Recoil DevTools Chrome extension
- Atom/selector inspection
- State history
- Time-travel debugging

## State Migration (Recoil → Jotai)

### Migration Checklist

**For Each Atom:**

1. **Convert Atom Definition:**
```typescript
// Before (Recoil)
const myAtom = atom({
  key: 'myKey',
  default: defaultValue
});

// After (Jotai)
import { atom } from 'jotai';

const myAtom = atom(defaultValue);
```

2. **Convert Atom Family:**
```typescript
// Before (Recoil)
const myFamily = atomFamily({
  key: 'myFamily',
  default: (id) => defaultValue
});

// After (Jotai)
import { atomFamily } from 'jotai/utils';

const myFamily = atomFamily((id) => defaultValue);
```

3. **Convert Selectors:**
```typescript
// Before (Recoil)
const mySelector = selector({
  key: 'mySelector',
  get: ({ get }) => {
    const value = get(myAtom);
    return value * 2;
  }
});

// After (Jotai)
const mySelector = atom((get) => {
  const value = get(myAtom);
  return value * 2;
});
```

4. **Update Component Usage:**
```typescript
// Before (Recoil)
import { useRecoilValue, useSetRecoilState } from 'recoil';

const value = useRecoilValue(myAtom);
const setValue = useSetRecoilState(myAtom);

// After (Jotai)
import { useAtom, useAtomValue, useSetAtom } from 'jotai';

const value = useAtomValue(myAtom);
const setValue = useSetAtom(myAtom);
```

## Performance Optimization

### 1. Selective Subscriptions

**Jotai (Automatic):**
```typescript
// Only re-renders when userName changes
const userName = useAtomValue(userNameAtom);
```

**Recoil:**
```typescript
// Only re-renders when name property changes
const userName = useRecoilValue(userAtom).name;
```

### 2. Atom Splitting

**Split Large Atoms:**
```typescript
// ❌ Avoid - Large atom
const userAtom = atom({
  name: '',
  email: '',
  settings: {},
  preferences: {}
});

// ✅ Good - Split atoms
const userNameAtom = atom('');
const userEmailAtom = atom('');
const userSettingsAtom = atom({});
const userPreferencesAtom = atom({});
```

### 3. Query Deduplication

**TanStack Query automatically deduplicates:**
```typescript
// Both components share same query
const Component1 = () => {
  const { data } = useGetUser(userId); // Triggers fetch
};

const Component2 = () => {
  const { data } = useGetUser(userId); // Reuses existing query
};
```

### 4. Background Refetching

**Keep data fresh without blocking UI:**
```typescript
useQuery({
  queryKey: ['messages'],
  queryFn: getMessages,
  staleTime: 30 * 1000,
  refetchInterval: 60 * 1000 // Background refetch every 60s
});
```

## Testing

### Testing Jotai Atoms

```typescript
import { renderHook, act } from '@testing-library/react';
import { useAtom } from 'jotai';
import { Provider } from 'jotai';

test('should update atom value', () => {
  const { result } = renderHook(() => useAtom(myAtom), {
    wrapper: ({ children }) => <Provider>{children}</Provider>
  });

  act(() => {
    result.current[1]('new value');
  });

  expect(result.current[0]).toBe('new value');
});
```

### Testing TanStack Query

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { renderHook, waitFor } from '@testing-library/react';

test('should fetch data', async () => {
  const queryClient = new QueryClient();
  const wrapper = ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );

  const { result } = renderHook(() => useGetConversations(), { wrapper });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toBeDefined();
});
```

## Best Practices

### Do's

✅ **Use TanStack Query for server state:**
```typescript
const { data } = useGetMessages(conversationId);
```

✅ **Use Jotai for client state:**
```typescript
const [theme, setTheme] = useAtom(themeAtom);
```

✅ **Keep atoms focused and small:**
```typescript
const userNameAtom = atom('');
const userEmailAtom = atom('');
```

✅ **Use atom families for dynamic keys:**
```typescript
const messagesByConvoId = atomFamily((id) => []);
```

### Don'ts

❌ **Don't use atoms for server data:**
```typescript
// Bad - Use TanStack Query instead
const messagesAtom = atom([]);
```

❌ **Don't create huge atoms:**
```typescript
// Bad - Split into smaller atoms
const appStateAtom = atom({
  user: {},
  messages: [],
  conversations: [],
  settings: {}
});
```

❌ **Don't bypass query cache:**
```typescript
// Bad - Always use TanStack Query
const fetchMessages = async () => {
  const response = await fetch('/api/messages');
  return response.json();
};
```

## Related Documentation

- [Frontend Architecture](README.md) - Overall frontend overview
- [Component Architecture](component-architecture.md) - Component patterns
- [API Integration](api-integration.md) - Data provider details

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0
