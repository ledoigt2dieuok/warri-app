# Phase 2: Code Quality & Improvements - Detailed Implementation Guide

**Date Created**: June 5, 2026  
**Status**: Ready for Implementation  
**Target Duration**: 6-7 weeks

---

## Overview

Phase 2 focuses on enhancing code quality across the warri-app codebase. This phase runs in parallel with Phase 1 features and includes:

1. **Performance Optimization** (1-2 weeks)
2. **Error Handling & Recovery** (1 week)
3. **Code Refactoring** (2 weeks)
4. **Accessibility Enhancements** (1-2 weeks)
5. **Dependency Management** (1 week)

---

## 1. Performance Optimization 🚀

### Goal
Reduce bundle size, improve render times, and enhance API response handling.

### 1.1 Redux Selector Memoization

**Problem**: Component re-renders when unrelated state changes.

**Solution**: Use `reselect` library for memoized selectors.

**Implementation**:

```typescript
// File: src/store/selectors/authSelectors.ts (NEW)
import { createSelector } from 'reselect';
import { RootState } from './store';

// Base selectors
const selectAuthState = (state: RootState) => state.auth;

// Memoized selectors
export const selectUser = createSelector(
  [selectAuthState],
  (auth) => auth.user
);

export const selectIsAuthenticated = createSelector(
  [selectAuthState],
  (auth) => auth.isAuthenticated
);

export const selectToken = createSelector(
  [selectAuthState],
  (auth) => auth.token
);

// Complex selector
export const selectUserWithMeta = createSelector(
  [selectUser],
  (user) => ({
    ...user,
    initials: user?.name
      ?.split(' ')
      .map((n) => n[0])
      .join(''),
  })
);
```

**Usage in Components**:

```typescript
// Before: prone to re-renders
const user = useSelector((state) => state.auth.user);

// After: memoized, only updates when user actually changes
const user = useSelector(selectUser);
```

**Files to Create/Modify**:
- `src/store/selectors/authSelectors.ts` - New
- `src/store/selectors/transactionSelectors.ts` - New
- `src/store/selectors/index.ts` - New
- All components using selectors - Update imports

**Package to Add**:
```bash
npm install reselect
npm install --save-dev @types/reselect
```

**Testing**:
```typescript
// File: tests/store/selectors/authSelectors.test.ts
describe('Auth Selectors', () => {
  it('selectUser should memoize results', () => {
    const selector = selectUser;
    const state1 = { auth: { user: { id: '1', name: 'John' } } };
    const state2 = { auth: { user: { id: '1', name: 'John' } } };
    
    const result1 = selector(state1);
    const result2 = selector(state2);
    
    expect(result1).toBe(result2); // Same reference
  });
});
```

**Timeline**: 3-4 days

---

### 1.2 Component Re-render Optimization

**Problem**: Components re-render unnecessarily when props haven't changed.

**Solution**: Use `React.memo` and `useMemo`.

**Implementation**:

```typescript
// File: src/components/Button.tsx (MODIFY)
import React, { memo, useCallback } from 'react';

interface ButtonProps {
  label: string;
  onPress: () => void;
  disabled?: boolean;
}

export const Button = memo(
  ({ label, onPress, disabled }: ButtonProps) => {
    return (
      <TouchableOpacity
        disabled={disabled}
        onPress={onPress}
        style={[styles.button, disabled && styles.disabled]}
      >
        <Text style={styles.text}>{label}</Text>
      </TouchableOpacity>
    );
  },
  (prevProps, nextProps) => {
    // Custom comparison for memoization
    return (
      prevProps.label === nextProps.label &&
      prevProps.disabled === nextProps.disabled &&
      prevProps.onPress === nextProps.onPress
    );
  }
);

Button.displayName = 'Button';
```

**Usage in Parent Components**:

```typescript
const MyScreen = () => {
  // Stable reference using useCallback
  const handlePress = useCallback(() => {
    console.log('Button pressed');
  }, []);

  return <Button label="Press Me" onPress={handlePress} />;
};
```

