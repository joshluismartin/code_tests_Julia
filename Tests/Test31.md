# Test 31 — Frontend Testing & Tooling

**Rules:** Don't look at Lesson31.md while answering. Write your answers under each question. Then check `Answers31.md`.
**Total:** 39 points

---

## Part A — Multiple choice (1 point each)

**1.** What is Vitest?
- a) A UI library
- b) The frontend test runner
- c) A bundler
- d) A linter

**2.** What does `environment: 'happy-dom'` in `vitest.config.mts` do?
- a) Makes tests run in Chrome
- b) Gives tests a fake browser DOM inside Node
- c) Enables Storybook
- d) Turns off TypeScript

**3.** What is a **mock**?
- a) A failing test
- b) A fake replacement for a real function or module
- c) A Storybook story
- d) A snapshot file

**4.** Which matcher should you use to compare two objects by content?
- a) `toBe`
- b) `toEqual`
- c) `toHaveBeenCalled`
- d) `toContain`

**5.** Which Testing Library query finds an element the way a screen reader would?
- a) `document.getElementById`
- b) `screen.getByRole`
- c) `querySelector('.btn')`
- d) `screen.getByClassName`

**6.** In CSF (Component Story Format), what is each **named export**?
- a) A test
- b) One story (one state of the component)
- c) A route
- d) A Redux action

**7.** Why don't you write individual snapshot tests in freee-payroll?
- a) Snapshots are banned
- b) Each feature's `snapshots.test.ts` runs `runSnapshotTests(__dirname)`, turning every story into a snapshot automatically
- c) Snapshots are only for Ruby
- d) Prettier creates them

**8.** Which CI command detects a circular import?
- a) `yarn run tsc`
- b) `npx prettier -c .`
- c) `yarn run depcruise front`
- d) `yarn run vitest run`

**9.** What does `npx prettier -c .` do?
- a) Compiles TypeScript
- b) Checks formatting without changing files
- c) Runs tests
- d) Creates a new project

**10.** Why do many tests contain `import '../../../../.storybook/globalProps';`?
- a) To start Storybook
- b) To fill `window.$FREEE_DATA` with realistic fake data (side-effect import)
- c) To import a component
- d) To enable ESLint

---

## Part B — Fill in the blanks (1 point each blank)

**11.** In Vitest, the equivalents of RSpec's `before` and `after` are ______ and ______.

**12.** `vi.______()` creates a fake function; `vi.______('module-name', () => ({...}))` replaces a whole module.

**13.** To assert a fake was called with `2016, 7`, write `expect(onChange).______(2016, 7)`.

**14.** The old Storybook API `______` is banned by ESLint; write CSF instead.

**15.** In a story, fake callbacks are created with `______('onSave')` from `@storybook/addon-actions`.

**16.** To run a hook in a test, use `______(() => useMyHook())` and read its return value from `result.______`.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the story file for `LastName` (props: `value: string`, `onChange`, optional `disabled`):

```tsx
import { action } from '@storybook/addon-actions';
import * as React from 'react';

import LastName from './LastName';

export ______ {
  component: ______,
};

export const Default = () => (
  <LastName onChange={______('onChange')} value="給与" />
);

export const Disabled = () => (
  <LastName disabled onChange={action('onChange')} value="給与" />
);
```

**18.** Complete this test so it restores the flag after each test:

```ts
describe('buildLink', () => {
  const originalFlag = $FREEE_DATA.ff.new_link_enabled;

  ______(() => {
    $FREEE_DATA.ff.new_link_enabled = ______;
  });

  it('returns the new URL when the flag is on', () => {
    $FREEE_DATA.ff.new_link_enabled = true;
    expect(buildLink()).______({ url: '/new' });
  });
});
```

---

## Part D — Write the code (3 points each)

**19.** Write a Vitest + Testing Library test for this component (illustrative):

```tsx
type Props = { onSave: (memo: string) => void };
export const MemoForm = ({ onSave }: Props) => {
  const [memo, setMemo] = React.useState('');
  return (
    <>
      <input aria-label="メモ" onChange={(e) => setMemo(e.currentTarget.value)} value={memo} />
      <button onClick={() => onSave(memo)} type="button">保存</button>
    </>
  );
};
```

The test should: render it with a `vi.fn()` for `onSave`, type `"hello"` into the input (use `fireEvent.change(input, { target: { value: 'hello' } })`), click the 保存 button (find it with `getByRole`), and assert `onSave` was called with `'hello'`. Include imports and `afterEach(cleanup)`.

**20.** Write a CSF story file for `RiskChip` (illustrative) with props `level: 'HIGH' | 'STABLE'` and `onClick: () => void`, containing two stories: `High` and `Stable`.

---

## Part E — Answer in a sentence or two (2 points each)

**21.** List the frontend CI checks in the `front_test` job and say in a few words what each catches.

**22.** Why is the Presentational/Container split helpful for Storybook and snapshot testing?

**23.** A test sets `$FREEE_DATA.ff.x = true`. Why must it restore the value afterwards, and why is changing `$FREEE_DATA` inside a *story* forbidden?

---

## Part F — Read real repo code (4 points)

**24.** From `front/javascripts/company_social_insurance_settings/components/common/ApplyYearMonthSection.test.tsx`:

```tsx
  it('選択肢を選ぶと onChange がその年月で呼ばれる', () => {
    const onChange = vi.fn();
    renderSection(2016, 6, onChange);

    fireEvent.click(screen.getByText('2016年6月'));

    fireEvent.click(screen.getByRole('button', { name: '2016年7月' }));

    expect(onChange).toHaveBeenCalledWith(2016, 7);
  });
```

a) What is `onChange` here, and why is it passed to `renderSection`? (1 pt)
b) Describe the two user actions the test simulates. (1 pt)
c) What exactly does the final line prove? (1 pt)
d) Does this test check the component's internal state? Why is that good? (1 pt)
