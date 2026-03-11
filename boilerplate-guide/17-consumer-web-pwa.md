# 17. Consumer Web Frontend (PWA)

> Mobile-first PWA guide based on Next.js 16 + Zustand + Serwist.
> Covers mobile shell layout, multi-step forms, bottom tab navigation,
> Service Worker caching, and FCM push notifications.
> See alongside [07-admin-nextjs](./07-admin-nextjs.md) for shared Next.js patterns.

---

## 1. Dependencies

### package.json

```json
{
  "name": "@scope/web",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "next dev --port 15300",
    "build": "next build",
    "start": "next start --port 15300",
    "lint": "eslint src --max-warnings 0",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf .next out *.tsbuildinfo"
  },
  "dependencies": {
    "@scope/shared": "workspace:*",
    "next": "^16.1.6",
    "react": "^19.2.0",
    "react-dom": "^19.2.0",
    "@tanstack/react-query": "^5.66.0",
    "zustand": "^5.0.0",
    "react-hook-form": "^7.71.0",
    "@hookform/resolvers": "^5.0.0",
    "zod": "^3.24.0",
    "framer-motion": "^12.34.0",
    "recharts": "^3.7.0",
    "lucide-react": "^0.577.0",
    "firebase": "^12.10.0"
  },
  "devDependencies": {
    "@scope/eslint-config": "workspace:*",
    "@scope/tsconfig": "workspace:*",
    "@types/node": "^24.0.0",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0",
    "eslint": "^9.18.0",
    "typescript": "^5.7.3",
    "tailwindcss": "^4.2.1",
    "@tailwindcss/postcss": "^4.2.1",
    "postcss": "^8.5.1",
    "@serwist/next": "^9.5.6",
    "serwist": "^9.5.6"
  }
}
```

---

## 2. Config Files

### next.config.ts

```typescript
import withSerwist from "@serwist/next";
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@scope/shared"],
  typedRoutes: true,
  output: "standalone",
};

export default withSerwist({
  swSrc: "src/sw.ts",
  swDest: "public/sw.js",
  disable: process.env.NODE_ENV === "development",
})(nextConfig);
```

### tsconfig.json

```json
{
  "extends": "@scope/tsconfig/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] },
    "lib": ["ES2022", "DOM", "DOM.Iterable", "WebWorker"]
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### postcss.config.mjs

Same as [07-admin-nextjs](./07-admin-nextjs.md):

```javascript
const config = {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};

export default config;
```

### eslint.config.mjs

Same as [07-admin-nextjs](./07-admin-nextjs.md):

```javascript
import nextjsConfig from "@scope/eslint-config/nextjs";

/** @type {import('eslint').Linter.Config[]} */
const config = [...nextjsConfig];

export default config;
```

---

## 3. Directory Structure

```
apps/web/src/
├── app/
│   ├── layout.tsx              # Root layout (mobile shell)
│   ├── globals.css             # Tailwind import
│   ├── page.tsx                # Splash → redirect
│   ├── (auth)/                 # Public routes (no bottom tabs)
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── onboarding/
│   │       └── page.tsx
│   ├── (main)/                 # Authenticated routes (bottom tabs)
│   │   ├── layout.tsx          # Layout with BottomTabBar
│   │   ├── home/
│   │   │   └── page.tsx
│   │   ├── notifications/
│   │   │   └── page.tsx
│   │   └── profile/
│   │       └── page.tsx
│   ├── flow/                   # Multi-step flow (step navigation)
│   │   ├── layout.tsx          # Progress bar + prev/next
│   │   ├── step-1/
│   │   │   └── page.tsx
│   │   └── step-2/
│   │       └── page.tsx
│   ├── terms/
│   │   └── page.tsx
│   └── privacy/
│       └── page.tsx
├── components/
│   ├── ui/                     # Shared UI primitives
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── chip.tsx
│   │   ├── toggle.tsx
│   │   ├── progress-bar.tsx
│   │   ├── bottom-sheet.tsx
│   │   ├── search-input.tsx
│   │   ├── skeleton.tsx
│   │   └── toast.tsx
│   └── layout/
│       ├── mobile-shell.tsx    # 430px wrapper
│       ├── bottom-tab-bar.tsx  # Tab navigation
│       ├── flow-header.tsx     # Step progress
│       └── page-header.tsx     # Back button header
├── hooks/
│   ├── use-auth.ts
│   ├── use-online-status.ts
│   └── use-push.ts
├── stores/
│   ├── flow-store.ts           # Zustand: multi-step form state
│   └── ui-store.ts             # Zustand: bottom sheets, toasts
├── lib/
│   ├── api-client.ts           # HTTP client (see §14)
│   ├── auth-api.ts
│   ├── auth-context.tsx
│   ├── query-provider.tsx
│   ├── query-keys.ts
│   ├── types.ts
│   ├── constants.ts
│   ├── validators.ts           # Zod schemas for form validation
│   └── fcm.ts                  # Firebase Cloud Messaging setup
├── sw.ts                       # Serwist service worker source
└── proxy.ts                    # Session check proxy
```

---

## 4. Mobile Shell Layout

Consumer web apps render at mobile viewport width (390-430px) and center on larger screens:

### Root Layout

```typescript
// src/app/layout.tsx
import type { Metadata, Viewport } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "App Name",
  description: "App description",
  manifest: "/manifest.json",
};

