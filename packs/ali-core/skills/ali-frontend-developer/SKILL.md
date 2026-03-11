---
name: ali-frontend-developer
description: |
  Frontend Developer for Tax Practice AI. Use when:

  PLANNING: Designing component architecture, planning state management
  strategy, choosing UI patterns, architecting form flows

  IMPLEMENTATION: Creating pages/components in React/TypeScript, styling
  with Tailwind/shadcn/ui, implementing React Query hooks, Zustand stores,
  connecting to API endpoints, handling forms

  GUIDANCE: Asking about responsive design patterns, accessibility standards,
  loading/error state best practices, component composition patterns

  REVIEW: Checking component structure, validating data-testid coverage,
  reviewing accessibility compliance, auditing state management patterns

  Do NOT use for backend API development, database queries, or service layer logic
  (use ali-backend-developer instead), or test implementation (use ali-test-developer instead)
---

# Frontend Developer - Tax Practice AI

## STOP - MANDATORY DATA-TESTID REQUIREMENT

**BEFORE writing ANY component code, you MUST add data-testid attributes.**

This is a BLOCKING requirement. Code without data-testid will:
- Break automated tests
- Fail PR review
- Require rework

**VERIFICATION CHECKLIST (complete before saving):**
- [ ] Every `<button>` has `data-testid="btn-{action}-{context}"`
- [ ] Every `<input>` has `data-testid="input-{field}"`
- [ ] Every clickable row has `data-testid="row-{type}-{id}"`
- [ ] Every panel/section has `data-testid="panel-{name}"`
- [ ] Icon-only buttons have BOTH `aria-label` AND `data-testid`

**If you forget data-testid, you MUST go back and add it before moving on.**

---

## When I Activate

This skill activates when working on:

- `frontend/apps/**/*.tsx` - React components
- `frontend/apps/**/*.ts` - TypeScript utilities
- `frontend/packages/ui/**/*` - Shared UI library
- `frontend/**/hooks/*.ts` - Custom React hooks
- `frontend/**/stores/*.ts` - Zustand stores
- Component styling with Tailwind CSS
- Any UI/UX implementation work

---

## FIRST ACTION (MANDATORY)

Before ANY other action, use the Read tool to load:
- `docs/TEST_PROTOCOL.md` (Section 6: data-testid requirements)
- `frontend/apps/staff-app/src/test-ids.ts` (centralized test ID constants)

Do not proceed until these files are in your context. This is not optional.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Framework | React 18 | Component framework |
| Build | Vite | Fast dev, optimized builds |
| Language | TypeScript | Type safety |
| Styling | Tailwind CSS | Utility-first CSS |
| Components | shadcn/ui | Accessible, customizable primitives |
| Server State | React Query | API data, caching, sync |
| UI State | Zustand | Sidebar, modals, filters |
| Forms | React Hook Form + Zod | Validation, performance |
| Icons | Lucide React | Consistent iconography |

---

## Project Structure

```
frontend/
├── packages/ui/                    # Shared component library
│   └── src/
│       ├── components/ui/          # shadcn/ui primitives
│       │   ├── button.tsx
│       │   ├── card.tsx
│       │   ├── dialog.tsx
│       │   └── ...
│       └── lib/
│           ├── api.ts              # API client
│           ├── utils.ts            # cn() and helpers
│           └── auth.ts             # Auth utilities
│
├── apps/
│   ├── client-portal/              # Client-facing app
│   │   └── src/
│   │       ├── pages/
│   │       ├── components/
│   │       └── hooks/
│   │
│   └── staff-app/                  # Staff application
│       └── src/
│           ├── pages/
│           ├── components/
│           ├── hooks/
│           ├── stores/
│           └── test-ids.ts         # Centralized test IDs
│
├── features/                       # BDD feature files
├── steps/                          # Step definitions
└── tests/                          # Test utilities
```

---

## Golden Rules

### 1. Every Interactive Element Needs data-testid

This is non-negotiable. Tests break without stable selectors.

```tsx
// CORRECT - testable
<button
  data-testid="btn-save-client"
  onClick={handleSave}
>
  Save
</button>

// WRONG - untestable
<button onClick={handleSave}>
  Save
</button>
```

**Use centralized TEST_IDS:**

```tsx
// staff-app/src/test-ids.ts defines all IDs
import { TEST_IDS } from '../test-ids';

<button data-testid={TEST_IDS.CLIENT_DETAIL.SAVE_CLIENT}>
  Save
</button>
```

### 2. Use shadcn/ui Components

Never create custom primitives. Use and extend shadcn/ui.

```tsx
// CORRECT - use shadcn/ui
import { Button, Card, Dialog, Input } from '@tax-practice/ui';

<Button variant="destructive" size="sm">
  Delete
</Button>

// WRONG - custom primitives
<button className="bg-red-500 text-white px-3 py-1 rounded">
  Delete
</button>
```

### 3. Server State via React Query

All API data flows through React Query hooks.

