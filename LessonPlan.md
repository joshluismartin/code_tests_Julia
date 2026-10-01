# LessonPlan.md — freee-payroll Crash Course

Tick each box (`[ ]` → `[x]`) as you finish it.
Each lesson is about **1 hour**: ~40 min reading + practice, ~20 min test.
For every lesson: read `LessonN.md` → do the practice → take `TestN.md` → check `AnswersN.md`.

**Tip:** You don't have to do the parts strictly in order. After Part 1 (Foundations) you can alternate
between the Backend track (Parts 2–5) and the Frontend track (Parts 6–7), for example one Ruby day, then one React day.
Part 8 (Capstone) needs both tracks.

---

## Part 1 — Foundations (no coding knowledge needed)

- [ ] **Lesson 1 — How web apps work + a map of freee-payroll**
  - [ ] Terms: program, code, file, directory, repository, frontend, backend, server, client, browser
  - [ ] HTTP, URL, request/response, status codes, JSON, API
  - [ ] Database, table, row, column
  - [ ] Framework, library, package, dependency
  - [ ] The big picture: Browser (React/TypeScript) ⇄ Rails (Ruby) ⇄ MySQL + other freee services
  - [ ] Tour of the top-level folders: `app/`, `config/`, `db/`, `spec/`, `front/`, `lib/`, `docs/`
- [ ] **Lesson 2 — Your tools: terminal, Git, and reading a repo**
  - [ ] Terminal/shell commands: `cd`, `ls`, `cat`, `grep`, how to read a path
  - [ ] Git terms: commit, branch, merge, pull request (PR), `main`, trunk-based, gitmoji
  - [ ] Manifests: `Gemfile` / `Gemfile.lock`, `package.json` / `yarn.lock`, `.tool-versions`
  - [ ] Guidance files: `CLAUDE.md`, `README.md`, `docs/knowledge/`
  - [ ] Linters/formatters/type checkers/tests/CI: what each word means
  - [ ] How to search the codebase for a class or method

## Part 2 — Ruby (the language)

- [ ] **Lesson 3 — Ruby basics 1: values, variables, methods**
  - [ ] Integer, Float, String, Symbol, `nil`, `true`/`false`
  - [ ] Variables, `snake_case`, constants, comments, `puts`/`p`
  - [ ] String interpolation `"#{}"`, single vs double quotes (repo uses single)
  - [ ] Defining methods (`def`), arguments, implicit return, `?` and `!` method names
  - [ ] `# frozen_string_literal: true`
- [ ] **Lesson 4 — Ruby basics 2: decisions and collections**
  - [ ] `if / elsif / else / unless`, modifier `if` (`return if x`), ternary `a ? b : c`, `case/when`
  - [ ] Truthiness (only `nil` and `false` are falsy), `&&`, `||`, `!`
  - [ ] `&.` (safe navigation), `||=` (memoization)
  - [ ] Arrays, Hashes, symbol keys `{ id: 1 }`, ranges `1..12`, `date..`
  - [ ] Rails helpers `present?` / `blank?` / `presence`
- [ ] **Lesson 5 — Ruby basics 3: blocks and iteration**
  - [ ] Blocks `{ }` and `do ... end`, block parameters `|x|`
  - [ ] `each`, `map`, `select`, `reject`, `find`, `any?`, `all?`, `sum`, `count`
  - [ ] `group_by`, `index_by`, `each_slice`, `flat_map`, `each_with_object`
  - [ ] `&:method_name` shorthand, `yield`, lambdas `->(x) { }`
- [ ] **Lesson 6 — Ruby OOP 1: classes and objects**
  - [ ] Class, object/instance, `new`, `initialize`, instance variables `@x`
  - [ ] `attr_reader` / `attr_accessor`, instance methods, `self`
  - [ ] Class methods: `def self.x` and `class << self`
  - [ ] `private`, inheritance `<`, `super`
- [ ] **Lesson 7 — Ruby OOP 2: modules, namespaces, errors**
  - [ ] Modules as namespaces, `::`, `module A; module B; class C`
  - [ ] Mixins: `include` / `extend`; `ActiveSupport::Concern` preview
  - [ ] Keyword arguments `def x(company_id:, include_deleted: false)`, shorthand `f(x:)`, `**`
  - [ ] Exceptions: `raise`, `rescue`, `ensure`, custom error classes (`HrErrors::UserError`)
  - [ ] `freeze`, immutability
- [ ] **Lesson 8 — Sorbet: types for Ruby**
  - [ ] What a type / type checker is; static vs dynamic
  - [ ] Sigils `# typed: false / true / strict`
  - [ ] `extend T::Sig`, `sig { params(...).returns(...) }`, `.void`
  - [ ] `T.nilable`, `T::Array[]`, `T::Hash[]`, `T::Boolean`, `T.any`, `T.untyped`, `T.must`, `T.let`
  - [ ] `T::Struct` with `const` / `prop`

## Part 3 — Ruby on Rails

- [ ] **Lesson 9 — Rails overview: MVC and the request lifecycle**
  - [ ] Framework, MVC, "convention over configuration"
  - [ ] Directory conventions in `app/` and file-name ↔ class-name rule (Zeitwerk autoloading)
  - [ ] Environments: development / test / staging / production
  - [ ] Life of a request: route → controller → model → response
