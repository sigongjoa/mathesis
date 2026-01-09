# Frontend Project Structure & Conventions

## 1. Directory Structure

We use a Monorepo-style structure where shared code lives in `packages/common` (conceptually) or `src/common` and feature code lives in feature modules.

```
src/
├── app/                    # App-wide providers & shell
├── common/                 # Shared resources
│   ├── components/         # Atomic UI (Button, Card)
│   ├── hooks/              # Custom React hooks
│   ├── utils/              # Helper functions
│   └── styles/             # Global styles / Tailwind config
├── nodes/                  # Feature Modules (Node 0-6)
│   ├── node0/              # Student Hub
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── pages/
│   │   ├── store/          # Local State (Zustand/Context)
│   │   └── types/
│   ├── node1/              # Logic Engine
│   └── ...
└── services/               # API Clients (Axios/TanStack Query)
```

## 2. Naming Conventions

### 2.1 Files & Folders
- **Components**: PascalCase (e.g., `StudentCard.tsx`, `UserProfile.tsx`)
- **Hooks**: camelCase with use prefix (e.g., `useAuth.ts`, `useStudentData.ts`)
- **Utilities**: camelCase (e.g., `formatDate.ts`, `apiClient.ts`)
- **Types**: PascalCase (e.g., `Student.ts`, `Intervention.ts`)

### 2.2 Code
- **React Components**: PascalCase
- **Functions/Variables**: camelCase
- **Constants**: UPPER_SNAKE_CASE
- **Interfaces/Types**: PascalCase

## 3. State Management Strategy

### 3.1 Global State (client-state)
- Use **Zustand** or **Redux Toolkit** for session data, user preferences, and cross-node data (e.g., current selected student for global context).

### 3.2 Server State (server-state)
- Use **TanStack Query (React Query)** for all API data fetching, caching, and synchronization.
- **Do not** store API responses in global state stores unless absolutely necessary for complex client-side manipulation.

### 3.3 Local State (UI)
- Use `useState` and `useReducer` for form inputs, modal visibility, and temporary UI states.