```tsx
// hooks/useClients.ts
export function useClients() {
  return useQuery({
    queryKey: ['clients'],
    queryFn: () => api.getClients(),
    staleTime: 1000 * 60 * 5, // 5 minutes
  });
}

export function useCreateClient() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: api.createClient,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['clients'] });
    },
  });
}
```

### 4. UI State via Zustand

Sidebar, modals, filters, and UI preferences use Zustand.

```tsx
// stores/uiStore.ts
export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
    }),
    { name: 'ui-storage' }
  )
);
```

### 5. Tailwind for All Styling

No CSS files. Use Tailwind utilities and cn() for conditionals.

```tsx
import { cn } from '@tax-practice/ui';

<div className={cn(
  "flex items-center gap-2 p-4 rounded-lg",
  isActive && "bg-primary/10 border-primary",
  isDisabled && "opacity-50 cursor-not-allowed"
)}>
```

---

## Component Patterns

### Button Variants

```tsx
// Use variant props, not custom classes
<Button variant="default">Primary Action</Button>
<Button variant="secondary">Secondary</Button>
<Button variant="destructive">Delete</Button>
<Button variant="outline">Cancel</Button>
<Button variant="ghost">Subtle</Button>
<Button variant="link">Link Style</Button>

<Button size="sm">Small</Button>
<Button size="default">Default</Button>
<Button size="lg">Large</Button>
<Button size="icon"><Icon /></Button>
```

### Loading States

Always show loading feedback:

```tsx
function ClientList() {
  const { data, isLoading, error } = useClients();

  if (isLoading) {
    return (
      <div className="flex items-center justify-center p-8">
        <Loader2 className="h-6 w-6 animate-spin text-muted-foreground" />
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center p-8 text-destructive">
        Failed to load clients. Please try again.
      </div>
    );
  }

  if (!data?.length) {
    return <EmptyState />;
  }

  return <Table>...</Table>;
}
```

### Empty States

Provide helpful empty states with clear actions:

```tsx
function EmptyState() {
  return (
    <div
      data-testid="empty-clients"
      className="text-center p-8 border-2 border-dashed rounded-lg"
    >
      <Users className="h-12 w-12 mx-auto text-muted-foreground mb-4" />
      <h3 className="text-lg font-medium">No clients yet</h3>
      <p className="text-muted-foreground mt-1">
        Get started by adding your first client.
      </p>
      <Button
        data-testid="btn-add-first-client"
        className="mt-4"
        onClick={() => navigate('/clients/new')}
      >
        Add Client
      </Button>
    </div>
  );
}
```

### Error Handling

Show errors clearly with recovery actions:

```tsx
function ErrorBoundaryFallback({ error, resetErrorBoundary }) {
  return (
    <div className="p-4 bg-destructive/10 border border-destructive rounded-lg">
      <h3 className="font-medium text-destructive">Something went wrong</h3>
      <p className="text-sm text-muted-foreground mt-1">
        {error.message}
      </p>
      <Button
        variant="outline"
        size="sm"
        onClick={resetErrorBoundary}
        className="mt-2"
      >
        Try Again
      </Button>
    </div>
  );
}
```

### Form Patterns

Use React Hook Form with Zod validation:

```tsx
const formSchema = z.object({
  email: z.string().email('Invalid email address'),
  firstName: z.string().min(1, 'First name is required'),
  lastName: z.string().min(1, 'Last name is required'),
});

function ClientForm() {
  const form = useForm<z.infer<typeof formSchema>>({
    resolver: zodResolver(formSchema),
    defaultValues: { email: '', firstName: '', lastName: '' },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  data-testid="input-email"
                  placeholder="client@example.com"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        {/* More fields... */}
        <Button data-testid="btn-submit" type="submit">
          Save Client
        </Button>
      </form>
    </Form>
  );
}
```

---

## UX Requirements

### Accessibility

- All interactive elements must be keyboard accessible
- Use semantic HTML (button for actions, a for links)
- Provide aria-labels for icon-only buttons
- Ensure color contrast meets WCAG AA

```tsx
// CORRECT
<Button
  variant="ghost"
  size="icon"
  aria-label="Delete document"
  data-testid="btn-delete-doc"
>
  <Trash2 className="h-4 w-4" />
</Button>

// WRONG - no accessible label
<Button variant="ghost" size="icon">
  <Trash2 className="h-4 w-4" />
</Button>
```

### Loading Feedback

Every async action needs loading indication:

```tsx
<Button
  data-testid="btn-save"
  disabled={mutation.isPending}
  onClick={() => mutation.mutate(data)}
>
  {mutation.isPending ? (
    <>
      <Loader2 className="h-4 w-4 mr-2 animate-spin" />
      Saving...
    </>
  ) : (
    'Save'
  )}
</Button>
```

### Toast Notifications

Use toast for action feedback:

```tsx
import { useToast } from '@tax-practice/ui';

function SaveButton() {
  const { toast } = useToast();
  const mutation = useCreateClient();

  const handleSave = async () => {
    try {
      await mutation.mutateAsync(data);
      toast({
        title: 'Client saved',
        description: 'The client has been created successfully.',
      });
    } catch (error) {
      toast({
        variant: 'destructive',
        title: 'Error',
        description: 'Failed to save client. Please try again.',
      });
    }
  };
}
```

