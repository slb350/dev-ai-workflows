# GraphQL Development Workflow - Schema-First API Design

**Version:** 1.0
**Last Updated:** 2025-01-24
**Purpose:** Comprehensive workflow for building type-safe, tested GraphQL APIs with TypeScript

---

## Overview

This workflow prioritizes **type safety**, **schema design**, and **comprehensive testing** through Test-Driven Development with GraphQL. Build APIs that are self-documenting, type-safe from schema to implementation, and thoroughly tested.

---

## Core Principles

1. **Schema First** - Design the API contract before implementation
2. **Type Safety Everywhere** - Generated types from schema to resolvers to tests
3. **Tests First, Always** - Write tests for resolvers before implementation
4. **Self-Documenting** - Schema serves as API documentation
5. **No Versioning** - Evolve schema through additive changes and deprecation
6. **Resolver Simplicity** - Keep resolvers thin, delegate to service layer
7. **Track Everything** - Keep visible task tracker (TodoWrite, Linear, checklist)

---

## The Workflow (Red-Green-Refactor-Commit)

### Phase 1: Planning & Setup

```bash
# 1. Initialize Node.js project
mkdir graphql-api && cd graphql-api
npm init -y
npm pkg set type="module"

# 2. Install core dependencies
npm install graphql @apollo/server graphql-tag

# 3. Install TypeScript and development tools
npm install -D typescript @types/node tsx
npm install -D @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-resolvers
npm install -D @graphql-typed-document-node/core

# 4. Install testing dependencies
npm install -D vitest @vitest/coverage-v8 @graphql-tools/mock @graphql-tools/schema

# 5. Install code quality tools
npm install -D prettier eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# 6. Initialize TypeScript
npx tsc --init
```

**Create Todo List:**

```typescript
// Track todos in your preferred tool (TodoWrite, Linear, checklist):
// - Define GraphQL schema for feature
// - Generate TypeScript types from schema
// - Write failing resolver tests
// - Implement resolvers
// - Test queries/mutations end-to-end
// - Format, lint, commit
```

### Phase 2: Project Structure

```text
graphql-api/
├── src/
│   ├── schema/
│   │   ├── index.ts              # Combined schema
│   │   ├── typeDefs/
│   │   │   ├── user.graphql      # User type definitions
│   │   │   ├── post.graphql      # Post type definitions
│   │   │   └── common.graphql    # Shared types (Date, PageInfo, etc.)
│   │   └── resolvers/
│   │       ├── index.ts          # Combined resolvers
│   │       ├── user.ts           # User resolvers
│   │       ├── post.ts           # Post resolvers
│   │       └── scalars.ts        # Custom scalar resolvers
│   ├── services/
│   │   ├── userService.ts        # Business logic for users
│   │   └── postService.ts        # Business logic for posts
│   ├── datasources/
│   │   ├── database.ts           # Database client
│   │   └── cache.ts              # Redis/caching layer
│   ├── context.ts                # GraphQL context type
│   ├── server.ts                 # Apollo Server setup
│   └── index.ts                  # Entry point
├── tests/
│   ├── unit/
│   │   ├── resolvers/
│   │   │   ├── user.test.ts
│   │   │   └── post.test.ts
│   │   └── services/
│   │       └── userService.test.ts
│   └── integration/
│       └── queries.test.ts
├── generated/
│   └── graphql.ts                # Generated types (gitignored)
├── codegen.ts                    # GraphQL Code Generator config
├── tsconfig.json
├── vitest.config.ts
├── justfile
└── package.json
```

---

## Automation (`justfile`)

```just
set shell := ["bash", "-c"]

# Load environment from .env
set dotenv-load := true

# Default recipe (runs when you just type 'just')
default:
    @just --list

# Generate TypeScript types from GraphQL schema
codegen:
    graphql-codegen --config codegen.ts

# Watch mode for codegen (useful during development)
codegen-watch:
    graphql-codegen --config codegen.ts --watch

# Run TypeScript compiler
typecheck:
    tsc --noEmit

# Format code
fmt:
    prettier --write "src/**/*.{ts,graphql}" "tests/**/*.ts"

# Check formatting without modifying
fmt-check:
    prettier --check "src/**/*.{ts,graphql}" "tests/**/*.ts"

# Lint code
lint:
    eslint "src/**/*.ts" "tests/**/*.ts"

# Fix lint issues
lint-fix:
    eslint "src/**/*.ts" "tests/**/*.ts" --fix

# Run all tests
test:
    vitest run

# Run tests in watch mode
test-watch:
    vitest watch

# Run tests with coverage
test-cov:
    vitest run --coverage

# Run tests with minimum coverage threshold
test-cov-gate:
    vitest run --coverage --coverage.thresholds.lines=80 --coverage.thresholds.functions=80

# Run only unit tests
test-unit:
    vitest run tests/unit

# Run only integration tests
test-integration:
    vitest run tests/integration

# Full check (CI pipeline)
check:
    just codegen
    just fmt-check
    just lint
    just typecheck
    just test-cov-gate

# Development server
dev:
    tsx watch src/index.ts

# Build for production
build:
    just codegen
    just typecheck
    tsc

# Start production server
start:
    node dist/index.js
```

