# TypeScript / JavaScript

Language-specific conventions that extend the core rules in `SKILL.md`.

## React Import Order

After 3rd-party imports, order local imports: **context → hooks → rest**.

```typescript
import { Box, Typography } from '@mui/material';
import { useState } from 'react';

import { AppContext } from 'context/app';
import { ThemeContext } from 'context/theme';

import { useOptimisticState } from 'hooks/useOptimisticState';
import { useRemainingTimer } from 'hooks/useRemainingTimer';

import { formatLeaves } from 'modules/leaves';
import { AppLeaf } from 'types/common/leaf';
```

## Clean Types

Enforce immutability through patterns, not type annotations. Do not use `readonly` — keep type definitions clean and minimal.

```typescript
// Do
type User = {
  name: string;
  email: string;
};

// Don't
type User = {
  readonly name: string;
  readonly email: string;
};
```

## Interfaces vs Types

- Use `interface` for React component props
- Use `type` for everything else

```typescript
// Do — interface for component props
interface UserCardProps {
  name: string;
  email: string;
}

// Do — type for non-component definitions
type Status = 'active' | 'inactive';
type UserMap = Record<string, User>;

// Don't — type for component props
type UserCardProps = {
  name: string;
  email: string;
};
```

## Define Before Usage

Define interfaces and types **before** their first usage in the file, unless they are global/shared (in which case they live in a dedicated types file).

```typescript
// Do — definition first, then usage
interface UserCardProps {
  name: string;
  email: string;
}

const UserCard = ({ name, email }: UserCardProps) => (
  <div>{name} — {email}</div>
);

// Don't — usage before definition
const UserCard = ({ name, email }: UserCardProps) => (
  <div>{name} — {email}</div>
);

interface UserCardProps {
  name: string;
  email: string;
}
```

## MUI: Inline sx Props

Keep `sx` props inline. Do not extract them into variables or separate style objects.

```tsx
// Do
<Box sx={{ display: 'flex', gap: 2, p: 1 }}>
  <Typography sx={{ fontWeight: 'bold', color: 'text.secondary' }}>
    Hello
  </Typography>
</Box>

// Don't
const boxStyles = { display: 'flex', gap: 2, p: 1 };
const typographyStyles = { fontWeight: 'bold', color: 'text.secondary' };

<Box sx={boxStyles}>
  <Typography sx={typographyStyles}>Hello</Typography>
</Box>
```

## Unit Testing

Use the file name as the top-level `describe` block for each test file.

```typescript
// File: calculateTotal.test.ts
describe('calculateTotal', () => {
  it('should sum item prices', () => {
    expect(calculateTotal([{ price: 10 }, { price: 20 }])).toBe(30);
  });

  it('should return 0 for empty list', () => {
    expect(calculateTotal([])).toBe(0);
  });

  describe('with discounts', () => {
    it('should apply discount to total', () => {
      expect(calculateTotal([{ price: 100 }], 0.1)).toBe(90);
    });
  });
});
```