**Components to Optimize**:
- `src/components/Button.tsx`
- `src/components/Card.tsx`
- `src/components/TextInput.tsx`
- `src/components/Loading.tsx`
- `src/components/ErrorAlert.tsx`

**Files to Create/Modify**:
```
src/components/
├── Button.tsx (MODIFY)
├── Card.tsx (MODIFY)
├── TextInput.tsx (MODIFY)
├── Loading.tsx (MODIFY)
├── ErrorAlert.tsx (MODIFY)
└── index.ts (MODIFY)
```

**Testing**:
```typescript
// File: tests/components/Button.test.tsx
describe('Button Component', () => {
  it('should not re-render when props are the same', () => {
    const { rerender } = render(
      <Button label="Test" onPress={jest.fn()} />
    );

    const firstRender = jest.fn();
    rerender(
      <Button label="Test" onPress={jest.fn()} />
    );

    // Verify component used memo correctly
    expect(firstRender).not.toHaveBeenCalled();
  });
});
```

**Timeline**: 3-4 days

---

### 1.3 Image Compression & Lazy Loading

**Problem**: Large images slow down app loading and increase bundle size.

**Solution**: Compress images and implement lazy loading.

**Implementation**:

```typescript
// File: src/utils/imageOptimization.ts (NEW)
import { Image } from 'react-native';

interface ImageConfig {
  width: number;
  height: number;
  quality?: number;
}

export const optimizeImage = (
  uri: string,
  config: ImageConfig
): string => {
  // Return optimized URI or cached version
  // Implementation depends on image service used
  return uri;
};

// Lazy load images
export const lazyLoadImage = (uri: string): Promise<void> => {
  return new Promise((resolve, reject) => {
    Image.prefetch(uri)
      .then(() => resolve())
      .catch((error) => reject(error));
  });
};
```

**Lazy Image Component**:

```typescript
// File: src/components/LazyImage.tsx (NEW)
import React, { useState, useEffect } from 'react';
import { Image, ActivityIndicator, View } from 'react-native';

interface LazyImageProps {
  source: { uri: string };
  style?: any;
}

export const LazyImage: React.FC<LazyImageProps> = ({
  source,
  style,
}) => {
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    Image.prefetch(source.uri)
      .then(() => setLoading(false))
      .catch(() => setLoading(false));
  }, [source.uri]);

  return (
    <View style={style}>
      {loading && (
        <ActivityIndicator
          style={style}
          size="large"
          color="#0000ff"
        />
      )}
      <Image
        source={source}
        style={[style, { display: loading ? 'none' : 'flex' }]}
      />
    </View>
  );
};
```

**Files to Create/Modify**:
- `src/utils/imageOptimization.ts` - New
- `src/components/LazyImage.tsx` - New
- `src/screens/DashboardScreen.tsx` - Use LazyImage
- `src/screens/ProfileScreen.tsx` - Use LazyImage

**Package to Consider**:
```bash
npm install expo-image-picker  # If not already installed
npm install fast-image         # For advanced image handling
```

**Timeline**: 2-3 days

---

### 1.4 API Call Batching & Caching

**Problem**: Multiple redundant API calls and no response caching.

**Solution**: Implement request batching and response caching.

**Implementation**:

```typescript
// File: src/services/cacheService.ts (NEW)
interface CacheEntry<T> {
  data: T;
  timestamp: number;
  ttl: number; // Time to live in ms
}

class CacheService {
  private cache = new Map<string, CacheEntry<any>>();

  set<T>(key: string, data: T, ttl: number = 5 * 60 * 1000) {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl,
    });
  }

  get<T>(key: string): T | null {
    const entry = this.cache.get(key);
    
    if (!entry) return null;
    
    const isExpired = Date.now() - entry.timestamp > entry.ttl;
    if (isExpired) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.data as T;
  }

  clear(pattern?: string) {
    if (pattern) {
      Array.from(this.cache.keys())
        .filter((key) => key.includes(pattern))
        .forEach((key) => this.cache.delete(key));
    } else {
      this.cache.clear();
    }
  }
}

export const cacheService = new CacheService();
```