export const viewport: Viewport = {
  width: "device-width",
  initialScale: 1,
  maximumScale: 1,
  userScalable: false,
  themeColor: "#15803D",
};

export default function RootLayout({
  children,
}: {
  readonly children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className="bg-gray-200">
        <div className="mx-auto max-w-[430px] min-h-screen bg-gray-50 shadow-2xl relative">
          {children}
        </div>
      </body>
    </html>
  );
}
```

**Key dimensions:**

| Property  | Value           | Description                     |
| --------- | --------------- | ------------------------------- |
| max-width | 430px           | iPhone Pro Max baseline         |
| min-width | 360px (natural) | Smallest supported viewport     |
| Outer bg  | `bg-gray-200`   | Visible on tablet/desktop edges |
| Inner bg  | `bg-gray-50`    | Content area background         |
| Shadow    | `shadow-2xl`    | Depth cue on larger screens     |

---

## 5. Bottom Tab Navigation

### BottomTabBar Component

```typescript
"use client";

import { usePathname } from "next/navigation";
import Link from "next/link";
import { Home, Bell, User, type LucideIcon } from "lucide-react";
import type { Route } from "next";

interface Tab {
  readonly label: string;
  readonly href: Route;
  readonly icon: LucideIcon;
}

const tabs: readonly Tab[] = [
  { label: "Home", href: "/home" as Route, icon: Home },
  { label: "Alerts", href: "/notifications" as Route, icon: Bell },
  { label: "Profile", href: "/profile" as Route, icon: User },
] as const;

export function BottomTabBar() {
  const pathname = usePathname();

  return (
    <nav className="fixed bottom-0 left-1/2 -translate-x-1/2 w-full max-w-[430px] h-16 bg-white border-t border-gray-200 flex items-center justify-around z-50">
      {tabs.map((tab) => {
        const isActive = pathname.startsWith(tab.href);
        const Icon = tab.icon;
        return (
          <Link
            key={tab.href}
            href={tab.href}
            className={`flex flex-col items-center gap-0.5 text-xs ${
              isActive ? "text-green-600 font-medium" : "text-gray-400"
            }`}
          >
            <Icon className="w-5 h-5" />
            <span>{tab.label}</span>
          </Link>
        );
      })}
    </nav>
  );
}
```

### (main) Layout

```typescript
// src/app/(main)/layout.tsx
import { BottomTabBar } from "@/components/layout/bottom-tab-bar";

export default function MainLayout({
  children,
}: {
  readonly children: React.ReactNode;
}) {
  return (
    <>
      <div className="pb-16">{children}</div>
      <BottomTabBar />
    </>
  );
}
```

---

## 6. Multi-Step Form Architecture

### Zustand Store with Persist

```typescript
// src/stores/flow-store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface FlowState {
  readonly currentStep: number;
  readonly totalSteps: number;
  readonly stepData: Record<string, unknown>;
}

interface FlowActions {
  readonly setStep: (step: number) => void;
  readonly setStepData: (step: string, data: unknown) => void;
  readonly reset: () => void;
}

const initialState: FlowState = {
  currentStep: 1,
  totalSteps: 4,
  stepData: {},
};

