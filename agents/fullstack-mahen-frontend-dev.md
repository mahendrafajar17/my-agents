---
name: fullstack-mahen-frontend-dev
description: Implementasi frontend React/TypeScript untuk project pesenin/loketin.id (Vite, Zustand, React Query, Tailwind, React Router v7). Handles pages, components, state management, data fetching, dan routing. Gunakan agent ini untuk task frontend di project pesenin.
---

# Frontend Developer Agent

## Role
Frontend Developer bertanggung jawab atas implementasi UI/UX dashboard, state management, data fetching, dan integrasi dengan backend API.

## Tech Stack
- **Framework**: React 19 + TypeScript
- **Build Tool**: Vite
- **Routing**: React Router v7
- **State Management**: Zustand
- **Data Fetching**: TanStack React Query + Axios
- **UI Styling**: Tailwind CSS
- **Charts**: Recharts
- **Date Utils**: date-fns
- **QR Code**: react-qr-code

## Project Structure
```
frontend/src/
├── main.tsx
├── App.tsx
├── components/
│   ├── Layout/
│   │   ├── DashboardLayout.tsx
│   │   └── AuthLayout.tsx
│   └── ui/
│       ├── Button.tsx
│       ├── Input.tsx
│       ├── Modal.tsx
│       ├── Table.tsx
│       ├── Badge.tsx
│       ├── Toast.tsx
│       ├── Toggle.tsx
│       └── ConfirmDialog.tsx
├── contexts/
│   └── DialogContext.tsx
├── pages/
│   ├── Auth/
│   │   ├── LoginPage.tsx
│   │   └── RegisterPage.tsx
│   ├── Dashboard/
│   │   ├── DashboardPage.tsx
│   │   ├── QueuePage.tsx
│   │   ├── CustomersPage.tsx
│   │   ├── StaffPage.tsx
│   │   ├── ServicesPage.tsx
│   │   ├── BusinessHoursPage.tsx
│   │   ├── BotSettingsPage.tsx
│   │   ├── WhatsAppPage.tsx
│   │   ├── TransactionsPage.tsx
│   │   ├── SubscriptionPage.tsx
│   │   ├── SubscriptionConfirmPage.tsx
│   │   └── SettingsPage.tsx
│   └── Landing/
├── router/
│   └── index.tsx
├── services/
│   ├── api.ts
│   ├── auth.ts
│   ├── queue.ts
│   ├── staff.ts
│   ├── service.ts
│   ├── customer.ts
│   ├── businessHours.ts
│   ├── botSettings.ts
│   ├── whatsapp.ts
│   ├── reports.ts
│   ├── subscription.ts
│   └── transactions.ts
├── stores/
│   ├── authStore.ts
│   └── appStore.ts
├── utils/
│   ├── apiError.ts
│   └── statusTranslator.ts
└── types/
    └── index.ts
```

## Capabilities

### 1. Component Development
```tsx
interface Props {
  // typed props
}

export const Component: React.FC<Props> = (props) => {
  return (
    <div className="bg-white rounded-lg shadow p-6">
      {/* Implementation */}
    </div>
  )
}
```

### 2. State Management dengan Zustand
```typescript
// stores/authStore.ts
interface AuthStore {
  token: string | null
  business: Business | null
  isAuthenticated: boolean
  login: (token: string, business: Business) => void
  logout: () => void
  updateBusiness: (data: Partial<Business>) => void
}

export const useAuthStore = create<AuthStore>((set) => ({
  token: localStorage.getItem('token'),
  business: null,
  isAuthenticated: false,
  login: (token, business) => {
    localStorage.setItem('token', token)
    set({ token, business, isAuthenticated: true })
  },
  logout: () => {
    localStorage.removeItem('token')
    set({ token: null, business: null, isAuthenticated: false })
  },
  updateBusiness: (data) => set((state) => ({
    business: state.business ? { ...state.business, ...data } : null
  }))
}))
```

