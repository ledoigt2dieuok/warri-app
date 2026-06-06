# Phase 4: Test Coverage Expansion - Detailed Implementation Guide

**Date Created**: June 5, 2026  
**Status**: Ready for Implementation  
**Target Duration**: 10-14 weeks

---

## Overview

Phase 4 focuses on achieving 85%+ code coverage with comprehensive test suites across all layers of the application.

### Goals
1. Frontend component tests
2. Backend API tests
3. Service tests
4. State management tests
5. E2E tests
6. Performance tests

---

## 1. Frontend Component Tests 🧪

### 1.1 Setup Testing Infrastructure

**Install Dependencies**:

```bash
npm install --save-dev @testing-library/react-native @testing-library/jest-native jest-expo
npm install --save-dev @testing-library/user-event
npm install --save-dev jest-mock-async-storage
```

**Jest Configuration** (`jest.config.js`):

```javascript
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts?(x)', '**/?(*.)+(spec|test).ts?(x)'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

**Test Setup** (`tests/setup.ts`):

```typescript
import '@testing-library/jest-native/extend-expect';
import AsyncStorage from '@react-native-async-storage/async-storage';

// Mock AsyncStorage
jest.mock('@react-native-async-storage/async-storage', () =>
  require('jest-mock-async-storage')
);

// Reset mocks before each test
beforeEach(async () => {
  const { AsyncStorage } = require('jest-mock-async-storage');
  await AsyncStorage.clear();
});
```

### 1.2 Component Tests

**Button Component Test**:

```typescript
// tests/components/Button.test.tsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Button } from '@/components/Button';

describe('Button Component', () => {
  it('should render with label', () => {
    render(<Button label="Press Me" onPress={jest.fn()} />);
    expect(screen.getByText('Press Me')).toBeTruthy();
  });

  it('should call onPress when pressed', () => {
    const onPress = jest.fn();
    render(<Button label="Press Me" onPress={onPress} />);

    fireEvent.press(screen.getByRole('button'));
    expect(onPress).toHaveBeenCalled();
  });

  it('should be disabled when disabled prop is true', () => {
    const onPress = jest.fn();
    render(
      <Button label="Press Me" onPress={onPress} disabled={true} />
    );

    fireEvent.press(screen.getByRole('button'));
    expect(onPress).not.toHaveBeenCalled();
  });

  it('should have accessibility label', () => {
    render(<Button label="Press Me" onPress={jest.fn()} />);
    const button = screen.getByRole('button');
    expect(button.props.accessibilityLabel).toBe('Press Me');
  });
});
```

**TextInput Component Test**:

```typescript
// tests/components/TextInput.test.tsx
import React from 'react';
import { render, screen } from '@testing-library/react-native';
import { TextInput } from '@/components/TextInput';
import userEvent from '@testing-library/user-event';

describe('TextInput Component', () => {
  it('should render with placeholder', () => {
    render(
      <TextInput
        placeholder="Enter text"
        value=""
        onChangeText={jest.fn()}
      />
    );

    expect(screen.getByPlaceholderText('Enter text')).toBeTruthy();
  });

  it('should update value on text change', async () => {
    const onChangeText = jest.fn();
    const { getByDisplayValue } = render(
      <TextInput
        placeholder="Enter text"
        value=""
        onChangeText={onChangeText}
      />
    );

    const input = screen.getByPlaceholderText('Enter text');
    await userEvent.type(input, 'Hello');

    expect(onChangeText).toHaveBeenCalledWith('Hello');
  });

  it('should show error when provided', () => {
    render(
      <TextInput
        placeholder="Enter text"
        value=""
        onChangeText={jest.fn()}
        error="This field is required"
      />
    );

    expect(screen.getByText('This field is required')).toBeTruthy();
  });
});
```

**Screen Component Test**:

```typescript
// tests/screens/LoginScreen.test.tsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { Provider } from 'react-redux';
import { LoginScreen } from '@/screens/LoginScreen';
import { createStore } from '@/store/store';

