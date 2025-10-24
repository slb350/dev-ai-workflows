# TypeScript Development Workflow - TDD Best Practices

**Version:** 1.0
**Last Updated:** 2025-10-17
**Purpose:** Stellar workflow for TypeScript development with comprehensive test coverage, type safety, and modern JavaScript ecosystem integration

---

## Overview

This workflow prioritizes **type safety**, **maintainability**, and **developer experience** through rigorous Test-Driven Development (TDD). Every feature is built with tests first, leveraging TypeScript's static typing to catch errors at compile-time while maintaining excellent runtime performance.

---

## Core Principles

1. **Tests First, Always** - No production code without failing tests
2. **Type Safety First** - Let TypeScript catch errors before runtime
3. **One Task at a Time** - Complete each section fully before moving on
4. **Immediate Feedback** - Run tests and type checking after every change
5. **Modern Practices** - Use modern TypeScript features and patterns
6. **Clear History** - Commit after each completed section
7. **Track Everything** - Keep a visible task tracker (TodoWrite, Linear, checklist, etc.)
8. **ESM Native** - Embrace ES modules and modern tooling

---

## The Workflow (Red-Green-Refactor-Commit)

### Phase 1: Planning & Setup

```bash
# 1. Initialize project
npm init -y
npm pkg set type=module
npm install typescript@5.4.5 @types/node@20.12.7 --save-dev --save-exact

# 2. Initialize TypeScript configuration (ESM + modern defaults)
npx tsc --init --strict --module esnext --moduleResolution node16 --target es2022 --outDir dist --rootDir src --resolveJsonModule --noEmitOnError --esModuleInterop

# 3. Set up project structure
mkdir -p src tests
mkdir -p src/{components,services,utils,types}
mkdir -p tests/{unit,integration}

# 4. Install development dependencies
npm install --save-dev --save-exact \
  jest@29.7.0 \
  @types/jest@29.5.12 \
  ts-jest@29.1.2 \
  eslint@8.57.0 \
  @typescript-eslint/eslint-plugin@7.8.0 \
  @typescript-eslint/parser@7.8.0 \
  prettier@3.2.5 \
  prettier-plugin-organize-imports@3.4.2 \
  husky@9.0.11 \
  lint-staged@15.2.3 \
  nodemon@3.1.0 \
  concurrently@8.2.2

# (Using pnpm? mirror these commands with `pnpm add --save-dev --save-exact ...`. Enable Corepack if needed: `corepack enable`.)

# 5. Verify environment (after scripts configured below)
npx tsc --version
npx jest --version
npm pkg get type  # Should output "module"

# After defining npm scripts (see below), run `npm run build` once to ensure compilation succeeds.
```

**Create Todo List:**

```typescript
// Track todos in your preferred tool (TodoWrite, Linear, checklist, etc.):
// - All major modules/features to implement
// - Sub-tasks for complex features
// - Format, test, commit, and docs tasks at end
```

### Phase 2: TDD Cycle (For Each Feature/Module)

#### Step 1: RED - Write Failing Tests

```typescript
// Create test file: tests/unit/Feature.test.ts
import { Feature, FeatureError } from '../../src/services/Feature';
import { describe, it, expect, beforeEach } from '@jest/globals';

describe('Feature', () => {
  let feature: Feature;

  beforeEach(() => {
    feature = new Feature('test-config');
  });

  describe('process', () => {
    it('should process valid input successfully', () => {
      // Arrange
      const input = 'test-input';

      // Act
      const result = feature.process(input);

      // Assert
      expect(result.isOk()).toBe(true);
      expect(result.unwrap()).toBe('PROCESSED-test-input');
      expect(feature.getState()).toBe('processed-PROCESSED-test-input');
    });

    it('should return error for empty input', () => {
      // Arrange
      const input = '';

      // Act
      const result = feature.process(input);

      // Assert
      expect(result.isErr()).toBe(true);
      expect(result.unwrapErr()).toBeInstanceOf(FeatureError);
      expect(result.unwrapErr().message).toContain('Input cannot be empty');
    });

    it('should handle special characters in input', () => {
      // Arrange
      const input = 'hello-world!@#$%';

      // Act
      const result = feature.process(input);

      // Assert
      expect(result.isOk()).toBe(true);
      expect(result.unwrap()).toContain('PROCESSED');
    });
  });

  describe('concurrent processing', () => {
    it('should handle concurrent operations safely', async () => {
      // Arrange
      const promises = Array.from({ length: 10 }, (_, i) =>
        feature.processAsync(`input-${i}`)
      );

      // Act
      const results = await Promise.all(promises);

      // Assert
      results.forEach((result, index) => {
        expect(result.isOk()).toBe(true);
        expect(result.unwrap()).toContain(`PROCESSED-input-${index}`);
      });
    });
  });

  describe('property-based tests', () => {
    it('should preserve input length in output', () => {
      // Test multiple inputs
      const testInputs = ['a', 'ab', 'abc', 'abcd', 'abcde'];

      testInputs.forEach(input => {
        const result = feature.process(input);
        expect(result.isOk()).toBe(true);
        const output = result.unwrap();
        expect(output.length).toBeGreaterThan(input.length);
      });
    });

    it('should handle Unicode characters correctly', () => {
      const unicodeInputs = ['café', 'naïve', '🚀 rocket', '🎉 celebration'];

      unicodeInputs.forEach(input => {
        const result = feature.process(input);
        expect(result.isOk()).toBe(true);
        expect(result.unwrap()).toContain(input);
      });
    });
  });
});
```

**Run tests to verify they fail:**

```bash
npm test -- Feature.test.ts
# Should see: FAIL (expected, since code doesn't exist yet)
```