**Request Batching**:

```typescript
// File: src/services/batchRequestService.ts (NEW)
class BatchRequestService {
  private queue: Array<{
    url: string;
    options: any;
    resolve: (data: any) => void;
    reject: (error: any) => void;
  }> = [];
  
  private isProcessing = false;
  private batchDelay = 10; // ms

  add(url: string, options: any): Promise<any> {
    return new Promise((resolve, reject) => {
      this.queue.push({ url, options, resolve, reject });
      
      if (!this.isProcessing) {
        this.processBatch();
      }
    });
  }

  private processBatch() {
    this.isProcessing = true;
    
    setTimeout(async () => {
      const batch = [...this.queue];
      this.queue = [];

      // Execute all requests in parallel
      const results = await Promise.allSettled(
        batch.map((item) =>
          fetch(item.url, item.options)
        )
      );

      // Resolve/reject each request
      results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          batch[index].resolve(result.value);
        } else {
          batch[index].reject(result.reason);
        }
      });

      this.isProcessing = false;

      // Process any queued requests
      if (this.queue.length > 0) {
        this.processBatch();
      }
    }, this.batchDelay);
  }
}

export const batchRequestService = new BatchRequestService();
```

**Usage in API Service**:

```typescript
// File: src/services/api.ts (MODIFY)
import { cacheService } from './cacheService';

export const getTransactionHistory = async (
  userId: string
): Promise<Transaction[]> => {
  const cacheKey = `transactions_${userId}`;
  
  // Check cache first
  const cached = cacheService.get<Transaction[]>(cacheKey);
  if (cached) return cached;

  // Fetch from API
  const response = await fetch(
    `/api/transactions/${userId}`
  );
  const data = await response.json();

  // Cache for 5 minutes
  cacheService.set(cacheKey, data, 5 * 60 * 1000);

  return data;
};
```

**Files to Create/Modify**:
- `src/services/cacheService.ts` - New
- `src/services/batchRequestService.ts` - New
- `src/services/api.ts` - Integrate caching
- `src/services/authService.ts` - Use cache
- `src/services/paymentService.ts` - Use cache

**Testing**:
```typescript
// File: tests/services/cacheService.test.ts
describe('CacheService', () => {
  it('should return cached data within TTL', () => {
    const service = new CacheService();
    const data = { id: 1, name: 'Test' };
    
    service.set('key', data);
    expect(service.get('key')).toEqual(data);
  });

  it('should return null for expired cache', (done) => {
    const service = new CacheService();
    service.set('key', { id: 1 }, 100);
    
    setTimeout(() => {
      expect(service.get('key')).toBeNull();
      done();
    }, 150);
  });
});
```

**Timeline**: 3-4 days

---

### 1.5 Bundle Size Reduction

**Problem**: Large JavaScript bundle increases app load time.

**Solution**: Code splitting, tree shaking, and dynamic imports.

**Implementation**:

```typescript
// File: src/screens/index.ts (MODIFY)
// Dynamic imports for code splitting
export const LoginScreen = lazy(() =>
  import('./LoginScreen').then((m) => ({
    default: m.LoginScreen,
  }))
);

export const DashboardScreen = lazy(() =>
  import('./DashboardScreen').then((m) => ({
    default: m.DashboardScreen,
  }))
);
```

**Analysis Tool**:
```bash
# Install bundle analyzer
npm install --save-dev react-native-bundle-visualizer

# Generate report
npx react-native-bundle-visualizer --entry-file index.js
```

**Optimization Checklist**:
- [ ] Remove unused dependencies
- [ ] Use `tree-shaking` in bundler config
- [ ] Implement code splitting for large features
- [ ] Lazy load screens and components
- [ ] Remove large libraries if possible