export const useFlowStore = create<FlowState & FlowActions>()(
  persist(
    (set) => ({
      ...initialState,
      setStep: (step) => set({ currentStep: step }),
      setStepData: (step, data) =>
        set((state) => ({
          stepData: { ...state.stepData, [step]: data },
        })),
      reset: () => set(initialState),
    }),
    { name: "flow-draft" },
  ),
);
```

### Per-Step Form with React Hook Form + Zod

```typescript
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useRouter } from "next/navigation";
import { useFlowStore } from "@/stores/flow-store";

const stepSchema = z.object({
  name: z.string().min(1, "Required"),
  age: z.number().int().min(1).max(150),
});

type StepData = z.infer<typeof stepSchema>;

export default function Step1Page() {
  const router = useRouter();
  const { setStepData, setStep } = useFlowStore();

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<StepData>({
    resolver: zodResolver(stepSchema),
  });

  const onSubmit = (data: StepData) => {
    setStepData("step1", data);
    setStep(2);
    router.push("/flow/step-2" as never);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="px-5 py-6 space-y-4">
      <input
        {...register("name")}
        placeholder="Name"
        className="w-full h-12 px-4 border rounded-xl"
      />
      {errors.name && (
        <p className="text-red-500 text-xs">{errors.name.message}</p>
      )}
      <button
        type="submit"
        className="w-full h-12 bg-green-600 text-white rounded-xl font-medium"
      >
        Next
      </button>
    </form>
  );
}
```

### Flow Layout with Progress Bar

```typescript
// src/app/flow/layout.tsx
"use client";

import { useFlowStore } from "@/stores/flow-store";
import { useRouter } from "next/navigation";
import { ChevronLeft } from "lucide-react";