---

## Phase 3: TDD Cycle (For Each Feature)

### Step 1: RED - Define Schema & Write Failing Tests

#### Define GraphQL Schema

```graphql
# src/schema/typeDefs/user.graphql
"""
Represents a user in the system.
"""
type User {
  """Unique identifier for the user."""
  id: ID!

  """User's email address."""
  email: String!

  """User's display name."""
  name: String!

  """
  User's posts.
  Uses cursor-based pagination for scalability.
  """
  posts(
    first: Int = 10
    after: String
  ): PostConnection!

  """Timestamp when user was created."""
  createdAt: DateTime!
}

"""
Paginated connection of posts.
"""
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

extend type Query {
  """
  Get a user by ID.
  Returns null if user doesn't exist.
  """
  user(id: ID!): User

  """
  Get all users with pagination.
  """
  users(
    first: Int = 10
    after: String
  ): UserConnection!
}

extend type Mutation {
  """
  Create a new user.
  """
  createUser(input: CreateUserInput!): CreateUserPayload!
}

"""Input for creating a new user."""
input CreateUserInput {
  email: String!
  name: String!
}

"""Payload returned when creating a user."""
type CreateUserPayload {
  user: User!
  userEdge: UserEdge!
}

"""
Custom DateTime scalar.
Serialized as ISO 8601 string.
"""
scalar DateTime
```

#### Generate Types

```bash
# Run code generator to create TypeScript types
just codegen
```

This generates types in `generated/graphql.ts`:

```typescript
// generated/graphql.ts (auto-generated)
export type User = {
  __typename?: 'User';
  id: Scalars['ID'];
  email: Scalars['String'];
  name: Scalars['String'];
  posts: PostConnection;
  createdAt: Scalars['DateTime'];
};

export type Resolvers<ContextType = Context> = {
  Query?: QueryResolvers<ContextType>;
  Mutation?: MutationResolvers<ContextType>;
  User?: UserResolvers<ContextType>;
  // ... more resolver types
};

export type QueryResolvers<ContextType = Context> = {
  user?: Resolver<Maybe<User>, ParentType, ContextType, RequireFields<QueryUserArgs, 'id'>>;
  users?: Resolver<UserConnection, ParentType, ContextType, Partial<QueryUsersArgs>>;
};
```

#### Write Failing Tests

```typescript
// tests/unit/resolvers/user.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { createMockContext } from '../../helpers/mockContext';
import { userResolvers } from '../../../src/schema/resolvers/user';
import type { User } from '../../../generated/graphql';

describe('User Resolvers', () => {
  describe('Query.user', () => {
    it('should return user by ID', async () => {
      // Arrange
      const mockContext = createMockContext();
      const mockUser = {
        id: '1',
        email: 'test@example.com',
        name: 'Test User',
        createdAt: new Date('2024-01-01'),
      };

      mockContext.services.userService.getUserById = vi.fn()
        .mockResolvedValue(mockUser);

      // Act
      const result = await userResolvers.Query.user(
        {},
        { id: '1' },
        mockContext,
        {} as any
      );

      // Assert
      expect(result).toEqual(mockUser);
      expect(mockContext.services.userService.getUserById).toHaveBeenCalledWith('1');
    });

    it('should return null for non-existent user', async () => {
      // Arrange
      const mockContext = createMockContext();
      mockContext.services.userService.getUserById = vi.fn()
        .mockResolvedValue(null);

      // Act
      const result = await userResolvers.Query.user(
        {},
        { id: '999' },
        mockContext,
        {} as any
      );

      // Assert
      expect(result).toBeNull();
    });

    it('should throw error when database fails', async () => {
      // Arrange
      const mockContext = createMockContext();
      mockContext.services.userService.getUserById = vi.fn()
        .mockRejectedValue(new Error('Database connection failed'));

      // Act & Assert
      await expect(
        userResolvers.Query.user({}, { id: '1' }, mockContext, {} as any)
      ).rejects.toThrow('Database connection failed');
    });
  });

  describe('Mutation.createUser', () => {
    it('should create user with valid input', async () => {
      // Arrange
      const mockContext = createMockContext();
      const input = {
        email: 'new@example.com',
        name: 'New User',
      };
      const mockCreatedUser = {
        id: '2',
        ...input,
        createdAt: new Date(),
      };

      mockContext.services.userService.createUser = vi.fn()
        .mockResolvedValue(mockCreatedUser);

      // Act
      const result = await userResolvers.Mutation.createUser(
        {},
        { input },
        mockContext,
        {} as any
      );

      // Assert
      expect(result.user).toEqual(mockCreatedUser);
      expect(mockContext.services.userService.createUser).toHaveBeenCalledWith(input);
    });

    it('should throw error for duplicate email', async () => {
      // Arrange
      const mockContext = createMockContext();
      const input = {
        email: 'existing@example.com',
        name: 'Test User',
      };

      mockContext.services.userService.createUser = vi.fn()
        .mockRejectedValue(new Error('User with this email already exists'));

      // Act & Assert
      await expect(
        userResolvers.Mutation.createUser({}, { input }, mockContext, {} as any)
      ).rejects.toThrow('User with this email already exists');
    });
  });

  describe('User.posts (field resolver)', () => {
    it('should return paginated posts for user', async () => {
      // Arrange
      const mockContext = createMockContext();
      const parent: User = {
        id: '1',
        email: 'test@example.com',
        name: 'Test User',
        createdAt: new Date(),
      } as User;

      const mockPosts = {
        edges: [
          { cursor: 'cursor1', node: { id: 'post1', title: 'Post 1' } },
        ],
        pageInfo: {
          hasNextPage: false,
          hasPreviousPage: false,
          startCursor: 'cursor1',
          endCursor: 'cursor1',
        },
        totalCount: 1,
      };

      mockContext.services.postService.getPostsByUserId = vi.fn()
        .mockResolvedValue(mockPosts);

      // Act
      const result = await userResolvers.User.posts(
        parent,
        { first: 10 },
        mockContext,
        {} as any
      );

      // Assert
      expect(result).toEqual(mockPosts);
      expect(mockContext.services.postService.getPostsByUserId)
        .toHaveBeenCalledWith('1', { first: 10, after: undefined });
    });
  });
});
```