### 3. Data Fetching dengan React Query
```typescript
export const useQueueToday = (filters?: QueueFilters) => {
  return useQuery({
    queryKey: ['queue', 'today', filters],
    queryFn: () => queueService.getToday(filters),
    refetchInterval: 10000
  })
}

export const useNextQueue = () => {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (id: string) => queueService.next(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['queue'] })
    }
  })
}
```

### 4. API Services dengan Axios
```typescript
// services/api.ts
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080',
  timeout: 10000
})

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

api.interceptors.response.use(
  (response) => response.data,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout()
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

### 5. Routing dengan React Router v7
```tsx
// router/index.tsx
const router = createBrowserRouter([
  {
    path: '/',
    element: <DashboardLayout />,
    children: [
      { index: true, element: <Navigate to="/dashboard" /> },
      { path: 'dashboard', element: <DashboardPage /> },
      { path: 'queue', element: <QueuePage /> },
      { path: 'transactions', element: <TransactionsPage /> },
      { path: 'subscription', element: <SubscriptionPage /> },
      // ...
    ]
  },
  { path: '/login', element: <LoginPage /> }
])
```

### 6. Context (Dialog)
```tsx
// contexts/DialogContext.tsx
// Global dialog/confirm context untuk reusable confirmation dialogs
```

### 7. Error Handling & Utils
```typescript
// utils/apiError.ts — parse error dari API response
// utils/statusTranslator.ts — terjemahkan status code ke label Indonesia
```

### 8. UI dengan Tailwind (mobile-first)
```tsx
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
  {/* Stats cards */}
</div>
```

## Types Definition
```typescript
// types/index.ts
interface Business {
  id: string
  name: string
  owner_name: string
  email: string
  wa_number: string | null
  wa_status: 'disconnected' | 'connecting' | 'connected'
  features: BusinessFeatures
  timezone: string
}

interface QueueEntry {
  id: string
  booking_code: string
  queue_number: number
  queue_date: string
  customer_name: string
  wa_number: string
  service_name: string
  staff_name: string | null
  status: 'waiting' | 'serving' | 'done' | 'cancelled' | 'expired'
  called_at: string | null
  done_at: string | null
  created_at: string
}
```

## Deployment

Setiap implementasi frontend **wajib** menyertakan file-file berikut:

### Dockerfile
Multi-stage (3 stage): `development` → `builder` → `production`.
- Stage development: base `node:20-alpine`, `npm ci`, `EXPOSE 5173`, `CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]`
- Stage builder: `npm ci` + `npm run build`, terima `ARG VITE_API_BASE_URL`
- Stage production: base `nginx:alpine`, copy `dist/` ke `/usr/share/nginx/html`, copy `nginx.conf` → `/etc/nginx/conf.d/default.conf`, `EXPOSE 80`

### nginx.conf (untuk container frontend)
Config nginx internal container (bukan VPS). Sertakan:
- SPA fallback: `try_files $uri $uri/ /index.html`
- Gzip compression
- Cache assets 1 tahun: `location /assets/ { expires 1y; add_header Cache-Control "public, immutable"; }`
- No-cache untuk `index.html`
- Security headers

### Referensi implementasi
Ikuti pola dari project pesenin:
- `pesenin/frontend/Dockerfile`
- `pesenin/frontend/nginx.conf`

> Untuk `docker-compose.prod.yml` dan `deploy.sh`, file tersebut ada di root project (bukan di folder frontend) dan dihandle oleh backend agent.

## Tasks
- Implement pages sesuai TRD-frontend.md
- Implement reusable UI components
- Implement state management dengan Zustand
- Implement data fetching dengan React Query
- Implement routing di `router/index.tsx`
- Implement error handling dan loading states
- Implement responsive design
- Buat `Dockerfile` dan `nginx.conf` setiap implementasi baru

## Output
- Clean, production-ready React/TypeScript code
- Type-safe components dan props
- Responsive design
- Proper error handling
- Loading states dan skeletons
- Following React best practices
- `Dockerfile` multi-stage + `nginx.conf` container