export default function FlowLayout({
  children,
}: {
  readonly children: React.ReactNode;
}) {
  const { currentStep, totalSteps } = useFlowStore();
  const router = useRouter();
  const progress = (currentStep / totalSteps) * 100;

  return (
    <div className="min-h-screen flex flex-col">
      <header className="px-4 py-3 flex items-center gap-3">
        <button onClick={() => router.back()} aria-label="Go back">
          <ChevronLeft className="w-6 h-6" />
        </button>
        <div className="flex-1 h-1 bg-gray-200 rounded-full overflow-hidden">
          <div
            className="h-full bg-green-600 rounded-full transition-all duration-300"
            style={{ width: `${progress}%` }}
          />
        </div>
        <span className="text-xs text-gray-500">
          {currentStep}/{totalSteps}
        </span>
      </header>
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

---

## 7. Page Transitions (Framer Motion)

### Route Transition Wrapper

```typescript
"use client";

import { motion, AnimatePresence } from "framer-motion";
import { usePathname } from "next/navigation";

const variants = {
  hidden: { opacity: 0, x: 20 },
  enter: { opacity: 1, x: 0 },
  exit: { opacity: 0, x: -20 },
} as const;

export function PageTransition({
  children,
}: {
  readonly children: React.ReactNode;
}) {
  const pathname = usePathname();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={pathname}
        variants={variants}
        initial="hidden"
        animate="enter"
        exit="exit"
        transition={{ duration: 0.2 }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
}
```

### Bottom Sheet

```typescript
"use client";

import { motion, AnimatePresence } from "framer-motion";

interface BottomSheetProps {
  readonly isOpen: boolean;
  readonly onClose: () => void;
  readonly children: React.ReactNode;
}

export function BottomSheet({ isOpen, onClose, children }: BottomSheetProps) {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            className="fixed inset-0 bg-black/40 z-40"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />
          <motion.div
            className="fixed bottom-0 left-1/2 -translate-x-1/2 w-full max-w-[430px] bg-white rounded-t-2xl z-50 p-5"
            initial={{ y: "100%" }}
            animate={{ y: 0 }}
            exit={{ y: "100%" }}
            transition={{ type: "spring", damping: 25, stiffness: 300 }}
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
}
```

---

## 8. Data Visualization (Recharts)

### Mobile-Responsive Chart

```typescript
"use client";

import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  ResponsiveContainer,
  Cell,
} from "recharts";

interface ChartItem {
  readonly label: string;
  readonly value: number;
  readonly max: number;
}

interface ScoreChartProps {
  readonly data: readonly ChartItem[];
}

export function ScoreChart({ data }: ScoreChartProps) {
  return (
    <ResponsiveContainer width="100%" height={200}>
      <BarChart data={[...data]} layout="vertical" margin={{ left: 60 }}>
        <XAxis type="number" domain={[0, 100]} hide />
        <YAxis type="category" dataKey="label" width={50} tick={{ fontSize: 12 }} />
        <Bar dataKey="value" radius={[0, 4, 4, 0]} barSize={16}>
          {data.map((entry, index) => (
            <Cell
              key={`cell-${index}`}
              fill={entry.value > entry.max * 0.8 ? "#16A34A" : "#D1D5DB"}
            />
          ))}
        </Bar>
      </BarChart>
    </ResponsiveContainer>
  );
}
```

---

## 9. PWA Configuration

### Web App Manifest

```json
{
  "name": "App Full Name",
  "short_name": "AppName",
  "description": "App description",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#F9FAFB",
  "theme_color": "#15803D",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    {
      "src": "/icons/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/home.png",
      "sizes": "390x844",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ]
}
```

### Required Icon Sizes

| File                     | Size    | Purpose                     |
| ------------------------ | ------- | --------------------------- |
| `icon-192.png`           | 192x192 | Standard icon               |
| `icon-512.png`           | 512x512 | Splash screen               |
| `icon-maskable-512.png`  | 512x512 | Adaptive icon (safe zone)   |

### Serwist Service Worker Source

```typescript
// src/sw.ts
import { defaultCache } from "@serwist/next/worker";
import { Serwist, type PrecacheEntry } from "serwist";

declare const self: ServiceWorkerGlobalScope & {
  __SW_MANIFEST: (PrecacheEntry | string)[];
};

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  runtimeCaching: defaultCache,
});

serwist.addEventListeners();
```

### Public Directory Structure

```
public/
├── manifest.json
├── sw.js                 # Generated by Serwist build
├── icons/
│   ├── icon-192.png
│   ├── icon-512.png
│   └── icon-maskable-512.png
├── screenshots/
│   └── home.png
└── firebase-messaging-sw.js
```

---

## 10. Service Worker Caching Strategies

| Resource Type              | Strategy      | TTL          |
| -------------------------- | ------------- | ------------ |
| App shell (HTML, CSS, JS)  | Cache First   | Build-time   |
| Static assets (images)     | Cache First   | 30 days      |
| API responses (`/api/*`)   | Network First | No cache     |
| Fonts                      | Cache First   | 1 year       |

Serwist's `defaultCache` applies sensible defaults. For custom strategies, override in `src/sw.ts`:

```typescript
import { CacheFirst, NetworkFirst } from "serwist";

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
      handler: new CacheFirst({ cacheName: "google-fonts" }),
    },
    {
      urlPattern: /\/api\/.*/i,
      handler: new NetworkFirst({ cacheName: "api-responses" }),
    },
  ],
});
```

---

## 11. Offline-First UX Patterns

### useOnlineStatus Hook

```typescript
// src/hooks/use-online-status.ts
import { useSyncExternalStore } from "react";

function subscribe(callback: () => void): () => void {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

function getSnapshot(): boolean {
  return navigator.onLine;
}

function getServerSnapshot(): boolean {
  return true;
}

export function useOnlineStatus(): boolean {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

### Offline Banner

```typescript
"use client";

import { useOnlineStatus } from "@/hooks/use-online-status";

export function OfflineBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div className="bg-yellow-50 border-b border-yellow-200 px-4 py-2 text-center text-xs text-yellow-800">
      No internet connection. Showing cached data.
    </div>
  );
}
```

### Submission Blocking Pattern

```typescript
const isOnline = useOnlineStatus();

<button
  type="submit"
  disabled={!isOnline || isSubmitting}
  className="w-full h-12 bg-green-600 text-white rounded-xl disabled:opacity-50"
>
  {!isOnline ? "Waiting for connection..." : "Submit"}
</button>
```

---

## 12. FCM Web Push Notifications

### Firebase Initialization

```typescript
// src/lib/fcm.ts
import { initializeApp } from "firebase/app";
import { getMessaging, getToken, onMessage, type Messaging } from "firebase/messaging";

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FCM_API_KEY,
  projectId: process.env.NEXT_PUBLIC_FCM_PROJECT_ID,
  messagingSenderId: process.env.NEXT_PUBLIC_FCM_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FCM_APP_ID,
};

const app = initializeApp(firebaseConfig);

let messaging: Messaging | null = null;

export function getFirebaseMessaging(): Messaging | null {
  if (typeof window === "undefined") return null;
  if (!messaging) {
    messaging = getMessaging(app);
  }
  return messaging;
}