**Run tests to verify they fail:**

```bash
just test-unit
# Should see: FAILED (expected, since resolvers don't exist yet)
```

### Step 2: GREEN - Implement Resolvers

```typescript
// src/schema/resolvers/user.ts
import type { Resolvers } from '../../generated/graphql';

export const userResolvers: Resolvers = {
  Query: {
    user: async (_parent, { id }, context) => {
      return context.services.userService.getUserById(id);
    },

    users: async (_parent, { first = 10, after }, context) => {
      return context.services.userService.getUsers({ first, after });
    },
  },

  Mutation: {
    createUser: async (_parent, { input }, context) => {
      const user = await context.services.userService.createUser(input);

      return {
        user,
        userEdge: {
          cursor: Buffer.from(user.id).toString('base64'),
          node: user,
        },
      };
    },
  },

  User: {
    // Field resolver for User.posts
    posts: async (parent, { first = 10, after }, context) => {
      return context.services.postService.getPostsByUserId(parent.id, {
        first,
        after,
      });
    },
  },
};
```

**Service Layer Implementation:**

```typescript
// src/services/userService.ts
import type { User, CreateUserInput } from '../generated/graphql';
import type { Database } from '../datasources/database';

export class UserService {
  constructor(private db: Database) {}

  async getUserById(id: string): Promise<User | null> {
    const row = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );

    return row ? this.mapRowToUser(row) : null;
  }

  async getUsers({ first, after }: { first: number; after?: string }) {
    const cursor = after ? this.decodeCursor(after) : null;

    const query = cursor
      ? 'SELECT * FROM users WHERE id > $1 ORDER BY id LIMIT $2'
      : 'SELECT * FROM users ORDER BY id LIMIT $1';

    const params = cursor ? [cursor, first + 1] : [first + 1];
    const rows = await this.db.queryAll(query, params);

    const hasNextPage = rows.length > first;
    const edges = rows.slice(0, first).map((row) => ({
      cursor: this.encodeCursor(row.id),
      node: this.mapRowToUser(row),
    }));

    return {
      edges,
      pageInfo: {
        hasNextPage,
        hasPreviousPage: !!after,
        startCursor: edges[0]?.cursor ?? null,
        endCursor: edges[edges.length - 1]?.cursor ?? null,
      },
      totalCount: await this.getTotalCount(),
    };
  }

  async createUser(input: CreateUserInput): Promise<User> {
    // Check for duplicate email
    const existing = await this.db.query(
      'SELECT id FROM users WHERE email = $1',
      [input.email]
    );

    if (existing) {
      throw new Error('User with this email already exists');
    }

    const row = await this.db.query(
      'INSERT INTO users (email, name, created_at) VALUES ($1, $2, NOW()) RETURNING *',
      [input.email, input.name]
    );

    return this.mapRowToUser(row);
  }

  private mapRowToUser(row: any): User {
    return {
      id: row.id,
      email: row.email,
      name: row.name,
      createdAt: row.created_at,
    };
  }

  private encodeCursor(id: string): string {
    return Buffer.from(id).toString('base64');
  }

  private decodeCursor(cursor: string): string {
    return Buffer.from(cursor, 'base64').toString('utf-8');
  }

  private async getTotalCount(): Promise<number> {
    const result = await this.db.query('SELECT COUNT(*) as count FROM users');
    return parseInt(result.count, 10);
  }
}
```