describe('LoginScreen', () => {
  const store = createStore();

  it('should render login form', () => {
    render(
      <Provider store={store}>
        <LoginScreen />
      </Provider>
    );

    expect(screen.getByPlaceholderText(/phone number/i)).toBeTruthy();
    expect(screen.getByPlaceholderText(/password/i)).toBeTruthy();
  });

  it('should submit form with valid data', async () => {
    const mockNavigate = jest.fn();

    render(
      <Provider store={store}>
        <LoginScreen navigation={{ navigate: mockNavigate }} />
      </Provider>
    );

    const phoneInput = screen.getByPlaceholderText(/phone number/i);
    const passwordInput = screen.getByPlaceholderText(/password/i);
    const submitButton = screen.getByText(/login/i);

    fireEvent.changeText(phoneInput, '+225XXXXXXXXXX');
    fireEvent.changeText(passwordInput, 'SecurePass123!');
    fireEvent.press(submitButton);

    await waitFor(() => {
      expect(mockNavigate).toHaveBeenCalledWith('Dashboard');
    });
  });

  it('should show validation error for invalid phone', async () => {
    render(
      <Provider store={store}>
        <LoginScreen />
      </Provider>
    );

    const phoneInput = screen.getByPlaceholderText(/phone number/i);
    const submitButton = screen.getByText(/login/i);

    fireEvent.changeText(phoneInput, 'invalid');
    fireEvent.press(submitButton);

    await waitFor(() => {
      expect(screen.getByText(/invalid phone/i)).toBeTruthy();
    });
  });
});
```

**Coverage Target**: 80%+

**Timeline**: 2-3 weeks

---

## 2. Backend API Tests 🔌

### 2.1 Setup API Testing

**Install Dependencies**:

```bash
npm install --save-dev supertest @types/supertest jest ts-jest
```

**Jest Config for Backend** (`jest.backend.config.js`):

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['tests/**/*.test.ts'],
  collectCoverageFrom: ['backend/**/*.ts'],
};
```

### 2.2 API Tests

**Authentication Routes Test**:

```typescript
// tests/api/auth.test.ts
import request from 'supertest';
import app from '@/backend/server';
import { db } from '@/backend/config/database';

describe('Authentication Routes', () => {
  beforeAll(async () => {
    // Setup test database
    await db.migrate.latest();
  });

  afterAll(async () => {
    // Cleanup test database
    await db.destroy();
  });

  describe('POST /auth/register', () => {
    it('should register new user', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          phone_number: '+225XXXXXXXXXX',
          email: 'test@example.com',
          password: 'SecurePass123!',
          full_name: 'Test User',
        });

      expect(response.status).toBe(201);
      expect(response.body.user.phone_number).toBe('+225XXXXXXXXXX');
      expect(response.body.token).toBeTruthy();
    });

    it('should fail with duplicate phone number', async () => {
      // First registration
      await request(app)
        .post('/api/auth/register')
        .send({
          phone_number: '+225XXXXXXXXXX',
          email: 'test1@example.com',
          password: 'SecurePass123!',
          full_name: 'Test User 1',
        });

      // Duplicate attempt
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          phone_number: '+225XXXXXXXXXX',
          email: 'test2@example.com',
          password: 'SecurePass123!',
          full_name: 'Test User 2',
        });

      expect(response.status).toBe(409);
      expect(response.body.error.code).toBe('DUPLICATE_PHONE');
    });

    it('should validate password strength', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          phone_number: '+225XXXXXXXXXX',
          email: 'test@example.com',
          password: 'weak',
          full_name: 'Test User',
        });

      expect(response.status).toBe(400);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });
  });

  describe('POST /auth/login', () => {
    beforeEach(async () => {
      await request(app)
        .post('/api/auth/register')
        .send({
          phone_number: '+225XXXXXXXXXX',
          email: 'test@example.com',
          password: 'SecurePass123!',
          full_name: 'Test User',
        });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          phone_number: '+225XXXXXXXXXX',
          password: 'SecurePass123!',
        });

      expect(response.status).toBe(200);
      expect(response.body.token).toBeTruthy();
      expect(response.body.user.phone_number).toBe('+225XXXXXXXXXX');
    });

    it('should fail with invalid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          phone_number: '+225XXXXXXXXXX',
          password: 'WrongPassword123!',
        });

      expect(response.status).toBe(401);
      expect(response.body.error.code).toBe('INVALID_CREDENTIALS');
    });
  });

  describe('GET /auth/profile', () => {
    let token: string;

    beforeEach(async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          phone_number: '+225XXXXXXXXXX',
          password: 'SecurePass123!',
        });
      token = response.body.token;
    });

    it('should get authenticated user profile', async () => {
      const response = await request(app)
        .get('/api/auth/profile')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(200);
      expect(response.body.phone_number).toBe('+225XXXXXXXXXX');
    });

    it('should fail without token', async () => {
      const response = await request(app).get('/api/auth/profile');

      expect(response.status).toBe(401);
    });
  });
});
```

**Transaction Routes Test**:

```typescript
// tests/api/transactions.test.ts
import request from 'supertest';
import app from '@/backend/server';

describe('Transaction Routes', () => {
  let authToken: string;
  let userId: string;

  beforeEach(async () => {
    // Register and login
    const regRes = await request(app)
      .post('/api/auth/register')
      .send({
        phone_number: '+225XXXXXXXXXX',
        email: 'test@example.com',
        password: 'SecurePass123!',
        full_name: 'Test User',
      });

    userId = regRes.body.user.id;

    const loginRes = await request(app)
      .post('/api/auth/login')
      .send({
        phone_number: '+225XXXXXXXXXX',
        password: 'SecurePass123!',
      });

    authToken = loginRes.body.token;
  });

  describe('POST /transactions/send', () => {
    it('should send money successfully', async () => {
      const response = await request(app)
        .post('/api/transactions/send')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          recipient_phone: '+225YYYYYYYYYY',
          amount: 5000,
          provider: 'orange_money',
          description: 'Test transfer',
        });

      expect(response.status).toBe(201);
      expect(response.body.transaction.status).toBe('pending');
      expect(response.body.transaction.amount).toBe(5000);
    });

    it('should validate recipient phone number', async () => {
      const response = await request(app)
        .post('/api/transactions/send')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          recipient_phone: 'invalid',
          amount: 5000,
          provider: 'orange_money',
        });

      expect(response.status).toBe(400);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('should check sufficient balance', async () => {
      const response = await request(app)
        .post('/api/transactions/send')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          recipient_phone: '+225YYYYYYYYYY',
          amount: 999999999,
          provider: 'orange_money',
        });

      expect(response.status).toBe(400);
      expect(response.body.error.code).toBe('INSUFFICIENT_BALANCE');
    });
  });

  describe('GET /transactions', () => {
    it('should get transaction history', async () => {
      const response = await request(app)
        .get('/api/transactions')
        .set('Authorization', `Bearer ${authToken}`);

      expect(response.status).toBe(200);
      expect(Array.isArray(response.body.transactions)).toBe(true);
    });

    it('should paginate results', async () => {
      const response = await request(app)
        .get('/api/transactions?limit=10&offset=0')
        .set('Authorization', `Bearer ${authToken}`);

      expect(response.status).toBe(200);
      expect(response.body.limit).toBe(10);
      expect(response.body.offset).toBe(0);
    });
  });
});
```

**Coverage Target**: 85%+

**Timeline**: 2-3 weeks

---

## 3. Service Tests 🔧

**Payment Service Test**:

```typescript
// tests/services/paymentService.test.ts
import { paymentService } from '@/services/paymentService';
import { ProviderFactory } from '@/services/providers/ProviderFactory';

jest.mock('@/services/providers/ProviderFactory');

describe('Payment Service', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should process payment successfully', async () => {
    const mockProvider = {
      process: jest.fn().mockResolvedValue({
        id: 'txn_123',
        status: 'completed',
      }),
    };

    (ProviderFactory.getProvider as jest.Mock).mockReturnValue(
      mockProvider
    );

    const result = await paymentService.processPayment({
      amount: 5000,
      provider: 'orange_money',
      phoneNumber: '+225XXXXXXXXXX',
      userId: 'user_123',
    });

    expect(result.status).toBe('completed');
    expect(mockProvider.process).toHaveBeenCalled();
  });

  it('should calculate correct fee', async () => {
    const fee = paymentService.calculateFee(10000, 'orange_money');
    expect(fee).toBe(150); // 1.5% of 10000
  });

  it('should validate phone number', () => {
    expect(
      paymentService.validatePhoneNumber('+225XXXXXXXXXX')
    ).toBe(true);
    expect(
      paymentService.validatePhoneNumber('invalid')
    ).toBe(false);
  });
});
```

**Auth Service Test**:

```typescript
// tests/services/authService.test.ts
import { authService } from '@/services/authService';
import * as jwt from 'jsonwebtoken';

jest.mock('jsonwebtoken');

describe('Auth Service', () => {
  it('should generate valid token', () => {
    const token = authService.generateToken({ userId: '123' });

    expect(token).toBeTruthy();
    expect((jwt.sign as jest.Mock)).toHaveBeenCalled();
  });

  it('should verify token', () => {
    const payload = { userId: '123' };
    const token = 'valid_token';

    (jwt.verify as jest.Mock).mockReturnValue(payload);

    const result = authService.verifyToken(token);

    expect(result).toEqual(payload);
  });

  it('should hash password', async () => {
    const password = 'SecurePass123!';
    const hash = await authService.hashPassword(password);

    expect(hash).not.toBe(password);
    expect(hash.length).toBeGreaterThan(0);
  });

  it('should verify password', async () => {
    const password = 'SecurePass123!';
    const hash = await authService.hashPassword(password);

    const match = await authService.verifyPassword(
      password,
      hash
    );

    expect(match).toBe(true);
  });
});
```

**Coverage Target**: 80%+

**Timeline**: 2 weeks

---