- [ ] **Lesson 10 — Databases, SQL and migrations**
  - [ ] Table, row, column, primary key, foreign key, index, NULL
  - [ ] Basic SQL: `SELECT`, `WHERE`, `JOIN`, `INSERT`, `UPDATE`, `DELETE`
  - [ ] Migrations (`db/migrate`), `db/schema.rb`
  - [ ] Primary DB vs read replica
- [ ] **Lesson 11 — ActiveRecord 1: models and queries**
  - [ ] ORM, `ApplicationRecord`, CRUD (`create`, `find`, `find_by`, `update`, `destroy`)
  - [ ] Query methods: `where`, `order`, `limit`, `pluck`, `exists?`, `count`
  - [ ] Relations are lazy; `.to_a`
  - [ ] `save` vs `save!` (bang methods)
- [ ] **Lesson 12 — ActiveRecord 2: associations, validations, callbacks, scopes**
  - [ ] `belongs_to`, `has_many`, `has_one`, `through:`, `class_name:`, `dependent:`
  - [ ] freee rule: `belongs_to` is optional by default
  - [ ] `validates`, custom `validate`, `errors.add`
  - [ ] Callbacks: `before_save`, `after_create`, `after_commit`
  - [ ] `scope`, `default_scope`, `unscope`
- [ ] **Lesson 13 — ActiveRecord 3: freee-specific model rules**
  - [ ] Soft delete: `deleted_at = DEFAULT_DELETED_AT` means "not deleted" (never `deleted_at: nil`)
  - [ ] Concerns (e.g. `CompanyRules::BankAccountValidations`), `enumerize`, `enum`, `aasm` state machines
  - [ ] LeakCheck and `Authz.without_employee_leakcheck`
  - [ ] N+1 queries, `includes`/`preload`, `find_each`, `pluck` (memory rules)
  - [ ] Transactions, `DatabaseSelector.connected_to_reader`
- [ ] **Lesson 14 — Routing and controllers**
  - [ ] `config/routes.rb`, `draw`, `namespace`, `scope`, `resources`, `member`/`collection`
  - [ ] REST actions: index/show/create/update/destroy ↔ GET/POST/PATCH/DELETE
  - [ ] Controllers, `params`, strong params (`permit`), `before_action`
  - [ ] `render json:`, HTTP status, `current_company` / `current_user`
- [ ] **Lesson 15 — Private API, OpenAPI and error handling**
  - [ ] `/api/p/...`, `Api::Private::BaseController`
  - [ ] OpenAPI/Swagger `*.swagger.yaml` next to the controller; schema-driven development
  - [ ] `committee` runtime validation; `make generate-root-swagger`
  - [ ] `ErrorHandleable`, `HrErrors::UserError`, serializers
  - [ ] Full trace: `EmployeeDisplayNamesController`
