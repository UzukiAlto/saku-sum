# SakuSum（サクサム）設計書 ＆ 実装計画

## 1. アーキテクチャ方針（Next.js 16 App Router × Firebase）

Next.js（App Router）の実践的な学習とポートフォリオとしての品質を両立するため、**サーバー側（Server Components / Server Actions / Route Handlers + Firebase Admin SDK）** と **クライアント側（Client Components + Firebase Client SDK）** の役割を明確に分離した**ハイブリッド構成**を採用する。

| 層・機能 | 実行場所 | 主な技術 | 設計意図（Next.js App Router） |
| :--- | :--- | :--- | :--- |
| **公開 LP（`/`）** | Server Component | `getSession()`（DAL） | ログイン済みセッションがあればサーバー側で即 `/dashboard` へ `redirect()` し、画面チラつきを防ぐ |
| **認証ポップアップ / 匿名開始 / アカウント連携** | Client Component | Firebase Client SDK | 匿名認証や Google ポップアップ認証はブラウザ側でのみ実行可能。取得した ID トークンのみサーバーへ渡す |
| **セッション Cookie 発行・削除** | Route Handler（`/api/auth/session`） | Firebase Admin SDK + `await cookies()` | ブラウザ JS から読み取れない `httpOnly` Cookie をサーバー側で発行・削除する |
| **認証ガード・初期データ取得** | Server Components / DAL | `import 'server-only'` + Firebase Admin SDK | DB 通信と管理者認証情報をブラウザに出さず、サーバー側で安全に検証・取得する |
| **支出記録・Undo・編集・削除・カテゴリ CRUD** | Server Actions | Zod + Firebase Admin SDK + `revalidatePath` | 実行時バリデーション（Zod）と認証チェック（DAL）をサーバー側で強制しつつ、型安全にデータを更新する |
| **1タップ即時反映・トースト・モーダル** | Client Component | React 19 `useOptimistic` / `useTransition` | タップ直後に合計金額を楽観更新し、失敗時は自動ロールバックして「サクッと感」と整合性を両立する |

> **Next.js 16 における認証検証の原則**
> Next.js 16 では従来の `middleware.ts` が `proxy.ts` へ変更されているが、公式ドキュメントの推奨に従い、リクエスト境界（Proxy）だけに認証を依存させず、**Data Access Layer（DAL：`src/lib/auth/session.ts`）とすべての Server Actions の先頭で必ずセッション Cookie を検証する**設計とする。

---

## 2. 画面構成・ルーティング設計

App Router の **Route Group `(app)`** を利用し、公開 LP（`/`）とログイン後画面（共通ヘッダー＋モバイル下部ボトムナビ）のレイアウトを分離する。

| URL | ファイルパス（`src/app/` 配下） | 役割・主な機能 |
| :--- | :--- | :--- |
| `/` | `page.tsx` | 公開 LP（「登録なしで今すぐ試す」「Google でログイン」）。有効セッションがあれば `/dashboard` へリダイレクト |
| `/dashboard` | `(app)/dashboard/page.tsx` | 今月＆今日の合計、月切り替え（`?month=YYYY-MM`）、1タップ記録ボタン群（`useOptimistic`）、カテゴリ別内訳（Tailwind 割合バー）、直後 Undo トースト |
| `/history` | `(app)/history/page.tsx` | 選択月の支出履歴一覧（`?month=YYYY-MM`）、各支出の金額・日付・カテゴリ・メモ編集、削除 |
| `/categories` | `(app)/categories/page.tsx` | カテゴリ一覧、追加・編集（名前・目安金額・アイコン/色）・削除 |
| `/settings` | `(app)/settings/page.tsx` | アカウント状態確認、匿名→Google アカウント連携（既存アカウント衝突ハンドリング含む）、ログアウト |
| `/api/auth/session` | `api/auth/session/route.ts` | `POST`：ID トークン検証・セッション Cookie 発行・初回プリセット作成 / `DELETE`：Cookie 削除 |

---

## 3. ディレクトリ構成