#### Step 2: GREEN - Implement Minimum Code

```typescript
// Create implementation: src/services/Feature.ts
export class FeatureError extends Error {
  public readonly code: string;

  constructor(message: string, code: string = 'FEATURE_ERROR') {
    super(message);
    this.name = 'FeatureError';
    this.code = code;
  }
}

// Result type for better error handling
export type Result<T, E = Error> = {
  isOk(): this is { isOk: true; isErr: false; unwrap(): T };
  isErr(): this is { isOk: false; isErr: true; unwrapErr(): E };
  unwrap(): T;
  unwrapErr(): E;
};

export class Ok<T> implements Result<T> {
  constructor(private readonly value: T) {}

  isOk(): this is { isOk: true; isErr: false; unwrap(): T } {
    return true;
  }

  isErr(): this is { isOk: false; isErr: true; unwrapErr(): never } {
    return false;
  }

  unwrap(): T {
    return this.value;
  }

  unwrapErr(): never {
    throw new Error('Cannot unwrap error from Ok result');
  }
}

export class Err<E> implements Result<never, E> {
  constructor(private readonly error: E) {}

  isOk(): this is { isOk: true; isErr: false; unwrap(): never } {
    return false;
  }

  isErr(): this is { isOk: false; isErr: true; unwrapErr(): E } {
    return true;
  }

  unwrap(): never {
    throw this.error;
  }

  unwrapErr(): E {
    return this.error;
  }
}

export const ok = <T>(value: T): Result<T> => new Ok(value);
export const err = <E>(error: E): Result<never, E> => new Err(error);

export class Feature {
  private state: string;
  private readonly config: string;

  constructor(config: string) {
    this.config = config;
    this.state = 'initial';
  }

  public process(input: string): Result<string, FeatureError> {
    if (input.length === 0) {
      return err(new FeatureError('Input cannot be empty', 'EMPTY_INPUT'));
    }

    const processed = `PROCESSED-${input}`;
    this.state = `processed-${processed}`;

    return ok(processed);
  }

  public async processAsync(input: string): Promise<Result<string, FeatureError>> {
    // Simulate async processing
    await new Promise(resolve => setTimeout(resolve, Math.random() * 10));

    return this.process(input);
  }

  public getState(): string {
    return this.state;
  }

  public getConfig(): string {
    return this.config;
  }
}

// Utility functions
export const createFeature = (config: string): Feature => {
  return new Feature(config);
};

// Type guards
export const isFeatureError = (error: unknown): error is FeatureError => {
  return error instanceof FeatureError;
};
```

**Run tests to verify they pass:**

```bash
npm test -- Feature.test.ts
# Should see: PASS
```

#### Step 3: REFACTOR - Clean & Format

```bash
# Format code
npm run format

# Run linter and fix issues
npm run lint:fix

# Type checking
npm run type-check

# Verify tests still pass after refactoring
npm test -- Feature.test.ts
```

**Update todo:**

```typescript
// Mark current task as completed
// Mark next task as in_progress
// Update your chosen tracker (TodoWrite, Linear, checklist, etc.)
```

#### Step 4: COMMIT - Save Progress

```bash
# Stage changes
git add -A

# Commit with descriptive message
git commit -m "feat(services): implement Feature with comprehensive test coverage

Implemented Feature class with modern error handling:
- Result type for better error handling instead of exceptions
- Async processing capabilities with Promise support
- Unicode and special character handling
- Type-safe error codes and messages

Testing:
- 12 new tests covering all functionality and edge cases
- Property-based tests for input validation
- Async/concurrent testing scenarios
- All tests passing with 100% coverage

services/Feature implementation complete."
```

---

## Testing Guidelines

### Test Structure

```typescript
// tests/unit/Service.test.ts
import { Service, ServiceConfig } from '../../src/services/Service';
import { MockLogger, MockDatabase } from '../mocks';
import { describe, it, expect, beforeEach, afterEach } from '@jest/globals';

describe('Service', () => {
  let service: Service;
  let mockLogger: MockLogger;
  let mockDatabase: MockDatabase;

  beforeEach(() => {
    mockLogger = new MockLogger();
    mockDatabase = new MockDatabase();
    const config: ServiceConfig = {
      timeout: 5000,
      retries: 3,
    };
    service = new Service(config, mockLogger, mockDatabase);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('processData', () => {
    it('should process data successfully with valid input', async () => {
      // Arrange
      const inputData = { id: '123', value: 'test data' };
      mockDatabase.get.mockResolvedValueOnce({ id: '123', processed: false });

      // Act
      const result = await service.processData(inputData);

      // Assert
      expect(result.success).toBe(true);
      expect(result.data.processed).toBe(true);
      expect(mockDatabase.save).toHaveBeenCalledWith(
        expect.objectContaining({ id: '123', processed: true })
      );
    });

    it('should handle database errors gracefully', async () => {
      // Arrange
      const inputData = { id: '456', value: 'test data' };
      mockDatabase.get.mockRejectedValueOnce(new Error('Database connection failed'));

      // Act
      const result = await service.processData(inputData);

      // Assert
      expect(result.success).toBe(false);
      expect(result.error).toBeDefined();
      expect(mockLogger.error).toHaveBeenCalledWith(
        expect.stringContaining('Database error')
      );
    });
  });

  describe('batch processing', () => {
    it('should process multiple items concurrently', async () => {
      // Arrange
      const items = Array.from({ length: 5 }, (_, i) => ({
        id: `${i}`,
        value: `data-${i}`,
      }));

      mockDatabase.get.mockResolvedValue({ processed: false });
      mockDatabase.save.mockResolvedValue(undefined);

      // Act
      const results = await service.batchProcess(items);

      // Assert
      expect(results).toHaveLength(5);
      results.forEach(result => {
        expect(result.success).toBe(true);
      });
      expect(mockDatabase.save).toHaveBeenCalledTimes(5);
    });
  });
});
```