**Timeline**: 2-3 days

**Total for Section 1.1-1.5**: 15-19 days

---

## 2. Error Handling & Recovery 🛡️

### Goal
Create robust error handling with user-friendly messages and proper logging.

### 2.1 Error Boundary Component

**Implementation**:

```typescript
// File: src/components/ErrorBoundary.tsx (NEW)
import React, { ReactNode } from 'react';
import { View, Text, TouchableOpacity } from 'react-native';
import { errorService } from '../services/errorService';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends React.Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: any) {
    // Log to service
    errorService.logError(error, errorInfo);
  }

  handleReset = () => {
    this.setState({ hasError: false, error: undefined });
  };

  render() {
    if (this.state.hasError) {
      return (
        <View style={styles.container}>
          <Text style={styles.title}>Oops! Something went wrong</Text>
          <Text style={styles.message}>
            {this.state.error?.message ||
              'An unexpected error occurred'}
          </Text>
          <TouchableOpacity
            style={styles.button}
            onPress={this.handleReset}
          >
            <Text style={styles.buttonText}>Try Again</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = {
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 10,
  },
  message: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    marginBottom: 20,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 5,
  },
  buttonText: {
    color: '#fff',
    fontWeight: 'bold',
  },
};
```

**Usage**:

```typescript
// File: src/App.tsx (MODIFY)
import { ErrorBoundary } from './components/ErrorBoundary';

export default function App() {
  return (
    <ErrorBoundary>
      <RootNavigator />
    </ErrorBoundary>
  );
}
```

### 2.2 Centralized Error Service

**Implementation**:

```typescript
// File: src/services/errorService.ts (NEW)
import * as Sentry from '@sentry/react-native';

export enum ErrorCategory {
  NETWORK = 'NETWORK',
  VALIDATION = 'VALIDATION',
  AUTHENTICATION = 'AUTHENTICATION',
  AUTHORIZATION = 'AUTHORIZATION',
  SERVER = 'SERVER',
  UNKNOWN = 'UNKNOWN',
}

interface ErrorContext {
  userId?: string;
  screen?: string;
  action?: string;
  metadata?: Record<string, any>;
}

class ErrorService {
  private context: ErrorContext = {};

  setContext(context: Partial<ErrorContext>) {
    this.context = { ...this.context, ...context };
  }

  logError(
    error: Error,
    category: ErrorCategory = ErrorCategory.UNKNOWN,
    context?: Partial<ErrorContext>
  ) {
    const fullContext = { ...this.context, ...context };

    // Log to console in development
    if (__DEV__) {
      console.error('Error:', error, fullContext);
    }

    // Log to Sentry in production
    Sentry.captureException(error, {
      tags: { category },
      contexts: { app: fullContext },
    });
  }

  logMessage(message: string, level: 'info' | 'warning' | 'error' = 'info') {
    if (__DEV__) {
      console.log(`[${level.toUpperCase()}]`, message);
    }

    Sentry.captureMessage(message, level);
  }

  getUserFriendlyMessage(error: any): string {
    if (error.code === 'NETWORK_ERROR') {
      return 'Network connection failed. Please check your internet.';
    }

    if (error.code === 'TIMEOUT') {
      return 'Request timed out. Please try again.';
    }

    if (error.code === 'VALIDATION_ERROR') {
      return error.message || 'Please check your input and try again.';
    }

    if (error.code === 'UNAUTHORIZED') {
      return 'Please log in again.';
    }

    if (error.code === 'FORBIDDEN') {
      return 'You do not have permission to perform this action.';
    }

    return 'Something went wrong. Please try again later.';
  }
}

export const errorService = new ErrorService();
```

### 2.3 Network Error Handling

**Implementation**:

```typescript
// File: src/services/networkErrorHandler.ts (NEW)
import { AxiosError } from 'axios';

interface ApiError {
  code: string;
  message: string;
  statusCode: number;
  userMessage: string;
}

export const handleNetworkError = (error: any): ApiError => {
  // Network error
  if (!error.response) {
    return {
      code: 'NETWORK_ERROR',
      message: error.message,
      statusCode: 0,
      userMessage: 'Network connection failed',
    };
  }

  const { status, data } = error.response;

  // Server errors
  if (status >= 500) {
    return {
      code: 'SERVER_ERROR',
      message: data?.message || 'Server error',
      statusCode: status,
      userMessage: 'Server error. Please try again later.',
    };
  }

  // Client errors
  if (status === 401) {
    return {
      code: 'UNAUTHORIZED',
      message: 'Invalid credentials',
      statusCode: status,
      userMessage: 'Please log in again.',
    };
  }

  if (status === 403) {
    return {
      code: 'FORBIDDEN',
      message: 'Access denied',
      statusCode: status,
      userMessage: 'You do not have permission.',
    };
  }

  if (status === 404) {
    return {
      code: 'NOT_FOUND',
      message: 'Resource not found',
      statusCode: status,
      userMessage: 'Resource not found.',
    };
  }

  // Validation errors
  if (status === 400) {
    return {
      code: 'VALIDATION_ERROR',
      message: data?.message || 'Invalid request',
      statusCode: status,
      userMessage: data?.message || 'Please check your input.',
    };
  }

  return {
    code: 'UNKNOWN_ERROR',
    message: data?.message || 'Unknown error',
    statusCode: status,
    userMessage: 'An unexpected error occurred.',
  };
};
```

### 2.4 Async Error Handling

**Implementation**:

```typescript
// File: src/utils/asyncHandler.ts (NEW)
export const asyncHandler = <T extends any[], R>(
  fn: (...args: T) => Promise<R>
) => {
  return async (...args: T): Promise<[R | null, Error | null]> => {
    try {
      const result = await fn(...args);
      return [result, null];
    } catch (error) {
      return [null, error as Error];
    }
  };
};

// Usage
const safeGetTransactions = asyncHandler(async (userId: string) => {
  const response = await fetch(`/api/transactions/${userId}`);
  return response.json();
});

const [transactions, error] = await safeGetTransactions('user123');
if (error) {
  console.error('Failed to fetch:', error);
} else {
  console.log('Transactions:', transactions);
}
```

**Files to Create/Modify**:
- `src/components/ErrorBoundary.tsx` - New
- `src/services/errorService.ts` - New
- `src/services/networkErrorHandler.ts` - New
- `src/utils/asyncHandler.ts` - New
- `src/App.tsx` - Integrate ErrorBoundary
- All API calls - Use error handler

**Package to Add**:
```bash
npm install @sentry/react-native
```

**Timeline**: 4-5 days

---

## 3. Code Refactoring 🔨

### Goal
Simplify code, improve readability, and reduce complexity.

### 3.1 Identify Complex Functions

**Analysis**:

```bash
# Install complexity analyzer
npm install --save-dev complexity-report

# Analyze codebase
npx complexity-report --output json src/ > complexity.json
```

**Target**: Cyclomatic complexity < 10

### 3.2 Refactor Payment Service

**Before**:

```typescript
// src/services/paymentService.ts (BEFORE)
export const processPayment = async (
  amount: number,
  provider: string,
  phoneNumber: string,
  userId: string
) => {
  // Long, complex function
  if (!amount || amount <= 0) return { error: 'Invalid amount' };
  if (!phoneNumber) return { error: 'Invalid phone' };
  if (!provider) return { error: 'Invalid provider' };

  const fee = amount * (provider === 'orange' ? 0.015 : 0.018);
  const total = amount + fee;

  if (provider === 'orange') {
    // Complex Orange logic
  } else if (provider === 'mtn') {
    // Complex MTN logic
  } else if (provider === 'wave') {
    // Complex Wave logic
  }

  // ... 200+ lines
};
```