**Custom Scalar Resolvers:**

```typescript
// src/schema/resolvers/scalars.ts
import { GraphQLScalarType, Kind } from 'graphql';

export const DateTimeScalar = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601 date-time string',

  serialize(value: unknown): string {
    if (value instanceof Date) {
      return value.toISOString();
    }
    throw new Error('DateTime must be a Date instance');
  },

  parseValue(value: unknown): Date {
    if (typeof value === 'string') {
      return new Date(value);
    }
    throw new Error('DateTime must be a string');
  },

  parseLiteral(ast): Date {
    if (ast.kind === Kind.STRING) {
      return new Date(ast.value);
    }
    throw new Error('DateTime must be a string');
  },
});
```

**Combine Resolvers:**

```typescript
// src/schema/resolvers/index.ts
import { userResolvers } from './user';
import { postResolvers } from './post';
import { DateTimeScalar } from './scalars';
import type { Resolvers } from '../../generated/graphql';

export const resolvers: Resolvers = {
  Query: {
    ...userResolvers.Query,
    ...postResolvers.Query,
  },
  Mutation: {
    ...userResolvers.Mutation,
    ...postResolvers.Mutation,
  },
  User: userResolvers.User,
  Post: postResolvers.Post,
  DateTime: DateTimeScalar,
};
```

**Run tests to verify they pass:**

```bash
just test-unit
# Should see: All tests passing ✅
```

### Step 3: REFACTOR - Clean & Format

```bash
# Generate types (if schema changed)
just codegen

# Type check
just typecheck

# Format code
just fmt

# Lint and fix
just lint-fix

# Verify tests still pass
just test-unit
```

**Update todo:**

```typescript
// Mark current task as completed
// Mark next task as in_progress
// Update your chosen tracker (TodoWrite, Linear, checklist)
```

### Step 4: COMMIT - Save Progress

```bash
# Run full test suite with coverage
just test-cov-gate

# Stage changes
git add -A

# Commit with descriptive message
git commit -m "feat: Implement User queries and mutations

Implemented User schema with comprehensive resolver tests:
- Query.user resolver with null handling
- Query.users resolver with cursor-based pagination
- Mutation.createUser with duplicate email validation
- User.posts field resolver with pagination
- Custom DateTime scalar

Testing:
- 8 new resolver tests covering happy paths and edge cases
- All 15 tests passing
- 85% code coverage

Schema includes full documentation with descriptions."

# Push to remote
git push
```

---

## Testing Guidelines

### Test Structure for Resolvers

```typescript
// tests/unit/resolvers/query.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { createTestServer } from '../../helpers/testServer';
import { graphql } from '../../helpers/graphql'; // Generated typed document nodes

describe('User Queries', () => {
  describe('user query', () => {
    it('should return user by ID with all fields', async () => {
      // Arrange
      const server = createTestServer({
        mockData: {
          users: [
            { id: '1', email: 'test@example.com', name: 'Test User' },
          ],
        },
      });

      const query = graphql(`
        query GetUser($id: ID!) {
          user(id: $id) {
            id
            email
            name
            createdAt
          }
        }
      `);

      // Act
      const result = await server.executeOperation({
        query,
        variables: { id: '1' },
      });

      // Assert
      expect(result.body.kind).toBe('single');
      if (result.body.kind === 'single') {
        expect(result.body.singleResult.errors).toBeUndefined();
        expect(result.body.singleResult.data?.user).toMatchObject({
          id: '1',
          email: 'test@example.com',
          name: 'Test User',
        });
      }
    });
  });
});
```

### Integration Tests