### Test Coverage Checklist

For each module/class, test:

- ✅ **Happy path** - Normal usage works correctly
- ✅ **Error handling** - All error scenarios covered
- ✅ **Edge cases** - Empty inputs, null/undefined, boundary values
- ✅ **Type safety** - TypeScript compilation catches issues
- ✅ **Async behavior** - Promises, async/await scenarios
- ✅ **Integration** - Works with other modules
- ✅ **Performance** - Not too slow for expected usage
- ✅ **Accessibility** - If UI components, test a11y features

### Test Naming Convention

```typescript
describe('ClassName', () => {
  describe('methodName', () => {
    it('should do X when Y', () => {});
    it('should return error when condition Z', () => {});
    it('should handle edge case A', () => {});
  });
});

// Or using BDD style
describe('Feature', () => {
  it('processes valid input successfully', () => {});
  it('rejects invalid input with appropriate error', () => {});
  it('handles concurrent requests safely', () => {});
});
```

### Mock Testing

```typescript
// tests/mocks/Logger.mock.ts
import { Logger } from '../../src/types/Logger';

export class MockLogger implements Logger {
  public logs: Array<{ level: string; message: string; meta?: any }> = [];
  public error = jest.fn();
  public warn = jest.fn();
  public info = jest.fn();
  public debug = jest.fn();

  constructor() {
    this.error.mockImplementation((message: string, meta?: any) => {
      this.logs.push({ level: 'error', message, meta });
    });
    this.info.mockImplementation((message: string, meta?: any) => {
      this.logs.push({ level: 'info', message, meta });
    });
  }

  public hasLogged(level: string, message: string): boolean {
    return this.logs.some(log =>
      log.level === level && log.message.includes(message)
    );
  }

  public clear(): void {
    this.logs = [];
    jest.clearAllMocks();
  }
}

// tests/mocks/Database.mock.ts
import { Database } from '../../src/types/Database';
import { User } from '../../src/types/User';

export class MockDatabase implements Database {
  public users = new Map<string, User>();
  public get = jest.fn();
  public save = jest.fn();
  public delete = jest.fn();
  public find = jest.fn();

  constructor() {
    this.get.mockImplementation(async (id: string) => {
      return this.users.get(id);
    });

    this.save.mockImplementation(async (user: User) => {
      this.users.set(user.id, user);
    });
  }

  public addUser(user: User): void {
    this.users.set(user.id, user);
  }

  public clear(): void {
    this.users.clear();
    jest.clearAllMocks();
  }
}
```

After saving `package.json`, run:

```bash
npm run build
```

---

## Project Structure Best Practices

```text
projectname/
├── src/
│   ├── index.ts                 # Main entry point
│   ├── types/                   # Type definitions
│   │   ├── index.ts
│   │   ├── User.ts
│   │   ├── Config.ts
│   │   └── Result.ts
│   ├── services/                # Business logic
│   │   ├── index.ts
│   │   ├── UserService.ts
│   │   ├── AuthService.ts
│   │   └── ProcessingService.ts
│   ├── utils/                   # Utility functions
│   │   ├── index.ts
│   │   ├── validation.ts
│   │   ├── formatting.ts
│   │   └── helpers.ts
│   ├── components/              # UI components (if applicable)
│   │   ├── index.ts
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.test.tsx
│   │   │   └── Button.styles.ts
│   │   └── Form/
│   │       ├── Form.tsx
│   │       ├── Form.test.tsx
│   │       └── Form.styles.ts
│   ├── adapters/                # External integrations
│   │   ├── index.ts
│   │   ├── database.ts
│   │   ├── http.ts
│   │   └── logger.ts
│   └── constants/               # Constants and enums
│       ├── index.ts
│       ├── errors.ts
│       └── config.ts
├── tests/
│   ├── setup.ts                 # Jest setup
│   ├── mocks/                   # Mock implementations
│   │   ├── Logger.mock.ts
│   │   └── Database.mock.ts
│   ├── unit/                    # Unit tests
│   │   ├── services/
│   │   └── utils/
│   ├── integration/             # Integration tests
│   │   ├── api.integration.test.ts
│   │   └── database.integration.test.ts
│   └── fixtures/                # Test data
│       ├── users.json
│       └── config.json
├── docs/                        # Documentation
├── examples/                    # Example usage
├── scripts/                     # Build and utility scripts
├── .github/
│   └── workflows/
│       └── ci.yml
├── package.json
├── tsconfig.json
├── jest.config.js
├── .eslintrc.js
├── .prettierrc
├── .gitignore
└── README.md
```

### Module Organization Principles

1. **Barrel Exports** - Use `index.ts` files for clean imports
2. **Type Co-location** - Keep types close to their usage
3. **Separate Concerns** - Clear separation between business logic and infrastructure
4. **Test Proximity** - Keep tests close to source files
5. **Dependency Direction** - Dependencies should point inward

---

## TypeScript-Specific Patterns

### Type Safety Patterns