**After**:

```typescript
// src/services/paymentService.ts (AFTER)
import { PaymentValidator } from './validators/paymentValidator';
import { PaymentCalculator } from './calculators/paymentCalculator';
import { ProviderFactory } from './providers/ProviderFactory';

export const processPayment = async (
  amount: number,
  provider: string,
  phoneNumber: string,
  userId: string
) => {
  // Validate inputs
  const validation = PaymentValidator.validate({
    amount,
    phoneNumber,
    provider,
  });

  if (!validation.isValid) {
    return { error: validation.error };
  }

  // Calculate fees
  const { fee, total } = PaymentCalculator.calculate(
    amount,
    provider
  );

  // Get provider instance
  const paymentProvider =
    ProviderFactory.getProvider(provider);

  // Process payment
  return paymentProvider.process({
    amount,
    total,
    fee,
    phoneNumber,
    userId,
  });
};
```

**New Files**:

```typescript
// src/services/validators/paymentValidator.ts (NEW)
interface PaymentInput {
  amount: number;
  phoneNumber: string;
  provider: string;
}

interface ValidationResult {
  isValid: boolean;
  error?: string;
}

export class PaymentValidator {
  static validate(input: PaymentInput): ValidationResult {
    if (!input.amount || input.amount <= 0) {
      return { isValid: false, error: 'Invalid amount' };
    }

    if (!input.phoneNumber) {
      return { isValid: false, error: 'Invalid phone number' };
    }

    if (!input.provider) {
      return { isValid: false, error: 'Invalid provider' };
    }

    return { isValid: true };
  }
}
```

```typescript
// src/services/calculators/paymentCalculator.ts (NEW)
const PROVIDER_FEES = {
  orange: 0.015,
  mtn: 0.018,
  wave: 0.012,
};

export class PaymentCalculator {
  static calculate(amount: number, provider: string) {
    const feeRate = PROVIDER_FEES[provider] || 0.015;
    const fee = amount * feeRate;

    return {
      amount,
      fee,
      total: amount + fee,
    };
  }
}
```

### 3.3 Extract Reusable Patterns

**Identify Pattern**: Form validation

**Before**:
```typescript
// Repeated in multiple screens
const [email, setEmail] = useState('');
const [emailError, setEmailError] = useState('');

const validateEmail = () => {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!regex.test(email)) {
    setEmailError('Invalid email');
    return false;
  }
  return true;
};
```

**After**:
```typescript
// src/hooks/useFormField.ts (NEW)
interface UseFormFieldOptions {
  validator?: (value: string) => boolean;
  errorMessage?: string;
}

export const useFormField = (
  initialValue: string = '',
  options?: UseFormFieldOptions
) => {
  const [value, setValue] = useState(initialValue);
  const [error, setError] = useState('');

  const validate = () => {
    if (options?.validator && !options.validator(value)) {
      setError(options.errorMessage || 'Invalid input');
      return false;
    }
    setError('');
    return true;
  };

  return {
    value,
    setValue,
    error,
    validate,
    reset: () => {
      setValue(initialValue);
      setError('');
    },
  };
};
```

**Usage**:
```typescript
const email = useFormField('', {
  validator: (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
  errorMessage: 'Invalid email',
});

const password = useFormField('', {
  validator: (value) => value.length >= 8,
  errorMessage: 'Password must be at least 8 characters',
});
```

### 3.4 Remove Code Duplication

**Identify**: Duplicate validation logic across screens

**Solution**: Create validation utilities

```typescript
// src/utils/validation.ts (ENHANCE)
export const validators = {
  email: (value: string) =>
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),

  phone: (value: string) =>
    /^\+?1?\d{9,15}$/.test(value),

  password: (value: string) =>
    value.length >= 8 &&
    /[A-Z]/.test(value) &&
    /[0-9]/.test(value),

  amount: (value: number) =>
    value > 0 && value <= 999999,
};

export const getValidationError = (
  field: string,
  value: any
): string | null => {
  const validations: Record<string, string> = {
    email: 'Invalid email address',
    phone: 'Invalid phone number',
    password: 'Password must be at least 8 characters with uppercase and numbers',
    amount: 'Amount must be between 0 and 999999',
  };

  if (!validators[field] || !validators[field](value)) {
    return validations[field] || 'Invalid input';
  }

  return null;
};
```