```typescript
// tests/integration/user-flow.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createTestServer } from '../helpers/testServer';
import { graphql } from '../helpers/graphql';

describe('User Flow Integration', () => {
  let server: ReturnType<typeof createTestServer>;

  beforeAll(async () => {
    server = createTestServer({ useRealDatabase: true });
    await server.start();
  });

  afterAll(async () => {
    await server.stop();
  });

  it('should create user and fetch it', async () => {
    // Step 1: Create user
    const createMutation = graphql(`
      mutation CreateUser($input: CreateUserInput!) {
        createUser(input: $input) {
          user {
            id
            email
            name
          }
        }
      }
    `);

    const createResult = await server.executeOperation({
      query: createMutation,
      variables: {
        input: { email: 'integration@example.com', name: 'Integration User' },
      },
    });

    expect(createResult.body.kind).toBe('single');
    let userId: string | undefined;

    if (createResult.body.kind === 'single') {
      expect(createResult.body.singleResult.errors).toBeUndefined();
      userId = createResult.body.singleResult.data?.createUser.user.id;
      expect(userId).toBeDefined();
    }

    // Step 2: Fetch created user
    const getQuery = graphql(`
      query GetUser($id: ID!) {
        user(id: $id) {
          id
          email
          name
        }
      }
    `);

    const getResult = await server.executeOperation({
      query: getQuery,
      variables: { id: userId! },
    });

    if (getResult.body.kind === 'single') {
      expect(getResult.body.singleResult.data?.user).toMatchObject({
        id: userId,
        email: 'integration@example.com',
        name: 'Integration User',
      });
    }
  });
});
```

### Test Coverage Checklist

For each resolver, test:

- ✅ **Happy path** - Normal query/mutation works correctly
- ✅ **Null handling** - Returns null when entity not found (queries)
- ✅ **Error cases** - Invalid input, business rule violations
- ✅ **Authorization** - User has permission to perform action
- ✅ **Field resolvers** - Nested fields resolve correctly
- ✅ **Pagination** - Cursor-based pagination works (hasNextPage, cursors)
- ✅ **Input validation** - Schema validation catches invalid inputs

---

## Configuration Files

### TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Node",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "types": ["node", "vitest/globals"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### GraphQL Code Generator Configuration

```typescript
// codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  // Schema sources (can be local files or remote URLs)
  schema: './src/schema/typeDefs/**/*.graphql',

  // Where to find GraphQL operations (for client-side codegen)
  documents: ['tests/**/*.ts'],

  generates: {
    // Generate TypeScript types for schema and resolvers
    './generated/graphql.ts': {
      plugins: [
        'typescript',
        'typescript-resolvers',
      ],
      config: {
        // Use custom context type
        contextType: '../src/context#Context',

        // Map custom scalars to TypeScript types
        scalars: {
          DateTime: 'Date',
        },

        // Make resolvers type-safe
        useIndexSignature: false,

        // Add useful types
        maybeValue: 'T | null | undefined',
      },
    },

    // Generate typed document nodes for tests
    './tests/helpers/': {
      preset: 'client',
      config: {
        documentMode: 'string',
      },
    },
  },

  // Watch for schema changes
  watch: false,

  // Fail on invalid documents
  ignoreNoDocuments: false,
};

export default config;
```

### Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'dist/',
        'generated/',
        'tests/',
        '**/*.config.ts',
        '**/*.d.ts',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
    setupFiles: ['./tests/setup.ts'],
  },
});
```

### Prettier Configuration

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### ESLint Configuration

```javascript
// eslint.config.js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    rules: {
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/explicit-function-return-type': 'off',
    },
  }
);
```

---

## Schema Design Best Practices

### Naming Conventions

```graphql
# Types: PascalCase
type User { }
type PostConnection { }

# Fields: camelCase
type User {
  firstName: String!
  createdAt: DateTime!
}

# Arguments: camelCase
type Query {
  users(first: Int, after: String): UserConnection!
}

# Enums: SCREAMING_SNAKE_CASE for values
enum UserRole {
  ADMIN
  MODERATOR
  USER
}

# Input types: PascalCase with "Input" suffix
input CreateUserInput {
  email: String!
  name: String!
}