```typescript
// Discriminated unions for better type safety
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function handleApiResponse<T>(response: ApiResponse<T>): T | null {
  if (response.success) {
    return response.data;
  }
  return null;
}

// Branded types for type-safe IDs
type UserId = string & { readonly __brand: unique symbol };
type EmailAddress = string & { readonly __brand: unique symbol };

function createUserId(id: string): UserId {
  return id as UserId;
}

function createEmail(email: string): EmailAddress {
  if (!email.includes('@')) {
    throw new Error('Invalid email format');
  }
  return email as EmailAddress;
}

// Type guards for runtime type checking
function isUser(obj: unknown): obj is { id: UserId; name: string } {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    'id' in obj &&
    'name' in obj &&
    typeof obj.name === 'string'
  );
}

// Template literal types for configuration
type LogLevel = 'error' | 'warn' | 'info' | 'debug';
type LogPrefix = `[${LogLevel}]`;

type LogMessage<T extends string> = `${LogPrefix} ${T}`;

function createLogMessage<T extends string>(
  level: LogLevel,
  message: T
): LogMessage<T> {
  return `[${level}] ${message}` as LogMessage<T>;
}

// Utility types for better API design
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

type RequiredFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

type OptionalFields<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

// Example usage
interface User {
  id: UserId;
  name: string;
  email: EmailAddress;
  age: number;
  createdAt: Date;
}

type CreateUserRequest = OptionalFields<User, 'id' | 'createdAt'>;
type UpdateUserRequest = DeepPartial<Omit<User, 'id' | 'createdAt'>>;
```

### Async Patterns

```typescript
// Result-based async error handling
async function safeAsync<T, E = Error>(
  asyncFn: () => Promise<T>
): Promise<Result<T, E>> {
  try {
    const result = await asyncFn();
    return ok(result);
  } catch (error) {
    return err(error as E);
  }
}

// Usage
class UserService {
  async getUser(id: UserId): Promise<Result<User, Error>> {
    return safeAsync(async () => {
      const user = await this.database.get(id);
      if (!user) {
        throw new Error(`User with id ${id} not found`);
      }
      return user;
    });
  }
}

// Timeout wrapper
async function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<Result<T, Error>> {
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('Operation timed out')), timeoutMs);
  });

  return safeAsync(() => Promise.race([promise, timeoutPromise]));
}

// Retry mechanism
async function retry<T>(
  operation: () => Promise<T>,
  maxAttempts: number = 3,
  delayMs: number = 1000
): Promise<Result<T, Error>> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const result = await safeAsync(operation);

    if (result.isOk()) {
      return result;
    }

    lastError = result.unwrapErr();

    if (attempt < maxAttempts) {
      await new Promise(resolve => setTimeout(resolve, delayMs * attempt));
    }
  }

  return err(lastError!);
}
```

### Functional Patterns

```typescript
// Immutable update patterns
type UpdateFn<T> = (current: T) => T;

function updateObject<T extends object>(
  obj: T,
  updates: Partial<T>
): T {
  return { ...obj, ...updates };
}

function updateNested<T extends object, K extends keyof T>(
  obj: T,
  key: K,
  updateFn: UpdateFn<T[K]>
): T {
  return {
    ...obj,
    [key]: updateFn(obj[key])
  };
}

// Pipeline pattern for data transformation
type PipeFunction<T, R> = (input: T) => R;

function pipe<T, R1, R2, R3>(
  input: T,
  fn1: PipeFunction<T, R1>,
  fn2: PipeFunction<R1, R2>,
  fn3: PipeFunction<R2, R3>
): R3;
function pipe<T, R1, R2>(
  input: T,
  fn1: PipeFunction<T, R1>,
  fn2: PipeFunction<R1, R2>
): R2;
function pipe<T, R1>(
  input: T,
  fn1: PipeFunction<T, R1>
): R1;
function pipe<T, R>(input: T, ...fns: Array<PipeFunction<any, any>>): any {
  return fns.reduce((acc, fn) => fn(acc), input);
}

// Usage examples
const processUser = (user: User) =>
  pipe(
    user,
    u => updateObject(u, { name: u.name.trim() }),
    u => updateObject(u, { updatedAt: new Date() }),
    u => validateUser(u)
  );

// Option type for handling null/undefined
type Option<T> = Some<T> | None;

interface Some<T> {
  _tag: 'Some';
  value: T;
}

interface None {
  _tag: 'None';
}

const some = <T>(value: T): Option<T> => ({ _tag: 'Some', value });
const none = (): Option<never> => ({ _tag: 'None' });

const isSome = <T>(option: Option<T>): option is Some<T> =>
  option._tag === 'Some';

const isNone = <T>(option: Option<T>): option is None =>
  option._tag === 'None';

const map = <T, R>(option: Option<T>, fn: (value: T) => R): Option<R> =>
  isSome(option) ? some(fn(option.value)) : none();

const flatMap = <T, R>(option: Option<T>, fn: (value: T) => Option<R>): Option<R> =>
  isSome(option) ? fn(option.value) : none();
```

---

## Configuration Files

### TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": false,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUncheckedIndexedAccess": true,
    "allowUnusedLabels": false,
    "allowUnreachableCode": false,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  },
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "tests"
  ]
}
```

### Jest Configuration

```javascript
// jest.config.js
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest/presets/default-esm',
  testEnvironment: 'node',
  extensionsToTreatAsEsm: ['.ts'],
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: [
    '**/__tests__/**/*.ts',
    '**/?(*.)+(spec|test).ts'
  ],
  globals: {
    'ts-jest': {
      tsconfig: './tsconfig.json',
      useESM: true
    }
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: [
    'text',
    'lcov',
    'html'
  ],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@tests/(.*)$': '<rootDir>/tests/$1',
  },
  clearMocks: true,
  restoreMocks: true,
  resetMocks: true,
};