### Responsive Design

Mobile-first approach with Tailwind breakpoints:

```tsx
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {/* Cards stack on mobile, 2 cols on tablet, 3 on desktop */}
</div>

<Dialog>
  <DialogContent className="sm:max-w-md">
    {/* Full width on mobile, constrained on larger screens */}
  </DialogContent>
</Dialog>
```

---

## Test ID Conventions

From test-ids.ts:

| Element | Pattern | Examples |
|---------|---------|----------|
| Button | `btn-{action}` | `btn-save-client`, `btn-delete` |
| Input | `input-{field}` | `input-email`, `input-first-name` |
| Select | `select-{name}` | `select-tax-year`, `select-role` |
| Checkbox | `checkbox-{name}` | `checkbox-send-email` |
| Switch | `switch-{name}` | `switch-is-business` |
| Tab | `tab-{name}` | `tab-documents`, `tab-analysis` |
| Panel | `panel-{name}` | `panel-ai-analysis`, `panel-documents` |
| Page | `{name}-page` | `clients-page`, `dashboard-page` |
| Row | `{type}-row` | `client-row`, `document-row` |
| Empty | `empty-{type}` | `empty-clients`, `empty-documents` |

**Import from centralized file:**

```tsx
import { TEST_IDS } from '../test-ids';

<button data-testid={TEST_IDS.CLIENTS.NEW_CLIENT}>New Client</button>
<input data-testid={TEST_IDS.CREATE_CLIENT.INPUT_EMAIL} />
```

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| Custom button styles | Inconsistent, hard to maintain | Use Button variants |
| Inline styles | Not responsive, no theme | Use Tailwind classes |
| CSS files | Split concerns, no colocation | Use Tailwind |
| Missing loading states | Poor UX, no feedback | Always show loading |
| Missing error states | User stuck on failure | Show error + retry |
| Missing empty states | Confusing blank page | Show helpful message |
| Missing data-testid | Tests break | Add to all interactive elements |
| Text-based selectors | Fragile tests | Use data-testid |
| Hardcoded colors | No dark mode support | Use Tailwind theme colors |
| Direct fetch calls | No caching, no sync | Use React Query |

---

## File Templates

### New Page

```tsx
// pages/ExamplePage.tsx
import { useState } from 'react';
import { useExample } from '../hooks/useExample';
import { Button, Card, CardHeader, CardTitle, CardContent } from '@tax-practice/ui';
import { Loader2 } from 'lucide-react';
import { TEST_IDS } from '../test-ids';

export function ExamplePage() {
  const { data, isLoading, error } = useExample();

  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-[400px]">
        <Loader2 className="h-6 w-6 animate-spin text-muted-foreground" />
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center p-8 text-destructive">
        Failed to load data. Please try again.
      </div>
    );
  }

  return (
    <div data-testid="example-page" className="space-y-6">
      <h1 className="text-2xl font-bold">Example Page</h1>

      <Card>
        <CardHeader>
          <CardTitle>Example Content</CardTitle>
        </CardHeader>
        <CardContent>
          {/* Page content */}
        </CardContent>
      </Card>
    </div>
  );
}
```

### New Hook

```tsx
// hooks/useExample.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@tax-practice/ui';

export function useExample() {
  return useQuery({
    queryKey: ['example'],
    queryFn: () => api.getExample(),
  });
}

export function useCreateExample() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: api.createExample,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['example'] });
    },
  });
}
```

---

## Quick Reference

### Imports

```tsx
// UI Components
import {
  Button, Card, CardHeader, CardTitle, CardContent,
  Dialog, DialogContent, DialogHeader, DialogTitle,
  Input, Label, Select, Checkbox, Switch,
  Table, TableHeader, TableBody, TableRow, TableCell,
  Badge, Tabs, TabsList, TabsTrigger, TabsContent,
  cn,
} from '@tax-practice/ui';

// Icons
import { Loader2, Plus, Trash2, Edit, Check, X } from 'lucide-react';

// Test IDs
import { TEST_IDS } from '../test-ids';
```

### Common Tailwind Patterns

```tsx
// Centering
className="flex items-center justify-center"

// Spacing children
className="space-y-4"  // vertical
className="space-x-4"  // horizontal
className="gap-4"      // grid/flex gap

// Responsive grid
className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4"

// Conditional classes
className={cn("base-class", condition && "conditional-class")}

// Status colors (use semantic names)
className="text-destructive"      // errors
className="text-muted-foreground" // secondary text
className="bg-primary/10"         // subtle highlight
```

---

## Related Documentation

- `docs/TEST_PROTOCOL.md` - Testing requirements (Section 6: data-testid)
- `docs/architecture/frontend.md` - Full frontend architecture
- `frontend/apps/staff-app/src/test-ids.ts` - Centralized test IDs
- `frontend/packages/ui/` - Shared component library
