# Test 26 — React 1: Components, JSX, Props

**Rules:** Don't look at Lesson26.md while answering. Write your answers under each question. Then check `Answers26.md`.
**Total:** 38 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a React *component*?
- a) A CSS file
- b) A function (Capitalized name) that returns what should appear on screen
- c) A database table
- d) A Rails controller

**2.** What are *props*?
- a) The component's private memory that it can change
- b) Inputs passed from a parent component to a child, read-only in the child
- c) Global variables
- d) CSS properties

**3.** What does "one-way data flow" mean in React?
- a) Data can only be sent to the server, never received
- b) Data flows down from parent to child through props; children notify parents via callback functions
- c) Only one component may exist per page
- d) State can only be set once

**4.** Why does React need a `key` on list items?
- a) To encrypt the list
- b) To sort the list alphabetically
- c) To give each item a stable identity so React updates the right element when the list changes
- d) To make the item clickable

**5.** What is the *virtual DOM* idea?
- a) React keeps an in-memory copy of the UI, compares new vs old, and updates only what changed in the real DOM
- b) A second browser window
- c) A database of HTML files
- d) A fake page used only in tests

**6.** A file containing JSX in freee-payroll usually has which extension?
- a) `.ts`
- b) `.tsx`
- c) `.jsx.erb`
- d) `.rb`

**7.** In freee-payroll, which export of a file like `JoiningEmployeesPanel.tsx` calls the API?
- a) `export default JoiningEmployeesPanel`
- b) `export const JoiningEmployeesPanelContainer`
- c) Neither; API calls are only in Ruby
- d) The `Props` type

**8.** What is `data-test="姓"` used for in freee-payroll components?
- a) Styling
- b) Translating text
- c) A marker so automated UI tests can find the element
- d) Sending data to the server

---

## Part B — Fill in the blanks (1 point each)

**9.** Inside JSX, curly braces `{ }` let you insert any JavaScript ______ (not a statement).

**10.** To group several elements without adding an extra DOM element, wrap them in a ______, written `<>...</>`.

**11.** A prop written with no value, like `<TaskDialog isOpen />`, means `isOpen={______}`.

**12.** Event-handler props in React are named like `on______` (e.g. for a click).

**13.** freee-payroll's ESLint rule `react/jsx-sort-props` makes JSX props appear in ______ order.

**14.** The special prop that holds whatever you put between a component's opening and closing tags is called ______.

---

## Part C — Fill in the missing code (2 points each)

**15.** Render one `<li>` per employee with a proper key.
```tsx
<ul>
  {employees.______((employee) => (
    <li ______={______}>{employee.name}</li>
  ))}
</ul>
```

**16.** Show `<NewBadge />` only when `isNew` is true, and show `<Text>0件</Text>` or `<List />` depending on whether `items.length === 0`.
```tsx
<>
  {isNew ______ <NewBadge />}
  {items.length === 0 ______ <Text>0件</Text> ______ <List items={items} />}
</>
```

**17.** Fix the bug: the handler runs immediately during render instead of on click.
```tsx
<Button onClick={onSave(trimmed)}>保存</Button>
```
Corrected: `<Button onClick={______}>保存</Button>`

---

## Part D — Write the code (4 points each)

**18.** Write a presentational component `RetiredEmployeeCount` with a `Props` type `{ count: number }` that renders `<p>退職者: {count}名</p>`, but renders **nothing** when `count` is `0`. Use `export default`. Be careful of the `0` trap.

**19.** Write a component `AssigneeList` with props `{ assignees: Array<{ id: string; name: string }>; onRemove: (id: string) => void }`. It renders a `<ul>`; each `<li>` shows the name and a `<button>` "削除" that calls `onRemove` with that assignee's id. Use correct keys.

---

## Part E — Answer in a sentence or two (2 points each)

**20.** Why is `{count && <Panel />}` risky when `count` is a number? How do you fix it?

**21.** Why does freee-payroll split many components into a default-exported presentational component and a `XxxContainer`? Give one benefit.

**22.** Why do many `.tsx` files in freee-payroll write `import * as React from 'react';` even when the code doesn't mention `React` directly?

---

## Part F — Read real repo code (4 points)

**23.** From `front/javascripts/employees/components/Employee/MemberInfoCard.tsx:214-245`:
```tsx
      {showActionButtons &&
        (isInvited ? (
          <HStack alignItems="center" gap={0.5}>
            <Button ... onClick={onCancelInvitationClick}>招待解除</Button>
            {account.invitationStatus === 'invited' && (
              <Button ... onClick={onResendInvitationClick}>招待再送</Button>
            )}
          </HStack>
        ) : (
          <Button ... onClick={onInviteClick}>招待</Button>
        ))}
```
List which buttons appear in each of these cases:
- a) `showActionButtons = false`
- b) `showActionButtons = true`, `isInvited = false`
- c) `showActionButtons = true`, `isInvited = true`, `invitationStatus = 'invited'`
- d) `showActionButtons = true`, `isInvited = true`, `invitationStatus = 'expired'`