export default config;
```

### ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 2022,
    sourceType: 'module',
    project: './tsconfig.json',
  },
  plugins: ['@typescript-eslint'],
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    '@typescript-eslint/recommended-requiring-type-checking',
  ],
  root: true,
  env: {
    node: true,
    jest: true,
  },
  rules: {
    '@typescript-eslint/interface-name-prefix': 'off',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/explicit-module-boundary-types': 'warn',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-unused-vars': 'error',
    '@typescript-eslint/prefer-nullish-coalescing': 'error',
    '@typescript-eslint/prefer-optional-chain': 'error',
    '@typescript-eslint/no-floating-promises': 'error',
    '@typescript-eslint/await-thenable': 'error',
    '@typescript-eslint/no-misused-promises': 'error',
    'prefer-const': 'error',
    'no-var': 'error',
  },
  ignorePatterns: [
    'dist/',
    'node_modules/',
    '*.js',
  ],
};
```

### Prettier Configuration

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "endOfLine": "lf",
  "quoteProps": "as-needed",
  "bracketSameLine": false,
  "plugins": ["prettier-plugin-organize-imports"]
}
```

### Package.json Scripts

```json
{
  "name": "typescript-project",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "format": "prettier --write src/**/*.ts",
    "format:check": "prettier --check src/**/*.ts",
    "type-check": "tsc --noEmit",
    "pre-commit": "lint-staged",
    "prepare": "husky install",
    "dev": "concurrently \"npm run build:watch\" \"npm run test:watch\"",
    "clean": "rm -rf dist coverage",
    "start": "node dist/index.js",
    "start:dev": "nodemon --exec \"npm run build && node dist/index.js\""
  },
  "lint-staged": {
    "*.ts": [
      "prettier --write",
      "eslint --fix",
      "jest --bail --findRelatedTests"
    ]
  }
}
```

---

## Build and Development Tools

### Husky Git Hooks

```bash
# Initialize husky hooks (package installed during setup)
npm run prepare

# Add pre-commit hook
npx husky add .husky/pre-commit "npm run pre-commit"

# Add pre-push hook
npx husky add .husky/pre-push "npm run test:coverage && npm run type-check"
```

### Nodemon Configuration

```json
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.test.ts"],
  "exec": "npm run build && npm start",
  "env": {
    "NODE_ENV": "development"
  }
}
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
    - uses: actions/checkout@v4

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Type check
      run: npm run type-check

    - name: Lint
      run: npm run lint

    - name: Format check
      run: npm run format:check

    - name: Run tests
      run: npm run test:coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella

    - name: Build
      run: npm run build

  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20.x'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run security audit
      run: npm audit --audit-level=high
```

---

## Testing Advanced Scenarios

### Integration Testing

```typescript
// tests/integration/api.integration.test.ts
import { setupTestServer, teardownTestServer } from '../helpers/server';
import { createTestClient } from '../helpers/client';
import { TestDatabase } from '../helpers/database';

describe('API Integration Tests', () => {
  let server: any;
  let client: any;
  let database: TestDatabase;

  beforeAll(async () => {
    database = new TestDatabase();
    await database.setup();
    server = await setupTestServer(database);
    client = createTestClient(server.url);
  });

  afterAll(async () => {
    await client.close();
    await server.close();
    await database.cleanup();
  });

  beforeEach(async () => {
    await database.clear();
  });

  describe('POST /users', () => {
    it('should create a new user successfully', async () => {
      // Arrange
      const userData = {
        name: 'John Doe',
        email: 'john@example.com',
      };

      // Act
      const response = await client.post('/users', userData);

      // Assert
      expect(response.status).toBe(201);
      expect(response.data).toMatchObject({
        id: expect.any(String),
        name: userData.name,
        email: userData.email,
        createdAt: expect.any(String),
      });

      // Verify in database
      const dbUser = await database.getUser(response.data.id);
      expect(dbUser).toBeTruthy();
      expect(dbUser.name).toBe(userData.name);
    });

    it('should return validation error for invalid data', async () => {
      // Arrange
      const invalidData = {
        name: '',
        email: 'invalid-email',
      };

      // Act
      const response = await client.post('/users', invalidData);

      // Assert
      expect(response.status).toBe(400);
      expect(response.data.errors).toEqual(
        expect.arrayContaining([
          expect.objectContaining({
            field: 'name',
            message: expect.stringContaining('required'),
          }),
          expect.objectContaining({
            field: 'email',
            message: expect.stringContaining('valid email'),
          }),
        ])
      );
    });
  });
});
```

### Performance Testing

```typescript
// tests/performance/processor.performance.test.ts
import { Processor } from '../../src/services/Processor';
import { performance } from 'perf_hooks';

describe('Processor Performance Tests', () => {
  const processor = new Processor();

  it('should process 1000 items within acceptable time', async () => {
    // Arrange
    const items = Array.from({ length: 1000 }, (_, i) => ({
      id: `item-${i}`,
      data: `test-data-${i}`,
    }));
    const maxTimeMs = 1000; // 1 second

    // Act
    const startTime = performance.now();
    const results = await processor.batchProcess(items);
    const endTime = performance.now();
    const duration = endTime - startTime;

    // Assert
    expect(duration).toBeLessThan(maxTimeMs);
    expect(results).toHaveLength(1000);
    results.forEach(result => {
      expect(result.success).toBe(true);
    });
  });

  it('should handle memory efficiently with large datasets', async () => {
    // Arrange
    const largeDataset = Array.from({ length: 10000 }, (_, i) => ({
      id: `item-${i}`,
      data: 'x'.repeat(1000), // 1KB per item
    }));

    // Act
    const initialMemory = process.memoryUsage().heapUsed;
    await processor.batchProcess(largeDataset);
    const finalMemory = process.memoryUsage().heapUsed;

    // Assert
    const memoryIncrease = finalMemory - initialMemory;
    const memoryPerItem = memoryIncrease / largeDataset.length;

    // Should use less than 5KB per item (allowing some overhead)
    expect(memoryPerItem).toBeLessThan(5 * 1024);
  });
});
```

### Property-Based Testing

```typescript
// tests/property/transformer.property.test.ts
import { Transformer } from '../../src/utils/Transformer';
import { fc } from 'fast-check';