```text
src/
├── app/
│   ├── layout.tsx                  # ルートレイアウト（フォント・メタデータ・全体ベース）
│   ├── globals.css                 # Tailwind CSS v4 設定
│   ├── page.tsx                    # 公開 LP（Server Component）
│   ├── (app)/
│   │   ├── layout.tsx              # アプリ共通レイアウト（認証ガード・モバイル幅・ヘッダー・ボトムナビ）
│   │   ├── loading.tsx             # 画面遷移時のローディング UI
│   │   ├── error.tsx               # エラーバウンダリ UI（Client Component）
│   │   ├── dashboard/page.tsx      # 記録＆集計画面（Server Component）
│   │   ├── history/page.tsx        # 履歴一覧＆編集画面（Server Component）
│   │   ├── categories/page.tsx     # カテゴリ管理画面（Server Component）
│   │   └── settings/page.tsx       # アカウント設定・連携画面（Server Component）
│   └── api/
│       └── auth/session/route.ts   # セッション Cookie 発行・削除 Route Handler
├── actions/                        # Server Actions（'use server'）
│   ├── expenses.ts                 # 支出の作成・削除（Undo 兼用）・更新
│   └── categories.ts               # カテゴリの作成・更新・削除
├── components/
│   ├── layout/                     # BottomNav, AppHeader 等
│   ├── auth/                       # LoginButtons, AccountLinkCard, LogoutButton 等
│   ├── dashboard/                  # DashboardClient, QuickRecordGrid, SummaryCard, CategoryBreakdown, UndoToast
│   ├── history/                    # ExpenseList, ExpenseEditModal, MonthSwitcher
│   └── categories/                 # CategoryList, CategoryFormModal
├── lib/
│   ├── firebase/
│   │   ├── client.ts               # Firebase Client SDK 初期化
│   │   └── admin.ts                # Firebase Admin SDK 初期化（import 'server-only'）
│   ├── auth/
│   │   └── session.ts              # セッション検証 DAL（verifySession / getSession）
│   ├── firestore/
│   │   ├── users.ts                # ユーザー初期化・初期プリセット生成
│   │   ├── categories.ts           # カテゴリ取得（サーバー側）
│   │   └── expenses.ts             # 月別支出取得（サーバー側）
│   ├── domain/                     # 純粋関数群（Vitest 単体テスト対象）
│   │   ├── aggregation.ts          # 今月合計・今日合計・カテゴリ別集計・構成比計算
│   │   ├── date.ts                 # JST（Asia/Tokyo）基準の日付・YYYY-MM 計算
│   │   └── presets.ts              # 初期プリセット・カラー/アイコン定義
│   └── validation/                 # Zod バリデーションスキーマ
│       ├── expense.ts
│       └── category.ts
└── types/
    └── index.ts                    # アプリ共通 TypeScript 型定義
```

---

## 4. Cloud Firestore データ構造 ＆ セキュリティルール

個人専用アプリのため、ユーザーごとのサブコレクション構成（`users/{uid}/...`）を採用する。
支出ドキュメントには**記録時点のカテゴリ名・色・金額のスナップショット**を保存し、後からカテゴリの目安金額や名前を変更しても過去の集計がズレないようにする。

### 4.1 コレクション設計

#### `users/{uid}`
| フィールド | 型 | 必須 | 説明 |
| :--- | :--- | :---: | :--- |
| `initializedAt` | `Timestamp` | ○ | 初期プリセット生成済み判定用（冪等性の担保） |
| `createdAt` | `Timestamp` | ○ | ドキュメント作成日時 |

#### `users/{uid}/categories/{categoryId}`
| フィールド | 型 | 必須 | 説明 |
| :--- | :--- | :---: | :--- |
| `name` | `string` | ○ | カテゴリ名（1〜20文字程度） |
| `defaultAmount` | `number` | ○ | 1タップ時の目安金額（正の整数） |
| `icon` | `string` | ○ | `lucide-react` 用のアイコン識別子（例：`coffee`, `utensils`） |
| `color` | `string` | ○ | プリセットカラーキー（例：`sky`, `amber`, `emerald`） |
| `order` | `number` | ○ | 表示順（昇順ソート用） |
| `createdAt` | `Timestamp` | ○ | 作成日時 |
| `updatedAt` | `Timestamp` | ○ | 更新日時 |

#### `users/{uid}/expenses/{expenseId}`
| フィールド | 型 | 必須 | 説明 |
| :--- | :--- | :---: | :--- |
| `categoryId` | `string` | ○ | 記録時に選択したカテゴリ ID |
| `categoryName` | `string` | ○ | **記録時点の**カテゴリ名スナップショット |
| `categoryColor` | `string` | ○ | **記録時点の**カラーキー |
| `amount` | `number` | ○ | 支出金額（正の整数） |
| `spentDate` | `string` | ○ | 支出日（JST 基準 `"YYYY-MM-DD"`） |
| `yearMonth` | `string` | ○ | 集計対象月（JST 基準 `"YYYY-MM"`。月別クエリ用） |
| `memo` | `string` | ○ | メモ（任意、未入力時は空文字 `""`） |
| `createdAt` | `Timestamp` | ○ | 作成日時 |
| `updatedAt` | `Timestamp` | ○ | 更新日時 |