export async function requestPushToken(): Promise<string | null> {
  const msg = getFirebaseMessaging();
  if (!msg) return null;

  try {
    const permission = await Notification.requestPermission();
    if (permission !== "granted") return null;

    return await getToken(msg, {
      vapidKey: process.env.NEXT_PUBLIC_FCM_VAPID_KEY,
    });
  } catch {
    return null;
  }
}

export function onForegroundMessage(callback: (payload: unknown) => void): () => void {
  const msg = getFirebaseMessaging();
  if (!msg) return () => {};
  return onMessage(msg, callback);
}
```

### Background Service Worker

```javascript
// public/firebase-messaging-sw.js
importScripts("https://www.gstatic.com/firebasejs/12.10.0/firebase-app-compat.js");
importScripts("https://www.gstatic.com/firebasejs/12.10.0/firebase-messaging-compat.js");

firebase.initializeApp({
  apiKey: "REPLACE_AT_BUILD",
  projectId: "REPLACE_AT_BUILD",
  messagingSenderId: "REPLACE_AT_BUILD",
  appId: "REPLACE_AT_BUILD",
});

const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  const { title, body } = payload.notification ?? {};
  if (title) {
    self.registration.showNotification(title, {
      body,
      icon: "/icons/icon-192.png",
      data: payload.data,
    });
  }
});
```

### Permission Request Timing

Request push notification permission **after the user has received value** (e.g., after completing a flow and viewing results), not on first visit. Requesting too early leads to 90%+ denial rates.

**Strategy:**
- Trigger a bottom sheet **3 seconds after the first results page render**, once only
- On dismissal, store `push_prompt_dismissed: true` in localStorage to prevent re-prompting
- On denial (`Notification.permission === "denied"`), respect the browser's decision permanently

```typescript
"use client";

import { useEffect, useRef, useState } from "react";
import { requestPushToken } from "@/lib/fcm";
import { BottomSheet } from "@/components/ui/bottom-sheet";

const PUSH_DISMISSED_KEY = "push_prompt_dismissed";

interface PushPermissionSheetProps {
  readonly isOpen: boolean;
  readonly onClose: () => void;
}

export function PushPermissionSheet({ isOpen, onClose }: PushPermissionSheetProps) {
  const handleAllow = async () => {
    const token = await requestPushToken();
    if (token) {
      await fetch("/api/push-tokens", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ token }),
      });
    }
    localStorage.setItem(PUSH_DISMISSED_KEY, "true");
    onClose();
  };

  const handleDismiss = () => {
    localStorage.setItem(PUSH_DISMISSED_KEY, "true");
    onClose();
  };

  return (
    <BottomSheet isOpen={isOpen} onClose={handleDismiss}>
      <h3 className="text-lg font-semibold mb-2">Enable Notifications?</h3>
      <p className="text-sm text-gray-500 mb-4">
        Get reminders and updates about new recommendations.
      </p>
      <button
        onClick={handleAllow}
        className="w-full h-12 bg-green-600 text-white rounded-xl font-medium mb-2"
      >
        Allow Notifications
      </button>
      <button
        onClick={handleDismiss}
        className="w-full text-sm text-gray-400 py-2"
      >
        Maybe Later
      </button>
    </BottomSheet>
  );
}

/**
 * Hook to trigger push permission prompt once, 3 seconds after mount.
 * Use on results/value-delivery pages only.
 */