describe('Transformer Property Tests', () => {
  const transformer = new Transformer();

  it('should preserve string length for certain transformations', () => {
    fc.assert(
      fc.property(fc.string(), fc.string(), (input, suffix) => {
        // Test that adding suffix preserves length relationship
        const result = transformer.addSuffix(input, suffix);
        return result.length === input.length + suffix.length;
      }),
      { numRuns: 1000 }
    );
  });

  it('should maintain sorting order invariant', () => {
    fc.assert(
      fc.property(fc.array(fc.integer()), (numbers) => {
        const sorted = transformer.sortNumbers(numbers);
        const isSorted = sorted.every((val, index) =>
          index === 0 || sorted[index - 1] <= val
        );
        return isSorted && sorted.length === numbers.length;
      }),
      { numRuns: 500 }
    );
  });

  it('should be idempotent for certain operations', () => {
    fc.assert(
      fc.property(fc.string(), (input) => {
        const once = transformer.normalize(input);
        const twice = transformer.normalize(once);
        return once === twice;
      }),
      { numRuns: 1000 }
    );
  });
});
```

---

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/) with TypeScript-specific conventions:

```text
type(scope): Brief description (max 72 chars)

Detailed explanation of what and why:
- Bullet point 1
- Bullet point 2
- Bullet point 3

Testing:
- X tests added
- All Y tests passing
- Coverage: Z%

Type Safety:
- Added/updated types
- Improved type inference
- Fixed type issues

Breaking Changes:
- List any breaking changes to public API

[Optional additional context]
```

### TypeScript-Specific Commit Types

- `feat(types):` - New types or interfaces
- `feat(service):` - New service implementation
- `feat(component):` - New UI component
- `fix(types):` - Type fixes or improvements
- `fix(service):` - Service bug fixes
- `refactor(types):` - Type system refactoring
- `perf(service):` - Performance optimizations
- `deps:` - Dependency updates

### Examples

```bash
# Feature with types
git commit -m "feat(types): add branded types for type-safe IDs

Added branded types for UserId, EmailAddress, and ProductId:
- Created type-safe ID types that cannot be accidentally mixed
- Added factory functions for ID creation with validation
- Updated all services to use typed IDs instead of strings

Testing:
- 15 new tests for ID validation and type safety
- All tests passing with 100% type coverage

Type Safety:
- Compile-time guarantees for ID type correctness
- Better IDE autocompletion and error messages"

# Service refactoring
git commit -m "refactor(service): migrate UserService to Result-based error handling

Refactored UserService to use Result<T, E> pattern instead of exceptions:
- Replaced try/catch with Result types for better error handling
- Updated all public methods to return Result types
- Added comprehensive error types with error codes

Testing:
- Updated 25 tests to work with Result types
- Added error case testing for all failure scenarios
- All tests passing with improved error coverage

Breaking Changes:
- UserService methods now return Result<T, Error> instead of throwing
- Callers must handle Result.unwrap() or Result.unwrapErr()

Migration:
- Replace try/catch blocks with Result pattern
- Update error handling to use Result.isOk()/isErr()"
```

---

## Code Quality Standards

### TypeScript Best Practices

```typescript
// Use explicit return types for public APIs
public function process(input: string): Result<string, ProcessingError> {
  // Implementation
}

// Prefer interfaces for public APIs, types for internal
export interface UserService {
  getUser(id: UserId): Promise<Result<User, UserError>>;
  createUser(data: CreateUserRequest): Promise<Result<User, UserError>>;
}

// Use readonly for immutable data
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
  readonly retries: number;
}

// Use utility types for better type safety
type UserUpdate = Partial<Omit<User, 'id' | 'createdAt'>>;

// Use generics with constraints
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
}

// Use conditional types for advanced patterns
type ApiResponse<T> = T extends string
  ? { message: T }
  : { data: T };

// Use mapped types for object transformations
type Optional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
```

### Error Handling Patterns

```typescript
// Create specific error classes
export class ValidationError extends Error {
  constructor(
    message: string,
    public readonly field: string,
    public readonly code: string
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class NetworkError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly response?: any
  ) {
    super(message);
    this.name = 'NetworkError';
  }
}

// Use Result types for operations that can fail
export type OperationResult<T> =
  | { success: true; data: T }
  | { success: false; error: Error };

// Create helper functions for error handling
export const createSuccess = <T>(data: T): OperationResult<T> => ({
  success: true,
  data,
});

export const createError = (error: Error): OperationResult<never> => ({
  success: false,
  error,
});

// Error boundary for async operations
export async function safeOperation<T>(
  operation: () => Promise<T>
): Promise<OperationResult<T>> {
  try {
    const data = await operation();
    return createSuccess(data);
  } catch (error) {
    return createError(error instanceof Error ? error : new Error(String(error)));
  }
}
```

---

## Full Session Checklist

**Before Starting:**

- [ ] Node.js and npm installed
- [ ] TypeScript project initialized
- [ ] Configuration files set up (tsconfig, jest, eslint, prettier)
- [ ] Development dependencies installed
- [ ] Git hooks configured
- [ ] Todo list created with all tasks

**For Each Module/Feature:**

- [ ] Write failing tests (RED)
- [ ] Verify tests fail
- [ ] Implement minimum code (GREEN)
- [ ] Verify tests pass
- [ ] Type checking passes (REFACTOR)
- [ ] Format code with Prettier
- [ ] Lint with ESLint
- [ ] Update todo (mark complete, next in_progress)
- [ ] Commit with descriptive message (COMMIT)

**After Each Phase:**

- [ ] Run full test suite: `npm run test:coverage`
- [ ] All tests passing with good coverage
- [ ] Type checking: `npm run type-check`
- [ ] Code formatted: `npm run format`
- [ ] Linting clean: `npm run lint`
- [ ] Documentation updated
- [ ] Commit documentation update
- [ ] Push all commits to remote

**Session Complete:**

- [ ] All todos completed
- [ ] Full test suite passing
- [ ] Type checking passes without errors
- [ ] Code coverage meets standards (>90%)
- [ ] Documentation updated
- [ ] All commits pushed
- [ ] Clean working directory (`git status`)
- [ ] Build succeeds: `npm run build`

---

## Example Session Flow

```bash
# 1. Setup
npm install
npm run type-check  # Baseline type checking
npm test  # Baseline tests (X tests passing)