# Payload types: PascalCase with "Payload" suffix
type CreateUserPayload {
  user: User!
}
```

### Documentation

```graphql
"""
Multi-line description for a type.
Supports **markdown** formatting.
"""
type User {
  """Single-line description for a field."""
  id: ID!

  """
  Field with detailed explanation.

  This field returns the user's email address.
  It is unique across all users.

  @example "user@example.com"
  """
  email: String!
}
```

### Pagination Patterns

```graphql
# Cursor-based (recommended for large datasets)
type Query {
  users(
    first: Int = 10
    after: String
  ): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  cursor: String!
  node: User!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Offset-based (simpler, but less scalable)
type Query {
  users(
    limit: Int = 10
    offset: Int = 0
  ): UserPage!
}

type UserPage {
  items: [User!]!
  totalCount: Int!
  hasMore: Boolean!
}
```

### Error Handling

```graphql
# Option 1: Use unions for explicit error types
type Mutation {
  createUser(input: CreateUserInput!): CreateUserResult!
}

union CreateUserResult = CreateUserSuccess | EmailTakenError | ValidationError

type CreateUserSuccess {
  user: User!
}

type EmailTakenError {
  message: String!
  email: String!
}

type ValidationError {
  message: String!
  field: String!
}

# Option 2: Use GraphQL errors (simpler, less type-safe)
type Mutation {
  createUser(input: CreateUserInput!): User!
}

# Resolver throws GraphQLError which appears in response.errors
```

### Nullable vs Non-Nullable

```graphql
# Non-nullable (!) - Field is guaranteed to return a value
type User {
  id: ID!          # Always present
  email: String!   # Always present
}

# Nullable - Field may return null
type Query {
  user(id: ID!): User  # Returns null if not found
}

# List non-null, items non-null
type User {
  posts: [Post!]!  # Always returns array, never null, items never null
}

# List nullable, items non-null
type User {
  posts: [Post!]   # May return null, but items never null
}

# List non-null, items nullable
type User {
  posts: [Post]!   # Always returns array, items may be null
}
```

### Schema Evolution (No Versioning)

```graphql
# ✅ GOOD: Add new fields (non-breaking)
type User {
  id: ID!
  email: String!
  name: String!
  avatar: String  # New optional field
}

# ✅ GOOD: Add new optional arguments (non-breaking)
type Query {
  users(
    first: Int
    after: String
    role: UserRole  # New optional filter
  ): UserConnection!
}

# ✅ GOOD: Deprecate instead of removing
type User {
  id: ID!
  email: String!
  name: String!

  """
  @deprecated Use `firstName` and `lastName` instead.
  """
  fullName: String @deprecated(reason: "Use firstName and lastName instead")

  firstName: String
  lastName: String
}

# ❌ BAD: Remove fields (breaking change)
type User {
  id: ID!
  # email: String!  # REMOVED - breaks existing queries
  name: String!
}

# ❌ BAD: Make nullable field non-nullable (breaking change)
type User {
  id: ID!
  email: String!  # Was String, now String! - breaks queries expecting null
}

# ❌ BAD: Change field type (breaking change)
type User {
  id: ID!
  age: String!  # Was Int!, now String! - breaks existing queries
}
```

---

## Context & Data Loaders

### Context Type

```typescript
// src/context.ts
import type { Request } from 'express';
import { UserService } from './services/userService';
import { PostService } from './services/postService';
import { Database } from './datasources/database';
import DataLoader from 'dataloader';
import type { User, Post } from './generated/graphql';

export interface Context {
  // Request information
  req: Request;

  // Current user (if authenticated)
  currentUser: User | null;

  // Services (business logic)
  services: {
    userService: UserService;
    postService: PostService;
  };

  // DataLoaders (batching and caching)
  loaders: {
    userLoader: DataLoader<string, User | null>;
    postLoader: DataLoader<string, Post | null>;
  };

  // Database client
  db: Database;
}

export async function createContext({ req }: { req: Request }): Promise<Context> {
  const db = new Database();

  // Services
  const userService = new UserService(db);
  const postService = new PostService(db);

  // DataLoaders for efficient batching
  const userLoader = new DataLoader<string, User | null>(async (ids) => {
    const users = await userService.getUsersByIds([...ids]);
    return ids.map((id) => users.find((u) => u.id === id) ?? null);
  });

  const postLoader = new DataLoader<string, Post | null>(async (ids) => {
    const posts = await postService.getPostsByIds([...ids]);
    return ids.map((id) => posts.find((p) => p.id === id) ?? null);
  });

  // Authentication (extract from JWT, session, etc.)
  const currentUser = await authenticateUser(req);

  return {
    req,
    currentUser,
    services: { userService, postService },
    loaders: { userLoader, postLoader },
    db,
  };
}
```

### Using DataLoaders in Resolvers

```typescript
// src/schema/resolvers/post.ts
import type { Resolvers } from '../../generated/graphql';

export const postResolvers: Resolvers = {
  Post: {
    // Without DataLoader (N+1 problem)
    author: async (parent, _args, context) => {
      // This queries database separately for each post!
      return context.services.userService.getUserById(parent.authorId);
    },

    // With DataLoader (batched queries)
    author: async (parent, _args, context) => {
      // DataLoader batches all getUserById calls in single query
      return context.loaders.userLoader.load(parent.authorId);
    },
  },
};
```

---

## Authorization

### Field-Level Authorization

```typescript
// src/middleware/auth.ts
import { GraphQLError } from 'graphql';
import type { Context } from '../context';

export function requireAuth(context: Context): void {
  if (!context.currentUser) {
    throw new GraphQLError('You must be logged in', {
      extensions: { code: 'UNAUTHENTICATED' },
    });
  }
}

export function requireRole(context: Context, role: string): void {
  requireAuth(context);

  if (context.currentUser!.role !== role) {
    throw new GraphQLError('You do not have permission', {
      extensions: { code: 'FORBIDDEN' },
    });
  }
}

// Use in resolvers
export const userResolvers: Resolvers = {
  Query: {
    me: async (_parent, _args, context) => {
      requireAuth(context);
      return context.currentUser!;
    },

    allUsers: async (_parent, _args, context) => {
      requireRole(context, 'ADMIN');
      return context.services.userService.getAllUsers();
    },
  },

  Mutation: {
    deleteUser: async (_parent, { id }, context) => {
      requireAuth(context);

      // User can only delete their own account, unless admin
      if (context.currentUser!.id !== id && context.currentUser!.role !== 'ADMIN') {
        throw new GraphQLError('You can only delete your own account', {
          extensions: { code: 'FORBIDDEN' },
        });
      }

      return context.services.userService.deleteUser(id);
    },
  },
};
```

---

## Server Setup

```typescript
// src/server.ts
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import { readFileSync } from 'fs';
import { join } from 'path';
import { resolvers } from './schema/resolvers';
import { createContext } from './context';

// Load all schema files
const userTypeDefs = readFileSync(
  join(__dirname, 'schema/typeDefs/user.graphql'),
  'utf-8'
);
const postTypeDefs = readFileSync(
  join(__dirname, 'schema/typeDefs/post.graphql'),
  'utf-8'
);
const commonTypeDefs = readFileSync(
  join(__dirname, 'schema/typeDefs/common.graphql'),
  'utf-8'
);

const typeDefs = [userTypeDefs, postTypeDefs, commonTypeDefs];

// Create Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,

  // Include stack traces in development
  includeStacktraceInErrorResponses: process.env.NODE_ENV === 'development',

  // Introspection (disable in production)
  introspection: process.env.NODE_ENV !== 'production',

  // Format errors
  formatError: (formattedError, error) => {
    // Don't expose internal errors to client
    if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
      console.error('Internal error:', error);
      return {
        message: 'An unexpected error occurred',
        extensions: { code: 'INTERNAL_SERVER_ERROR' },
      };
    }

    return formattedError;
  },
});

