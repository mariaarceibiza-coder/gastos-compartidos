# Test Coverage Analysis — Gastos Compartidos

## Current State

**Test coverage: 0%.** There are no tests, no testing framework, no test configuration, and no `package.json` at all. The entire application lives in a single `index.html` file that loads React, Babel, and Firebase from CDNs.

---

## Prerequisites: Project Restructuring

Before any tests can be written, the project needs a minimal build/test infrastructure:

1. **Initialize `package.json`** with `npm init`
2. **Install a testing framework** — Vitest is recommended (fast, native ESM, works well with React)
3. **Install React Testing Library** for component tests
4. **Extract business logic** from the monolithic `index.html` into importable modules

Suggested directory structure:

```
gastos-compartidos/
├── src/
│   ├── App.jsx                  # Main React component
│   ├── calculations.js          # Pure budget calculation functions
│   ├── firebase.js              # Firebase initialization & data layer
│   └── components/
│       ├── BudgetStatus.jsx
│       ├── FixedExpensesList.jsx
│       ├── ExpenseForm.jsx
│       └── HistoricoModal.jsx
├── tests/
│   ├── calculations.test.js
│   ├── App.test.jsx
│   └── components/
│       ├── BudgetStatus.test.jsx
│       ├── FixedExpensesList.test.jsx
│       ├── ExpenseForm.test.jsx
│       └── HistoricoModal.test.jsx
├── index.html
├── package.json
└── vitest.config.js
```

---

## Areas Requiring Test Coverage

### 1. Budget Calculation Logic (HIGH PRIORITY)

This is the most critical area. All the financial math is currently inline inside the component with no validation. These are pure functions that can be extracted and tested in isolation with no DOM or Firebase dependency.

| Calculation | Current Code (index.html) | What to Test |
|---|---|---|
| Total fixed expenses | `gastosFijos.reduce((sum, g) => sum + g.presupuesto, 0)` | Sum with decimals, empty array, single item |
| Paid vs. pending fixed expenses | `.filter(g => g.pagado).reduce(...)` | Mixed paid/unpaid, all paid, none paid |
| Remaining budget (`resto`) | `ingresos - totalGastosFijos` | Positive, zero, negative remainder |
| Days in month | `new Date(año, mes + 1, 0).getDate()` | Feb in leap year, 30-day months, 31-day months |
| Daily budget | `resto / diasMes` | Normal case, zero `resto`, negative `resto` |
| Total variable expenses | `gastosVariables.reduce((sum, g) => sum + g.monto, 0)` | Decimals, empty, large lists |
| Average daily spending | `totalGastosVariables / diaActual` | Division by zero (day 0), normal days |
| End-of-month projection | `totalGastosVariables + (gastoPromedioDiario * diasRestantes)` | First day, last day, mid-month |
| Projected surplus/deficit | `resto - proyeccionFinMes` | Surplus, exact zero, deficit |
| Budget status classification | `diferenciaPrevista >= 0 ? 'bien' : >= -100 ? 'ajustado' : 'peligro'` | Boundary values at 0, -1, -99, -100, -101 |
| Spending percentage | `(totalGastosVariables / resto) * 100` | 0%, 50%, 100%, >100%, division by zero when `resto` is 0 |

**Example test cases for `calculations.js`:**

```js
describe('calculateBudgetStatus', () => {
  it('returns "bien" when projected surplus is positive', () => {
    expect(calculateBudgetStatus(500)).toBe('bien');
  });

  it('returns "bien" when projected surplus is exactly 0', () => {
    expect(calculateBudgetStatus(0)).toBe('bien');
  });

  it('returns "ajustado" when deficit is between -1 and -100', () => {
    expect(calculateBudgetStatus(-50)).toBe('ajustado');
  });

  it('returns "peligro" when deficit exceeds -100', () => {
    expect(calculateBudgetStatus(-101)).toBe('peligro');
  });

  // Edge: boundary at exactly -100
  it('returns "ajustado" at the -100 boundary', () => {
    expect(calculateBudgetStatus(-100)).toBe('ajustado');
  });
});
```

### 2. Expense Management Operations (HIGH PRIORITY)

These are the core CRUD operations for both fixed and variable expenses.

**Variable expense addition:**
- Adding a valid expense (with date and amount) appends it to the list
- Adding without a date is rejected (no-op)
- Adding without an amount is rejected (no-op)
- Amount is parsed as float from string input
- Description defaults to empty string when omitted
- New expense gets a unique `id` via `Date.now()`

**Fixed expense operations:**
- Adding a new fixed expense generates a correct ID (`Math.max(...ids) + 1`)
- Adding when the list is empty (edge case: `Math.max()` of empty spread)
- Editing a fixed expense's category updates correctly
- Editing a fixed expense's amount updates correctly
- Deleting a fixed expense removes only that item
- Toggling `pagado` checkbox flips only the targeted item