### 4.2 集計・タイムゾーン設計
- **月別集計:** 対象月の `expenses` を `where('yearMonth', '==', targetMonth)` で取得し、TypeScript の純粋関数（`src/lib/domain/aggregation.ts`）で「今月の合計」「今日の合計」「カテゴリ別合計・割合」を算出する。
- **JST 基準の日付処理:** 本番ホスティング（Vercel）のサーバーは UTC で動作するため、日付・月計算は必ず `Asia/Tokyo` タイムゾーンを明示した純粋関数（`src/lib/domain/date.ts`）で行い、日本時間 0:00〜8:59 の日付ズレを防ぐ。

### 4.3 Firestore セキュリティルール方針
- リポジトリ直下に `firestore.rules` を配置し、バージョン管理する。
- 認証済みユーザーが自身の `uid` 配下のドキュメント（`users/{userId}/{document=**}`）のみ読み書きできるルールを設定し、他ユーザーのデータへの直接アクセスを完全に遮断する。

---

## 5. 認証フロー設計

### 5.1 匿名ログイン開始（`/` → `/dashboard`）
1. Client Component で Firebase Client SDK の `signInAnonymously(auth)` を実行し、`idToken` を取得。
2. `POST /api/auth/session` へ `idToken` を送信。
3. Route Handler で Firebase Admin SDK の `createSessionCookie(idToken, { expiresIn })` を生成し、`httpOnly` / `sameSite: 'lax'` / `path: '/'`（本番時は `secure: true`）で Cookie に保存。
4. `users/{uid}` が未初期化であれば、初期プリセットカテゴリ 6 件を一括作成（`WriteBatch` / トランザクション等で冪等に実行）。
5. クライアント側で `/dashboard` へ遷移し、サーバーコンポーネントを再評価する。

### 5.2 Google ログイン（`/` → `/dashboard`）
1. Client Component で `signInWithPopup(auth, googleProvider)` を実行し、`idToken` を取得。
2. `POST /api/auth/session` でセッション Cookie を発行し、未初期化であれば初期プリセットを作成。
3. `/dashboard` へ遷移。

### 5.3 匿名利用から Google アカウント連携（`/settings`）
1. Client Component で `onAuthStateChanged` により現在の匿名ユーザーを確認し、`linkWithPopup(currentUser, googleProvider)` を実行。
2. **成功時:** 同一 `uid` のまま Google アカウントへ昇格。`getIdToken(true)` で更新済み ID トークンを再取得し、`POST /api/auth/session` でセッション Cookie を更新する（データはすべて引き継がれる）。
3. **衝突時（`auth/credential-already-in-use`）:** 要件定義に従い、確認ダイアログ（「すでに登録済みの Google アカウントです。切り替えると現在の匿名データは引き継がれません」）を表示。ユーザーが同意した場合のみ、取得済み認証情報で既存アカウントにサインインし、`POST /api/auth/session` でセッションを差し替えて `/dashboard` へ遷移する。

### 5.4 ログアウト（`/settings`）
1. 匿名ユーザーの場合は「連携していないためデータが失われる」旨の確認ダイアログを表示。
2. Client SDK の `signOut(auth)` と `DELETE /api/auth/session`（Cookie 削除）を実行し、`/` へ遷移する。

---

## 6. 追加パッケージ ＆ テスト戦略

### 6.1 追加予定パッケージ
- **本番依存（`dependencies`）:**
  - `firebase`（Client SDK：認証ポップアップ・匿名認証）
  - `firebase-admin`（Admin SDK：セッション検証・Firestore 操作）
  - `server-only`（サーバー専用モジュールのクライアント誤インポート防止）
  - `zod`（Server Actions・フォーム入力の実行時バリデーション）
  - `lucide-react`（UI・カテゴリ用アイコン）
- **開発・テスト依存（`devDependencies`）:**
  - `vitest`、`@vitejs/plugin-react`、`jsdom`、`vite-tsconfig-paths`
  - `@testing-library/react`、`@testing-library/dom`、`@testing-library/jest-dom`、`@testing-library/user-event`