// Start server
export async function startServer() {
  const { url } = await startStandaloneServer(server, {
    context: createContext,
    listen: { port: parseInt(process.env.PORT || '4000', 10) },
  });

  console.log(`🚀 Server ready at ${url}`);
  console.log(`📊 GraphQL Playground: ${url}`);
}
```

```typescript
// src/index.ts
import { startServer } from './server';

startServer().catch((error) => {
  console.error('Failed to start server:', error);
  process.exit(1);
});
```

---

## Full Session Checklist

**Before Starting:**

- [ ] Project initialized with dependencies installed
- [ ] TypeScript and tools configured
- [ ] Justfile automation set up
- [ ] Todo list created with all tasks

**For Each Feature:**

- [ ] Define GraphQL schema (types, queries, mutations)
- [ ] Add documentation to schema
- [ ] Generate TypeScript types (`just codegen`)
- [ ] Write failing resolver tests (RED)
- [ ] Verify tests fail
- [ ] Implement resolvers and service layer (GREEN)
- [ ] Verify tests pass
- [ ] Refactor and optimize (DataLoaders if needed)
- [ ] Format and lint (REFACTOR)
- [ ] Type check passes
- [ ] Update todo (mark complete, next in_progress)
- [ ] Commit with descriptive message (COMMIT)

**After Each Phase:**

- [ ] Run full test suite with coverage: `just test-cov-gate`
- [ ] All tests passing
- [ ] Code formatted and linted: `just fmt && just lint`
- [ ] Type checks clean: `just typecheck`
- [ ] Update documentation
- [ ] Commit documentation update
- [ ] Push all commits to remote

**Session Complete:**

- [ ] All todos completed
- [ ] Full test suite passing
- [ ] Schema documented
- [ ] All commits pushed
- [ ] Clean working directory (`git status`)

---

## Anti-Patterns to Avoid

❌ **DON'T:**

- Define schema without documentation
- Write resolvers before tests
- Put business logic in resolvers
- Return different types from same field based on conditions (use unions)
- Use GraphQL enums for frequently changing values
- Break the schema (remove fields, change types)
- Expose internal error details to clients
- Skip DataLoaders for N+1 query problems
- Make everything non-nullable without considering failure cases
- Use REST-style paths in field names (`getUserById` → use `user(id:)`)

✅ **DO:**

- Document every type, field, and argument
- Write resolver tests first (TDD)
- Keep resolvers thin, delegate to service layer
- Use unions for fields that can return different types
- Use strings for frequently changing values, enums for fixed sets
- Evolve schema additively (add fields, deprecate old ones)
- Log errors, return generic messages to clients
- Use DataLoaders for fetching related entities
- Use nullable fields when failure is possible
- Use GraphQL-idiomatic naming (queries as nouns, mutations as verbs)

---

## Tools & Commands Reference

### Essential Commands

```bash
# Development
just dev                    # Start development server with watch
just codegen               # Generate TypeScript types from schema
just codegen-watch         # Watch mode for codegen