**Example edge case bugs to test for:**
```js
// Current code: Math.max(...gastosFijos.map(g => g.id), 0) + 1
// If gastosFijos is empty, Math.max(...[], 0) returns 0, so newId = 1. ✓
// But what if IDs are non-sequential (e.g., [1, 5, 3])? Max is 5, newId = 6. ✓
// This is fine, but should be tested.
```

### 3. Month/Year Navigation and Historical Data (MEDIUM PRIORITY)

- Changing month updates `mesKey` correctly (format: `YYYY-M`)
- Loading a historical month parses the key and sets state
- Historical months are sorted in reverse chronological order
- Variable expenses load from the correct month's historical data
- Switching to a month with no data shows empty variable expenses

**Specific edge cases:**
- December → January year transition
- Month key format consistency (`2025-0` for January vs potential `2025-00`)
- Historical data integrity when saving/loading

### 4. Firebase Data Synchronization (MEDIUM PRIORITY)

These tests require mocking Firebase Firestore.

- `onSnapshot` listener initializes state from Firestore document
- When Firestore document doesn't exist, default values are used
- Data is saved to Firestore when `ingresos`, `gastosFijos`, `gastosVariables`, or `saldoAhorro` change
- Historical data structure is built correctly on save
- The `synced` indicator shows briefly after sync and disappears
- Loading state is `true` initially and becomes `false` after first snapshot

**What to mock:**
```js
// Mock firebase.firestore()
const mockFirestore = {
  collection: vi.fn().mockReturnThis(),
  doc: vi.fn().mockReturnThis(),
  set: vi.fn().mockResolvedValue(),
  onSnapshot: vi.fn((callback) => {
    callback({ exists: true, data: () => mockData });
    return vi.fn(); // unsubscribe
  }),
};
```

### 5. UI Rendering and Conditional Display (LOWER PRIORITY)

Using React Testing Library:

- Loading spinner shows when `loading` is `true`
- Budget alert shows correct color/message for each `estadoProyeccion`
- Progress bar width matches `porcentajeGastado` (capped at 100%)
- Progress bar color: green (<70%), orange (70-90%), red (>90%)
- Fixed expense in edit mode shows input fields
- Fixed expense in view mode shows text and amounts
- Paid expenses have strikethrough styling (`.pagado` class)
- Historical modal opens/closes correctly
- Empty state message when no variable expenses exist

### 6. Input Validation and Edge Cases (LOWER PRIORITY)

- Income field: `parseFloat(value) || 0` — what happens with non-numeric input?
- Savings field: same `parseFloat` handling
- Expense amount with many decimal places
- Negative amounts (currently not prevented)
- Very large numbers (overflow in display?)
- Empty string inputs

---

## Bugs and Risks Discovered During Analysis

While analyzing the code for testability, the following issues were identified:

1. **Truncated source file**: `index.html` ends abruptly at line 381, mid-expression. The file appears incomplete — the closing tags for the component, script, body, and html are missing. This means the app as committed may not actually render.

2. **No input validation on expenses**: Negative amounts can be entered for both fixed and variable expenses. There's no max-length or range validation.

3. **Division by zero risk**: `porcentajeGastado = (totalGastosVariables / resto) * 100` — if `resto` is 0 (income equals fixed expenses), this produces `Infinity` or `NaN`.

4. **Division by zero on day 0**: `gastoPromedioDiario = diaActual > 0 ? totalGastosVariables / diaActual : 0` — this is handled, but `diasRestantes` could be negative if viewing a past month's data (since `diaActual` uses `new Date()` regardless of the selected month).

5. **`Date.now()` for IDs**: Variable expense IDs use `Date.now()`, which could collide if two expenses are added in the same millisecond.

6. **No Firebase error handling**: The `.set()` call has a `.then()` but no `.catch()`. Network failures are silently ignored.

7. **Stale closure in useEffect**: The second `useEffect` references `historico` from state and also calls `setHistorico`, which could cause infinite re-render loops depending on React's batching behavior.

---

## Recommended Testing Priority

| Priority | Area | Effort | Impact |
|----------|------|--------|--------|
| **P0** | Budget calculations (pure functions) | Low | High — financial correctness |
| **P0** | Expense CRUD operations | Low | High — core functionality |
| **P1** | Month navigation & historical data | Medium | Medium — data integrity |
| **P1** | Firebase sync (with mocks) | Medium | Medium — data persistence |
| **P2** | UI conditional rendering | Medium | Lower — visual correctness |
| **P2** | Input validation edge cases | Low | Lower — robustness |

---

## Summary

The project has **zero test coverage** and requires foundational work (package.json, module extraction, test framework setup) before any tests can be written. The highest-value tests are for the **budget calculation logic** — these are pure functions that can be extracted and tested trivially, and errors there directly affect financial accuracy. The second priority is **expense CRUD operations**, which are the core user interactions. Firebase integration and UI rendering tests provide additional safety but require more setup (mocking, React Testing Library).