### 3.5 Improve Type Safety

**Enable Strict Mode**:

```json
// tsconfig.json (MODIFY)
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

**Files to Create/Modify**:
- `src/services/validators/paymentValidator.ts` - New
- `src/services/calculators/paymentCalculator.ts` - New
- `src/hooks/useFormField.ts` - New
- `src/utils/validation.ts` - Enhance
- Various service files - Refactor
- `tsconfig.json` - Enable strict mode

**Timeline**: 7-10 days

---

## 4. Accessibility Enhancements ♿

### Goal
Ensure app is usable by everyone, including users with disabilities.

### 4.1 Screen Reader Optimization

**Implementation**:

```typescript
// File: src/components/AccessibleButton.tsx (NEW)
import React from 'react';
import { TouchableOpacity, AccessibilityInfo } from 'react-native';

interface AccessibleButtonProps {
  label: string;
  onPress: () => void;
  accessibilityHint?: string;
  testID?: string;
}

export const AccessibleButton: React.FC<AccessibleButtonProps> = ({
  label,
  onPress,
  accessibilityHint,
  testID,
}) => {
  return (
    <TouchableOpacity
      onPress={onPress}
      accessible={true}
      accessibilityLabel={label}
      accessibilityHint={accessibilityHint}
      accessibilityRole="button"
      testID={testID}
    >
      {/* Content */}
    </TouchableOpacity>
  );
};
```

**Audit Existing Components**:

```bash
# Check for accessibility issues
npm install --save-dev @testing-library/react-native

# Run accessibility tests
npm run test:a11y
```

### 4.2 Keyboard Navigation

**Implementation**:

```typescript
// File: src/screens/LoginScreen.tsx (MODIFY)
import React, { useRef } from 'react';
import { View, TextInput } from 'react-native';

export const LoginScreen = () => {
  const passwordInputRef = useRef<TextInput>(null);

  const handleEmailSubmit = () => {
    passwordInputRef.current?.focus();
  };

  return (
    <View>
      <TextInput
        placeholder="Email"
        returnKeyType="next"
        onSubmitEditing={handleEmailSubmit}
        testID="email-input"
      />
      <TextInput
        ref={passwordInputRef}
        placeholder="Password"
        returnKeyType="done"
        secureTextEntry
        testID="password-input"
      />
    </View>
  );
};
```

### 4.3 Color Contrast Compliance

**Checklist**:

```typescript
// src/theme/colors.ts (MODIFY)
export const colors = {
  // Ensure 4.5:1 contrast ratio for text
  text: {
    primary: '#1a1a1a', // Dark gray on light background
    secondary: '#4a4a4a',
    inverse: '#ffffff', // White on dark background
  },
  background: {
    light: '#ffffff',
    dark: '#f5f5f5',
  },
  status: {
    success: '#34C759', // Green - contrasts with white
    error: '#FF3B30', // Red - contrasts with white
    warning: '#FF9500', // Orange - contrasts with white
    info: '#007AFF', // Blue - contrasts with white
  },
};

// Utility to check contrast
export const checkContrast = (
  foreground: string,
  background: string
): number => {
  // Calculate relative luminance
  const getLuminance = (color: string) => {
    const rgb = parseInt(color.slice(1), 16);
    const r = (rgb >> 16) & 0xff;
    const g = (rgb >> 8) & 0xff;
    const b = (rgb >> 0) & 0xff;

    const luminance =
      (0.299 * r + 0.587 * g + 0.114 * b) / 255;
    return luminance <= 0.03
      ? luminance / 12.92
      : Math.pow((luminance + 0.055) / 1.055, 2.4);
  };

  const l1 = getLuminance(foreground);
  const l2 = getLuminance(background);

  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);

  return (lighter + 0.05) / (darker + 0.05);
};
```

### 4.4 Voice Command Support

**Implementation**:

```typescript
// File: src/services/voiceService.ts (NEW)
import * as Speech from 'expo-speech';