### 6.2 テスト戦略
- **単体テスト（Vitest）:**
  - `src/lib/domain/date.ts`（UTC/JST 境界時刻、前月・次月遷移、年またぎ）
  - `src/lib/domain/aggregation.ts`（今月合計、今日合計、カテゴリ別集計・割合、0件時）
  - `src/lib/validation/*.ts`（正常値・負数・0円・空文字・長すぎる文字列等の境界値）
- **コンポーネントテスト（Vitest + React Testing Library）:**
  - Client Component・表示専用コンポーネントを中心に、1タップボタンの描画とクリック発火、0件時の空状態表示、フォームバリデーションエラー表示などを検証する（`async` Server Component は Vitest 非対応のため対象外とする）。

---

## 7. 実装ロードマップ（タスク一覧 ＆ Git ブランチ計画）

原則として **1 タスク ＝ 1 機能ブランチ** で進め、各タスク完了時に動作確認・コードレビューを経て `main` へマージする。

| Task | 内容 | 推奨ブランチ名 | 完了条件（Done の定義） |
| :---: | :--- | :--- | :--- |
| **1** | **開発基盤セットアップ**：Firebase プロジェクト作成（Auth / Firestore 有効化）、環境変数（`.env.local` / `.env.example`）、追加パッケージ導入、Vitest 設定 | `chore/setup-foundation` | `npm run dev` / `npm run test` が通り、環境変数の雛形が整っている |
| **2** | **ドメインロジック＆単体テスト**：共通型定義、JST 日付関数、集計関数、初期プリセット定義、Zod スキーマ、Vitest 単体テスト | `feature/domain-logic` | 純粋関数と Zod スキーマの単体テストがすべて Pass する |
| **3** | **Firebase 初期化＆セッション認証基盤**：Client/Admin SDK、`/api/auth/session`（POST/DELETE）、DAL（`getSession` / `verifySession`）、初回プリセット生成ロジック | `feature/auth-session` | ID トークンからセッション Cookie 発行・検証・削除が動作する |
| **4** | **公開 LP ＆ ログイン導線**：`/` LP UI、匿名開始／Google ログインボタン、ログイン済み時のサーバー側リダイレクト | `feature/landing-and-login` | 未ログインで LP 表示、匿名・Google いずれかで `/dashboard` へ遷移できる |
| **5** | **アプリ共通シェル**：`(app)/layout.tsx`（認証ガード・モバイル幅・ヘッダー・ボトムナビ）、`loading.tsx`、`error.tsx` | `feature/app-shell` | 未ログイン時に `/dashboard` 等へ直接アクセスすると `/` へ戻され、ログイン後は4画面をボトムナビで移動できる |
| **6** | **ダッシュボード＆1タップ記録**：合計カード、月切り替え（`?month=YYYY-MM`）、1タップ記録 Server Action、`useOptimistic` 即時反映、直後 Undo トースト、カテゴリ割合バー | `feature/dashboard-quick-record` | 1タップで即座に合計・内訳へ反映され、リロード後も永続化、Undo で即取り消しできる |
| **7** | **支出履歴・編集・削除**：`/history` 月別履歴一覧、編集モーダル（金額・日付・カテゴリ・メモ）、削除 | `feature/expense-history` | 履歴の閲覧・月切替・編集・削除がすべて動作し、ダッシュボード集計にも正しく反映される |
| **8** | **カテゴリ管理 CRUD**：`/categories` 一覧、新規追加・編集（名前・目安金額・アイコン/色）・削除 | `feature/category-management` | カテゴリ CRUD が動作し、編集・削除後も過去の支出履歴の金額・名前が維持される |
| **9** | **アカウント連携＆ログアウト**：`/settings` 状態表示、匿名→Google 連携（既存アカウント衝突確認ダイアログ含む）、匿名ログアウト警告 | `feature/account-linking` | 匿名データを保持したまま Google アカウントへ引き継げ、衝突・ログアウト警告も仕様どおり動作する |
| **10** | **品質仕上げ＆本番公開**：主要 UI のコンポーネントテスト、`firestore.rules` 整備・適用、メタデータ/OGP 調整、Vercel 本番デプロイ | `feature/polish-and-deploy` | 全テスト・ビルド（`npm run build`）が通り、Vercel 本番 URL で全機能がスマホから動作する |