# 2. Create todo list
# Use your tracker of choice with 8-12 tasks

# 3. For each task:
#    a. Write tests (RED)
npm test -- Feature.test.ts  # FAIL ✅
#    b. Implement (GREEN)
npm test -- Feature.test.ts  # PASS ✅
#    c. Type check & Format (REFACTOR)
npm run type-check && npm run format && npm run lint:fix
#    d. Commit
git add -A && git commit -m "feat(service): ..."
#    e. Update todo (mark complete)

# 4. After all tasks:
npm run test:coverage  # All X+Y tests passing ✅
npm run type-check     # All types valid ✅
npm run build          # Build succeeds ✅
git add -A && git commit -m "docs: ..."
git push

# 5. Celebrate! 🎉
```

---

## Advanced TypeScript Patterns

### Higher-Order Types

```typescript
// Async function type extraction
type AsyncReturnType<T extends (...args: any) => Promise<any>> =
  T extends (...args: any) => Promise<infer R> ? R : never;

// Function parameter extraction
type FirstParameter<T extends (...args: any) => any> =
  T extends (arg: infer A, ...args: any) => any ? A : never;

// Deep readonly utility
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

// Optional deep properties
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Recursive type utilities
type Paths<T, Key extends string = ''> = T extends object
  ? {
      [K in keyof T]: `${string & K}` extends Key
        ? never
        : `${Key & string}${Key extends '' ? '' : '.'}${string & K}` | Paths<T[K], `${Key & string}.${string & K}`>;
    }[keyof T]
  : never;

// Usage example
interface User {
  id: string;
  profile: {
    name: string;
    address: {
      street: string;
      city: string;
    };
  };
}

type UserPaths = Paths<User>; // "id" | "profile" | "profile.name" | "profile.address" | "profile.address.street" | "profile.address.city"
```

### Generic Component Patterns

```typescript
// Generic repository pattern
interface Repository<T, K> {
  findById(id: K): Promise<T | null>;
  save(entity: T): Promise<T>;
  delete(id: K): Promise<boolean>;
  find(filter: Partial<T>): Promise<T[]>;
}

// Generic service with dependency injection
abstract class BaseService<T, K> {
  constructor(protected repository: Repository<T, K>) {}

  async getById(id: K): Promise<Result<T, Error>> {
    try {
      const entity = await this.repository.findById(id);
      if (!entity) {
        return err(new Error(`Entity with id ${id} not found`));
      }
      return ok(entity);
    } catch (error) {
      return err(error instanceof Error ? error : new Error(String(error)));
    }
  }

  async create(data: Omit<T, 'id'>): Promise<Result<T, Error>> {
    try {
      // Validation logic here
      const entity = await this.repository.save(data as T);
      return ok(entity);
    } catch (error) {
      return err(error instanceof Error ? error : new Error(String(error)));
    }
  }
}

// Concrete implementation
class UserService extends BaseService<User, string> {
  constructor(repository: Repository<User, string>) {
    super(repository);
  }

  async findByEmail(email: string): Promise<Result<User, Error>> {
    try {
      const users = await this.repository.find({ email } as Partial<User>);
      if (users.length === 0) {
        return err(new Error(`User with email ${email} not found`));
      }
      return ok(users[0]);
    } catch (error) {
      return err(error instanceof Error ? error : new Error(String(error)));
    }
  }
}
```

### Functional Programming Patterns

```typescript
// Either monad for error handling
type Either<L, R> = Left<L> | Right<R>;

interface Left<L> {
  _tag: 'Left';
  left: L;
}

interface Right<R> {
  _tag: 'Right';
  right: R;
}

const left = <L>(value: L): Either<L, never> => ({ _tag: 'Left', left: value });
const right = <R>(value: R): Either<never, R> => ({ _tag: 'Right', right: value });

const isLeft = <L, R>(either: Either<L, R>): either is Left<L> =>
  either._tag === 'Left';

const isRight = <L, R>(either: Either<L, R>): either is Right<R> =>
  either._tag === 'Right';

const fold = <L, R, T>(
  either: Either<L, R>,
  onLeft: (left: L) => T,
  onRight: (right: R) => T
): T => isLeft(either) ? onLeft(either.left) : onRight(either.right);

const map = <L, R, R2>(
  either: Either<L, R>,
  fn: (right: R) => R2
): Either<L, R2> => isLeft(either)
  ? either
  : right(fn(either.right));

const flatMap = <L, R, R2>(
  either: Either<L, R>,
  fn: (right: R) => Either<L, R2>
): Either<L, R2> => isLeft(either)
  ? either
  : fn(either.right);

// Usage example
function parseUser(json: string): Either<Error, User> {
  try {
    const user = JSON.parse(json);
    return right(user);
  } catch (error) {
    return left(error instanceof Error ? error : new Error('Parse error'));
  }
}