export class VoiceService {
  static async speak(text: string) {
    await Speech.speak(text, {
      language: 'en',
      pitch: 1,
      rate: 1,
    });
  }

  static stop() {
    Speech.stop();
  }
}
```

**Usage in Components**:

```typescript
import { VoiceService } from '../services/voiceService';

const handleTransferSuccess = async () => {
  // Speak feedback
  await VoiceService.speak(
    'Transfer completed successfully'
  );
};
```

### 4.5 Text Scaling Support

**Implementation**:

```typescript
// File: src/utils/textScaling.ts (NEW)
import { useWindowDimensions } from 'react-native';

export const useScaledFont = () => {
  const { fontScale } = useWindowDimensions();

  return {
    small: 12 * fontScale,
    body: 14 * fontScale,
    subtitle: 16 * fontScale,
    title: 18 * fontScale,
    heading: 24 * fontScale,
  };
};
```

**Usage**:

```typescript
const scaledFonts = useScaledFont();

<Text style={{ fontSize: scaledFonts.body }}>
  This text scales with user settings
</Text>
```

**Files to Create/Modify**:
- `src/components/AccessibleButton.tsx` - New
- `src/services/voiceService.ts` - New
- `src/utils/textScaling.ts` - New
- `src/theme/colors.ts` - Enhance
- All components - Add accessibility props
- `tests/accessibility/` - New accessibility tests

**Package to Add**:
```bash
npm install expo-speech
```

**Timeline**: 5-7 days

---

## 5. Dependency Management 📦

### Goal
Keep dependencies updated and secure.

### 5.1 Update Process

```bash
# Check for outdated packages
npm outdated

# Update minor/patch versions safely
npm update

# Update major versions (with caution)
npm install package-name@latest

# Review changes
git diff package-lock.json
```

### 5.2 Security Audits

```bash
# Run security audit
npm audit

# Fix vulnerabilities automatically
npm audit fix

# Fix with potential breaking changes
npm audit fix --force
```

### 5.3 Remove Unused Dependencies

```bash
# Analyze unused packages
npm install --save-dev depcheck
npx depcheck

# Remove unused
npm uninstall package-name
```

**Timeline**: 3-4 days

---

## Phase 2 Summary

| Task | Timeline | Priority |
|------|----------|----------|
| Performance Optimization | 2-3 wks | HIGH |
| Error Handling | 1 wk | HIGH |
| Code Refactoring | 2 wks | MEDIUM |
| Accessibility | 1-2 wks | MEDIUM |
| Dependencies | 3-4 days | LOW |

**Total Phase 2**: 6-7 weeks

---

## Success Metrics

✅ **Code Quality**:
- Cyclomatic complexity < 10
- TypeScript strict mode enabled
- 0 console errors in CI/CD

✅ **Performance**:
- Bundle size reduced by 10-15%
- Component render time < 100ms
- API cache hit rate > 50%

✅ **Error Handling**:
- 100% error boundary coverage
- All API errors handled
- User-friendly error messages

✅ **Accessibility**:
- WCAG AA compliance
- All interactive elements keyboard accessible
- Contrast ratio 4.5:1+ for all text

---

## Implementation Order

1. **Week 1-2**: Performance Optimization
2. **Week 2**: Error Handling
3. **Week 3-4**: Code Refactoring
4. **Week 4-5**: Accessibility
5. **Week 5-6**: Dependency Management & Testing
6. **Week 6-7**: Buffer & final testing

---

**Ready to enhance warri-app code quality!** 💪