# Testing
just test                  # Run all tests
just test-watch            # Run tests in watch mode
just test-unit             # Run only unit tests
just test-integration      # Run only integration tests
just test-cov              # Run tests with coverage report
just test-cov-gate         # Run tests with coverage threshold

# Code Quality
just fmt                   # Format code
just fmt-check             # Check formatting without modifying
just lint                  # Run linter
just lint-fix              # Fix lint issues automatically
just typecheck             # Run TypeScript compiler

# CI/CD
just check                 # Run all checks (fmt, lint, typecheck, test)
just build                 # Build for production

# Git
git status
git add -A
git commit -m "type: message"
git push
```

### Test Helpers

```typescript
// tests/helpers/testServer.ts
import { ApolloServer } from '@apollo/server';
import { typeDefs } from '../../src/schema/typeDefs';
import { resolvers } from '../../src/schema/resolvers';
import type { Context } from '../../src/context';

export function createTestServer(options: {
  mockData?: any;
  useRealDatabase?: boolean;
} = {}) {
  const server = new ApolloServer<Context>({
    typeDefs,
    resolvers,
  });

  return {
    async executeOperation(operation: any) {
      return server.executeOperation(operation, {
        contextValue: createMockContext(options),
      });
    },
    async start() {
      await server.start();
    },
    async stop() {
      await server.stop();
    },
  };
}

export function createMockContext(options: any = {}): Context {
  return {
    req: {} as any,
    currentUser: options.currentUser ?? null,
    services: {
      userService: createMockUserService(options.mockData),
      postService: createMockPostService(options.mockData),
    },
    loaders: {} as any,
    db: {} as any,
  };
}
```

---

## Success Metrics

After following this workflow, you should have:

✅ **Type Safety** - Schema types flow through entire codebase
✅ **Self-Documenting** - Schema serves as API documentation
✅ **High Confidence** - Comprehensive resolver tests
✅ **Performance** - DataLoaders prevent N+1 queries
✅ **Evolvable** - Schema evolves without breaking changes
✅ **Maintainability** - Thin resolvers, business logic in services
✅ **Visibility** - Todo list tracks feature progress

---

## Troubleshooting

### Type Generation Fails

```bash
# Ensure schema files are valid GraphQL
npx graphql-inspector validate 'src/schema/typeDefs/**/*.graphql'

# Check for syntax errors
just codegen --verbose

# Clear generated files and regenerate
rm -rf generated/
just codegen
```

### Tests Can't Find Generated Types

```bash
# Ensure types are generated before running tests
just codegen
just test

# Add to package.json scripts
{
  "scripts": {
    "pretest": "graphql-codegen"
  }
}
```

### Resolver Type Errors

```typescript
// Problem: Type mismatch in resolver
const resolvers: Resolvers = {
  Query: {
    // ❌ Error: Type 'Promise<UserModel>' is not assignable to type 'User'
    user: async (_parent, { id }, context) => {
      return context.db.getUserById(id); // Returns UserModel from DB
    },
  },
};

// Solution: Map database model to GraphQL type
const resolvers: Resolvers = {
  Query: {
    // ✅ Correct: Map UserModel to User type
    user: async (_parent, { id }, context) => {
      const userModel = await context.db.getUserById(id);
      return userModel ? mapUserModelToUser(userModel) : null;
    },
  },
};
```

### N+1 Query Problems

```typescript
// Problem: N+1 queries in field resolver
const resolvers: Resolvers = {
  Post: {
    // ❌ Called once per post, causing N database queries
    author: async (parent, _args, context) => {
      return context.services.userService.getUserById(parent.authorId);
    },
  },
};

// Solution: Use DataLoader
const resolvers: Resolvers = {
  Post: {
    // ✅ DataLoader batches requests into single query
    author: async (parent, _args, context) => {
      return context.loaders.userLoader.load(parent.authorId);
    },
  },
};
```

---

## Related Workflows

- **[TypeScript Development Workflow](typescript-development-workflow.md)** - Core TypeScript practices
- **[TypeScript Service Workflow](typescript-service-workflow.md)** - Production service patterns
- **[PostgreSQL Development Workflow](postgresql-development-workflow.md)** - Database integration
- **[Observability Workflow](observability-workflow.md)** - Logging, metrics, tracing

---

**Remember:** GraphQL's strength is its type system and self-documenting nature. Invest time in schema design upfront, generate types for everything, and let the type system guide your implementation.

**When in doubt, write a test first and let the types guide you!**
