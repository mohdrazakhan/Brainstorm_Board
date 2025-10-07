# AI-Powered Brainstorming Board – Architecture & Implementation Plan

## 1. Product Overview
The goal is to build a Trello/Notion-style brainstorming board that lets users curate ideas, collaborate with AI for creativity boosts, and keep their board state synchronized across sessions. The app must deliver:

- Simple per-user authentication.
- Persistent drag-and-drop boards with CRUD for idea cards.
- AI features for idea suggestions, clustering, and summarization.
- A polished UI with a left command toolbar and right AI insights panel.
- Deployment assets: GitHub repository, README, and demo video.

## 2. Technology Stack
| Layer | Choice | Rationale |
| --- | --- | --- |
| Frontend | Next.js 14 (App Router) + React + TypeScript | Modern full-stack React framework with built-in routing, server actions, and optimized rendering. |
| Styling | Tailwind CSS + shadcn/ui components | Rapid UI development with consistent styling and accessible components. |
| Drag & Drop | `@hello-pangea/dnd` | Mature drag & drop library compatible with modern React. |
| Authentication | NextAuth.js (Credentials provider) | Lightweight username-based login with session management handled automatically. |
| Backend | Next.js API routes / server actions | Co-located backend logic with front-end for simpler deployment. |
| Database | Prisma ORM + SQLite (dev) / PostgreSQL (prod-ready) | Quick local setup with Prisma migrations, easy to switch to Postgres deployment. |
| AI Provider | OpenAI API (Responses + Embeddings endpoints) | Reliable models for text generation and embeddings; easily replaceable if needed. |
| State Management | React Query (TanStack Query) | Client-side caching for board data and AI interactions. |
| Testing | Vitest + Testing Library + Playwright (E2E smoke) | Fast unit/component tests; optional UI automation for drag/drop. |
| Tooling | ESLint, Prettier, Husky pre-commit hooks | Maintain code quality and formatting. |

## 3. Domain Model
Defined via Prisma schema:

```
model User {
  id        String   @id @default(cuid())
  username  String   @unique
  email     String?  @unique
  boards    Board[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Board {
  id            String    @id @default(cuid())
  user          User      @relation(fields: [userId], references: [id])
  userId        String
  title         String    @default("My Brainstorm")
  columns       Column[]
  cards         Card[]
  suggestions   SuggestedIdea[]
  summaries     BoardSummary[]
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

model Column {
  id        String  @id @default(cuid())
  board     Board   @relation(fields: [boardId], references: [id])
  boardId   String
  title     String
  position  Int
  cards     Card[]
}

model Card {
  id           String   @id @default(cuid())
  board        Board    @relation(fields: [boardId], references: [id])
  boardId      String
  column       Column?  @relation(fields: [columnId], references: [id])
  columnId     String?
  title        String
  description  String?
  aiGenerated  Boolean  @default(false)
  clusterLabel String?
  embedding    Json?
  mood         String?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

model SuggestedIdea {
  id        String   @id @default(cuid())
  board     Board    @relation(fields: [boardId], references: [id])
  boardId   String
  sourceCard Card?   @relation(fields: [sourceCardId], references: [id])
  sourceCardId String?
  content   String
  accepted  Boolean  @default(false)
  createdAt DateTime @default(now())
}

model BoardSummary {
  id        String   @id @default(cuid())
  board     Board    @relation(fields: [boardId], references: [id])
  boardId   String
  content   String
  createdAt DateTime @default(now())
}
```

> Note: `Json` column holds serialized embedding vectors (array of floats). Switching to Postgres enables native `vector` type if desired.

## 4. Key Application Flows

### 4.1 Authentication
1. User arrives at `/login`.
2. Submits username (and optional email). No password required per brief.
3. NextAuth Credentials provider:
   - Upserts user by username.
   - Issues session cookie.
4. Authenticated user redirected to `/board` (their unique board).

### 4.2 Board CRUD & Drag/Drop
- React Query fetches board snapshot from `/api/board` (server action ensures user scoping).
- Drag & drop uses `@hello-pangea/dnd`.
- On drop or edit, optimistic update + mutation to `/api/board/cards` to persist. Prisma ensures column ordering updates.

### 4.3 AI Idea Suggestions
- When a card is created, client triggers `/api/ai/suggest` with card title/description.
- Endpoint calls OpenAI Responses API to produce 2–3 bullet ideas.
- Suggestions stored in `SuggestedIdea` table and shown beneath the card (with “Add to board” button).

### 4.4 AI Clustering
- Background server action `/api/ai/cluster` fetches all card embeddings (computing if missing with OpenAI Embeddings API).
- Runs K-Means clustering (using `ml-kmeans` or custom implementation) to assign `clusterLabel`.
- Client displays clusters via colored badges + optional section grouping (filter by cluster).