## 4. State Management Tests 📊

```typescript
// tests/store/authSlice.test.ts
import authReducer, {
  setUser,
  clearAuth,
} from '@/store/authSlice';

describe('Auth Slice', () => {
  const initialState = {
    user: null,
    token: null,
    isLoading: false,
    error: null,
  };

  it('should return initial state', () => {
    expect(authReducer(undefined, { type: '' })).toEqual(
      initialState
    );
  });

  it('should handle setUser', () => {
    const user = {
      id: '123',
      name: 'John Doe',
      email: 'john@example.com',
    };

    const state = authReducer(
      initialState,
      setUser(user)
    );

    expect(state.user).toEqual(user);
  });

  it('should handle clearAuth', () => {
    const currentState = {
      ...initialState,
      user: { id: '123', name: 'John' },
      token: 'abc123',
    };

    const state = authReducer(currentState, clearAuth());

    expect(state.user).toBeNull();
    expect(state.token).toBeNull();
  });
});
```

**Coverage Target**: 90%+

**Timeline**: 1-2 weeks

---

## 5. E2E Tests 🎯

**Setup Cypress**:

```bash
npm install --save-dev cypress cypress-react-selector
npx cypress open
```

**E2E Test Example**:

```typescript
// cypress/e2e/auth.cy.ts
describe('Authentication Flow', () => {
  beforeEach(() => {
    cy.visit('/');
  });

  it('should register new user', () => {
    // Click sign up
    cy.contains('Sign Up').click();

    // Fill form
    cy.get('[data-testid=phone-input]').type(
      '+225XXXXXXXXXX'
    );
    cy.get('[data-testid=email-input]').type(
      'test@example.com'
    );
    cy.get('[data-testid=password-input]').type(
      'SecurePass123!'
    );
    cy.get('[data-testid=name-input]').type('Test User');

    // Submit
    cy.get('[data-testid=register-button]').click();

    // Verify redirect to dashboard
    cy.url().should('include', '/dashboard');
    cy.contains('Welcome, Test User').should('be.visible');
  });

  it('should complete full transfer workflow', () => {
    // Login
    cy.login('+225XXXXXXXXXX', 'SecurePass123!');

    // Navigate to send money
    cy.contains('Send Money').click();

    // Fill transfer form
    cy.get('[data-testid=recipient-input]').type(
      '+225YYYYYYYYYY'
    );
    cy.get('[data-testid=amount-input]').type('5000');
    cy.get('[data-testid=provider-select]').select(
      'orange_money'
    );

    // Submit
    cy.get('[data-testid=send-button]').click();

    // Verify success message
    cy.contains('Transfer successful').should('be.visible');
  });
});
```

**Coverage Target**: All critical user flows

**Timeline**: 2-3 weeks

---

## 6. Performance Tests ⚡

```typescript
// tests/performance/api.test.ts
describe('API Performance', () => {
  it('should return list of transactions within 200ms', async () => {
    const start = performance.now();

    const response = await fetch('/api/transactions');

    const end = performance.now();
    const duration = end - start;

    expect(duration).toBeLessThan(200);
    expect(response.status).toBe(200);
  });

  it('should process payment within 1000ms', async () => {
    const start = performance.now();

    const response = await fetch('/api/transactions/send', {
      method: 'POST',
      body: JSON.stringify({
        amount: 5000,
        provider: 'orange_money',
        recipient_phone: '+225XXXXXXXXXX',
      }),
    });

    const end = performance.now();
    const duration = end - start;

    expect(duration).toBeLessThan(1000);
    expect(response.status).toBe(201);
  });
});
```

**Timeline**: 1-2 weeks

---

## Test Execution Scripts

**Update `package.json`**:

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:api": "jest --config jest.backend.config.js",
    "test:e2e": "cypress run",
    "test:e2e:open": "cypress open",
    "test:performance": "jest --testPathPattern=performance"
  }
}
```

---

## Phase 4 Summary

| Task | Timeline | Coverage |
|------|----------|----------|
| Component Tests | 2-3 wks | 80%+ |
| API Tests | 2-3 wks | 85%+ |
| Service Tests | 2 wks | 80%+ |
| State Tests | 1-2 wks | 90%+ |
| E2E Tests | 2-3 wks | Critical flows |
| Performance Tests | 1-2 wks | <200ms APIs |

**Total Phase 4**: 10-14 weeks

---

## Success Metrics

✅ **Coverage**:
- Overall: 85%+
- Components: 80%+
- Services: 80%+
- API: 85%+
- State: 90%+

✅ **Quality**:
- All critical flows tested
- Performance benchmarks met
- No flaky tests
- CI/CD tests pass

---

**Ready for comprehensive testing!** 🧪