function validateUser(user: any): Either<ValidationError[], User> {
  const errors: ValidationError[] = [];

  if (!user.id) errors.push(new ValidationError('id is required', 'id'));
  if (!user.name) errors.push(new ValidationError('name is required', 'name'));
  if (!user.email) errors.push(new ValidationError('email is required', 'email'));

  return errors.length > 0 ? left(errors) : right(user);
}

// Chain operations
const result = pipe(
  parseUser('{"id": "123", "name": "John"}'),
  userEither => flatMap(userEither, validateUser),
  userEither => map(userEither, user => ({ ...user, processed: true })),
);

const finalResult = fold(
  result,
  errors => console.error('Validation failed:', errors),
  user => console.log('Processed user:', user)
);
```

---

## Anti-Patterns to Avoid

❌ **DON'T:**

- Use `any` type without justification
- Ignore TypeScript compilation errors
- Write tests without proper type checking
- Use `@ts-ignore` without documented reasons
- Create overly complex generic constraints
- Skip interface definitions for public APIs
- Ignore null/undefined handling
- Commit code that doesn't pass `npm run type-check`
- Leave todos as "in_progress" when complete
- Write vague commit messages
- Push without running full test suite and type checking
- Skip documentation for types and interfaces

✅ **DO:**

- Use specific types and interfaces
- Let TypeScript catch errors at compile-time
- Write tests that verify type safety
- Document reasons for `@ts-ignore` if absolutely necessary
- Keep generics simple and understandable
- Define clear interfaces for public APIs
- Handle null/undefined explicitly
- Only commit code that passes all checks
- Update todos immediately after completing tasks
- Write descriptive, detailed commit messages
- Run full test suite and type checking before pushing
- Document all public types and interfaces

---

## Tools & Commands Reference

### Essential Commands

```bash
# Project management
npm init -y                     # Initialize package.json
npm install typescript --save-dev  # Install TypeScript
npm install @types/node --save-dev # Install Node.js types
npx tsc --init                 # Initialize tsconfig.json

# Building
npm run build                  # Compile TypeScript
npm run build:watch            # Compile with watch mode
npm run clean                  # Clean build artifacts

# Testing
npm test                       # Run all tests
npm run test:watch             # Run tests in watch mode
npm run test:coverage          # Run tests with coverage
npm run test:unit              # Run unit tests only
npm run test:integration       # Run integration tests only

# Code quality
npm run type-check             # Type checking without compilation
npm run lint                   # Run ESLint
npm run lint:fix               # Fix linting issues automatically
npm run format                 # Format code with Prettier
npm run format:check           # Check formatting without fixing

# Development
npm run dev                    # Start development mode
npm start                      # Start compiled application
npm run clean                  # Clean build artifacts
```

### TypeScript-Specific Commands

```bash
# Type checking
npx tsc --noEmit               # Type check without emitting files
npx tsc --noEmit --strict      # Strict type checking
npx tsc --listFiles            # List files to be compiled

# Compilation
npx tsc                        # Compile according to tsconfig.json
npx tsc --project ./tsconfig.build.json  # Use specific config
npx tsc --outDir ./dist        # Specify output directory

# Declaration files
npx tsc --declaration          # Generate .d.ts files
npx tsc --declarationMap       # Generate declaration source maps
```

### Testing Commands

```bash
# Jest specific
npx jest                        # Run all tests
npx jest --watch                # Watch mode
npx jest --coverage            # With coverage
npx jest --testNamePattern="test name"  # Run specific test
npx jest --pathPattern="service"  # Run tests in path
npx jest --updateSnapshot      # Update snapshots

# Debugging tests
npx jest --runInBand           # Run tests serially
npx jest --detectOpenHandles   # Detect open handles
npx jest --forceExit           # Force exit after tests
```

---

## Success Metrics

After following this workflow, you should have:

✅ **Type Safety** - Compile-time guarantee of type correctness
✅ **Runtime Confidence** - Comprehensive test coverage
✅ **Code Quality** - Consistent formatting and linting
✅ **Documentation** - Self-documenting types and interfaces
✅ **Maintainability** - Easy refactoring with type safety
✅ **Developer Experience** - Excellent IDE support and autocompletion
✅ **Error Handling** - Robust error handling with proper types
✅ **Performance** - Optimized builds and efficient code
✅ **Modern Practices** - Up-to-date with TypeScript features
✅ **Clear History** - Commits showing development progression

---

## Troubleshooting

### TypeScript Compilation Issues

```bash
# Check for type errors
npx tsc --noEmit

# Get detailed error information
npx tsc --pretty --noEmit

# Check specific files
npx tsc src/**/*.ts --noEmit

# Update types
npm update @types/node @types/jest
```

### Test Issues

```bash
# Debug test compilation
npx jest --no-cache --verbose

# Check test configuration
npx jest --showConfig

# Run specific test with debugging
node --inspect-brk node_modules/.bin/jest --runInBand test-file.test.ts
```

### Build Performance Issues

```bash
# Use TypeScript project references
# Add to tsconfig.json:
{
  "compilerOptions": {
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}

# Use esbuild for faster development builds
npm install esbuild --save-dev
npx esbuild src/index.ts --bundle --platform=node --outfile=dist/index.js
```

### Dependency Issues

```bash
# Check for type conflicts
npx tsc --noEmit --traceResolution

# Clean install
rm -rf node_modules package-lock.json
npm install

# Check for outdated dependencies
npm outdated
```

---

**Remember:** This workflow leverages TypeScript's static typing to catch errors early while maintaining excellent developer experience. The combination of compile-time safety and comprehensive testing creates a robust development process.

**When in doubt, let the TypeScript compiler be your guide!** 📘