### 4.5 Board Summarization
- `/api/ai/summarize` composes board context (top cards, clusters) and calls OpenAI for summary (themes, top ideas, next steps).
- Result saved in `BoardSummary` history; latest shown in right panel.

### 4.6 Right Panel Insights
- Displays latest summary.
- Scrollable log of AI suggestions and actions taken.

## 5. API Surface

| Endpoint | Method | Description |
| --- | --- | --- |
| `/api/auth/[...nextauth]` | `POST/GET` | NextAuth handler for sessions. |
| `/api/board` | `GET` | Fetch user board snapshot (columns, cards, clusters). |
| `/api/board/card` | `POST` | Create card; triggers AI suggestion pipeline. |
| `/api/board/card/:id` | `PATCH` | Update card details or column. |
| `/api/board/card/:id` | `DELETE` | Remove card & associated suggestions. |
| `/api/board/column` | `POST/PATCH/DELETE` | Manage columns. |
| `/api/ai/suggest` | `POST` | Generate related ideas for a card. |
| `/api/ai/cluster` | `POST` | Force re-cluster of board cards. |
| `/api/ai/summarize` | `POST` | Generate latest board summary. |
| `/api/ai/mood`* | `POST` | (Bonus) Derive sentiment labels per card. |
| `/api/ai/search`* | `GET` | (Bonus) Semantic search within board ideas. |

`*` denotes optional bonus endpoints.

## 6. Frontend Layout

- **Top Navbar**: App title, user menu (logout).
- **Left Toolbar**: Buttons for Add Card (opens modal), Cluster Now, Summarize Board, optional export/search controls.
- **Main Canvas**: Responsive grid of columns with drag & drop cards. Cards show cluster color, mood icon, suggestion toggles.
- **Right Panel**: Collapsible panel showing latest summary, previous summaries, and suggestion log (with filters).
- **Modals**: Card edit modal, confirm delete, settings.

Responsive breakpoints ensure vertical stacking on mobile.

## 7. AI Prompting Guidelines

- **Idea Suggestions**: Provide card title/description, board context (up to 5 sibling cards), ask for 3 short suggestions with titles + optional description.
- **Clustering**: Use embeddings `text-embedding-3-small`. K-Means `k` determined dynamically (sqrt(n)) with fallback to heuristics for small boards.
- **Summarization**: Provide board metadata (columns, cluster labels, top cards) and request structured JSON with `themes`, `topIdeas`, `nextSteps` for consistent rendering.
- **Mood Analysis (Bonus)**: Single prompt per card summarizing sentiment, storing `positive|neutral|negative`.

## 8. Deployment Strategy

1. **Environment Variables**:
   - `DATABASE_URL` (SQLite default or Postgres connection).
   - `NEXTAUTH_SECRET`
   - `OPENAI_API_KEY`
2. **Hosting**: Deploy to Vercel (frontend + API) with Supabase/Neon for Postgres. Alternatively, host on Railway/Fly for combined setup.
3. **Database Migration**: Run `npx prisma migrate deploy` during build.
4. **Demo Video**: Record Loom demo covering auth, DnD, AI features, summarization.

## 9. Milestones

1. **M1 – Project Setup**: Scaffold Next.js + Tailwind + Prisma + NextAuth, configure lint/test tools.
2. **M2 – Core Board CRUD**: Implement database schema, board endpoints, drag & drop UI, persistence.
3. **M3 – AI Suggestion Pipeline**: Integrate OpenAI for card suggestions and logging.
4. **M4 – Clustering & Visualization**: Embed cards, run clustering, display cluster UI.
5. **M5 – Summarization & Insights Panel**: Summaries, history, UI integration.
6. **M6 – Polish & Bonus Features**: Mood analysis, export, collaborative boards (optional).
7. **M7 – Testing & Documentation**: Automated tests, README, deployment instructions, demo video script.

## 10. Risks & Mitigations
- **AI Cost/Latency**: Batch embedding requests, cache embeddings in DB, expose rate limiting.
- **Drag & Drop Complexity**: Isolate DnD logic in dedicated provider; maintain column/card ordering fields.
- **Authentication Simplicity vs Security**: While spec allows simple auth, document limitations and path to stronger auth.
- **Clustering Accuracy**: Provide manual override to reassign cluster; display labels as AI-generated.

## 11. Bonus Feature Ideas
- **Collaborative Boards**: Introduce `BoardMember` join table with permissions, real-time updates via Pusher/Supabase Realtime.
- **Export**: Generate Markdown summary server-side, optional PDF using `@react-pdf/renderer`.
- **Semantic Search**: Use embedding similarity to return matching cards.
- **Mood Analysis**: Extend AI pipeline with sentiment to color-code cards.

---
This plan guides the subsequent implementation tasks in the project todo list. Adjust as needed once development begins.