- [ ] **Lesson 16 — Views, helpers, and how Rails hands off to React**
  - [ ] ERB views, layouts, helpers
  - [ ] `JS_FILE_MAP` (controller#action → JS entrypoint)
  - [ ] `$FREEE_DATA` produced by `js_exports_helper.rb`
  - [ ] Feature flags: freee-feature-flag / sysparam / env vars, `$FREEE_DATA.ff`

## Part 4 — freee-payroll backend architecture

- [ ] **Lesson 17 — Rails-extended layers: Services, Repositories, Values, Presenters**
  - [ ] Why extra layers exist ("business logic", "separation of concerns")
  - [ ] Services: one public class method `execute` (policy spec)
  - [ ] Repositories: `find_all*` / `find_list_by_*` / `find_one_by_*` naming
  - [ ] Values (value objects) with `T::Struct` + `ActsAsValueObject`
  - [ ] Presenters, serializers, decorators; RuboCop cops that enforce the rules
- [ ] **Lesson 18 — Layered architecture in `app/modules`**
  - [ ] DDD terms: domain, ubiquitous language, entity, value object, use case, repository, factory
  - [ ] `application/` → `domain/` ← `infrastructure/` dependency direction
  - [ ] Namespaces as boundaries: `Dependency/OverBoundary`
  - [ ] Walk through one full module slice
- [ ] **Lesson 19 — Jobs, external services, and operations helpers**
  - [ ] Synchronous vs asynchronous, queues, ActiveJob (`perform_later`) vs legacy Resque
  - [ ] Microservices, gRPC/protobuf clients (`proto-*` gems), `lib/*_client.rb`
  - [ ] `Omega::Bugcap.notify`, `ReachableChecker`, Bugsnag, no `Rails.logger`
  - [ ] Encryption (`DatabaseColumnEncryptor`)

## Part 5 — Testing Ruby with RSpec

- [ ] **Lesson 20 — RSpec basics**
  - [ ] What automated tests are; unit vs request tests
  - [ ] `describe`, `context`, `it`, `expect(...).to eq`, `is_expected`
  - [ ] `let`, `let!`, `subject`, `before`, `described_class`
  - [ ] Matchers: `eq`, `be`, `change`, `raise_error`, `include`, `match`
- [ ] **Lesson 21 — RSpec in freee-payroll**
  - [ ] FactoryBot: `factory`, `trait`, `sequence`, `create`/`build`/`create_list`
  - [ ] `default_company` (never `create(:company)`), `log_in_as_an_admin_user`
  - [ ] Request specs for endpoints; controller specs only for base classes
  - [ ] Mocks/stubs: `allow(...).to receive`, `have_received`, `instance_double`, `stub_request`
  - [ ] `travel_to`, Data Driven Tests, `shared_examples`, spec/CLAUDE.md rules

## Part 6 — JavaScript & TypeScript

- [ ] **Lesson 22 — JavaScript basics 1**
  - [ ] `const` / `let`, primitive types, `null` vs `undefined`
  - [ ] Functions, arrow functions, template literals
  - [ ] Objects and arrays, destructuring, spread `...`
  - [ ] Modules: `import` / `export`, default vs named exports
- [ ] **Lesson 23 — JavaScript basics 2**
  - [ ] `===`, truthy/falsy, `&&`, `||`, `??`, `?.`
  - [ ] `map`, `filter`, `find`, `some`, `reduce`, `forEach`
  - [ ] Asynchronous code: Promise, `async`/`await`, `try`/`catch`
  - [ ] The DOM (preview), the browser console, `debugger`
- [ ] **Lesson 24 — TypeScript 1**
  - [ ] What TS adds to JS; `.ts` vs `.tsx`; `tsc` / type errors
  - [ ] Type annotations, `type` vs `interface`, optional `?`, `readonly`
  - [ ] Union types, string literal types, `any` vs `unknown`
  - [ ] Function types, return types
- [ ] **Lesson 25 — TypeScript 2**
  - [ ] Generics `<T>`, `Array<T>`, `Promise<T>`
  - [ ] Utility types: `Partial`, `Pick`, `Omit`, `Record`, `ReturnType`
  - [ ] Narrowing (`typeof`, `in`, discriminated unions), `as`, `as const`, `!`
  - [ ] `.d.ts` files, global types (`$FREEE_DATA`), `tsconfig.json`, zod

## Part 7 — React

- [ ] **Lesson 26 — React 1: components, JSX, props**
  - [ ] What React is; component; JSX; `.tsx`
  - [ ] Props and Props types; children
  - [ ] Lists with `map` and `key`; conditional rendering (`&&`, ternary)
  - [ ] Events (`onClick`, `onChange`)
- [ ] **Lesson 27 — React 2: state and effects**
  - [ ] State, re-render, `useState`, controlled inputs
  - [ ] `useEffect`, dependency arrays, cleanup
  - [ ] Rules of Hooks
  - [ ] Loading data on mount (with loading/error states)
- [ ] **Lesson 28 — React 3: more hooks and shared state**
  - [ ] `useMemo`, `useCallback`, `useRef`
  - [ ] Custom hooks (`useXxx`)
  - [ ] Context (`createContext`, Provider, `useContext`), `useReducer`
  - [ ] Redux + ducks pattern (`@freee-payroll/ducks`)
- [ ] **Lesson 29 — React in freee-payroll 1: structure & UI**
  - [ ] `front/` layout, feature directories, entrypoints (`jsEntries.js`), Vite
  - [ ] No cross-feature imports; `@freee-payroll/*` packages; Yarn workspaces
  - [ ] Vibes (`@c-fo/vibes`) components, styled-components, a11y rule for FormControl
  - [ ] i18n (`useTranslation`, `t()`)
- [ ] **Lesson 30 — React in freee-payroll 2: data, routing, legacy**
  - [ ] Calling APIs with `@freee-payroll/api` (generated) and `configured-axios`
  - [ ] State rules: local state → Context → Redux
  - [ ] react-router v3, forms (formik / controlled), `$FREEE_DATA.ff`
  - [ ] Recognizing legacy code: `DEPRECATED_*`, flux, Backbone, CoffeeScript, `react_components/`
- [ ] **Lesson 31 — Frontend testing and tooling**
  - [ ] Vitest, Testing Library (`render`, `screen`, `userEvent`)
  - [ ] Storybook stories (also used as snapshot tests)
  - [ ] ESLint, Prettier, `tsc` type checking

## Part 8 — Capstone: reading any part of the repo

- [ ] **Lesson 32 — End-to-end trace: from a button click to the database and back**
  - [ ] Frontend call → route → controller → repository/service → model → JSON → UI
  - [ ] How to find "where is this screen's code?" (URL → route → controller → JS_FILE_MAP → feature dir)
- [ ] **Lesson 33 — Code-reading drills and working day-to-day**
  - [ ] A step-by-step method for reading an unfamiliar file
  - [ ] Debugging: `binding.b`, `debugger`, reading error messages / stack traces
  - [ ] Your first PR: branch naming `username/...`, gitmoji, CI, review, feature flags
  - [ ] Final mixed review (Ruby + Rails + TS + React)

---

## Progress log

| Date | Lesson | Test score | Notes |
|---|---|---|---|
|  |  |  |  |