export function usePushPrompt(): {
  readonly shouldShow: boolean;
  readonly dismiss: () => void;
} {
  const [shouldShow, setShouldShow] = useState(false);
  const timerRef = useRef<ReturnType<typeof setTimeout>>();

  useEffect(() => {
    const alreadyDismissed = localStorage.getItem(PUSH_DISMISSED_KEY) === "true";
    const alreadyGranted = typeof Notification !== "undefined" && Notification.permission === "granted";
    const alreadyDenied = typeof Notification !== "undefined" && Notification.permission === "denied";

    if (alreadyDismissed || alreadyGranted || alreadyDenied) return;

    timerRef.current = setTimeout(() => setShouldShow(true), 3000);
    return () => clearTimeout(timerRef.current);
  }, []);

  const dismiss = () => {
    setShouldShow(false);
    localStorage.setItem(PUSH_DISMISSED_KEY, "true");
  };

  return { shouldShow, dismiss };
}
```

### Environment Variables

```bash
# FCM Configuration
NEXT_PUBLIC_FCM_API_KEY=
NEXT_PUBLIC_FCM_PROJECT_ID=
NEXT_PUBLIC_FCM_SENDER_ID=
NEXT_PUBLIC_FCM_APP_ID=
NEXT_PUBLIC_FCM_VAPID_KEY=
```

---

## 13. Consumer Auth Pattern

Consumer-facing apps reuse the same JWT RS256 infrastructure from [06-auth-security](./06-auth-security.md) with these differences:

| Aspect           | Admin Frontend          | Consumer Frontend          |
| ---------------- | ----------------------- | -------------------------- |
| Guard chain      | JwtAuth + Roles + Domain | **JwtAuth only**          |
| Domain restriction | `ALLOWED_DOMAIN` env   | **None** (open to all)    |
| Role checking    | `@Roles('admin')`       | **Not used**              |
| Token payload    | `{ sub, email, role }`  | `{ sub, email }` (role ignored) |
| API target       | Admin Service           | User Service (direct)     |

### API Client Base URL

```typescript
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:15320";
```

The consumer frontend targets the **User Service** directly (not the Admin Service).

### Auth Context

Reuse the same `AuthProvider` pattern from [07-admin-nextjs](./07-admin-nextjs.md). The only change is omitting the `role` field from the parsed JWT payload:

```typescript
interface AuthUser {
  readonly id: string;
  readonly email: string;
}
```

### Session Proxy (proxy.ts)

Next.js 16 uses the `proxy.ts` file convention (renamed from the deprecated `middleware.ts`) to intercept requests before they reach routes. The session proxy redirects unauthenticated users to the login page while allowing public routes through:

```typescript
// src/proxy.ts
import { type NextRequest, NextResponse } from "next/server";

const PUBLIC_PATHS = ["/login", "/onboarding", "/terms", "/privacy"] as const;

export function proxy(request: NextRequest): NextResponse {
  const { pathname } = request.nextUrl;

  const isPublic = PUBLIC_PATHS.some((path) => pathname.startsWith(path));
  if (isPublic) {
    return NextResponse.next();
  }

  const token = request.cookies.get("refresh_token")?.value;
  if (!token) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|icons|screenshots|manifest.json|sw.js|firebase-messaging-sw.js).*)",
  ],
};
```

**Key decisions:**

| Aspect | Detail |
| ------ | ------ |
| Cookie checked | `refresh_token` (httpOnly cookie set by the backend) |
| Public paths | Routes under `(auth)` group + legal pages |
| Matcher exclusions | Static assets, PWA manifest, service worker files |
| Redirect | `/login?callbackUrl=<original-path>` for post-login redirect |

> **Note**: The proxy only checks for cookie **presence**, not validity. Token verification happens server-side when the API is called. This keeps the proxy lightweight and avoids importing crypto libraries at the edge.

---

## 14. API Client

Reuse the API client pattern from [07-admin-nextjs](./07-admin-nextjs.md) with these additions:

### TanStack Query Provider

```typescript
// src/lib/query-provider.tsx
"use client";

import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState, type ReactNode } from "react";

export function QueryProvider({ children }: { readonly children: ReactNode }) {
  const [client] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 5 * 60 * 1000,
            retry: 1,
          },
        },
      }),
  );

  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

### Query Keys Pattern

```typescript
// src/lib/query-keys.ts
export const queryKeys = {
  profile: {
    all: ["profile"] as const,
    current: () => [...queryKeys.profile.all, "current"] as const,
  },
  results: {
    all: ["results"] as const,
    latest: () => [...queryKeys.results.all, "latest"] as const,
    detail: (id: string) => [...queryKeys.results.all, "detail", id] as const,
  },
  notifications: {
    all: ["notifications"] as const,
    list: () => [...queryKeys.notifications.all, "list"] as const,
    unreadCount: () => [...queryKeys.notifications.all, "unread"] as const,
  },
} as const;
```

---

## Verification

```bash
# Typecheck
pnpm --filter @scope/web run typecheck

# ESLint
pnpm --filter @scope/web run lint

# Dev server
pnpm --filter @scope/web run dev
# → http://localhost:15300

# Build (generates standalone output + SW)
pnpm --filter @scope/web run build

# Verify PWA manifest
curl -s http://localhost:15300/manifest.json | head

# Verify Service Worker registration (in browser DevTools)
# Application → Service Workers → Status: activated
```
