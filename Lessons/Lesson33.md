# Lesson 33 — Code-Reading Drills and Working Day-to-Day

**Time:** ~1 hour (reading method 10 min, drills 30 min, workflow 10 min, then the final exam in Test33)
**Goal:** Have a **repeatable method** to read any file in freee-payroll, practise it on 5 real files that get harder as you go, know how to debug, and know the steps to ship your first PR.
**You need:** Everything from Lessons 1–32. The repo open. Paper or a notes file for the drills.

> The drills' answers are in **Answers33.md, Part 0**. Try each drill *before* you peek.
> Test33 is the **final mixed exam** for the whole course.

---

## 1. The 7-step method for reading an unfamiliar file

Don't read top-to-bottom like a novel. Read in this order:

| Step | Ask yourself | Where to look |
|---|---|---|
| **1. Header** | What language? How strict? | Ruby: `# typed: strict/true/false`, `# frozen_string_literal: true`. TS: `.ts` or `.tsx`, eslint-disable comments |
| **2. Location = meaning** | Which layer/feature is this? | Path: `app/controllers/api/private/` = Private API; `app/modules/<domain>/infrastructure/` = DB/external access; `front/javascripts/<feature>/hooks/` = custom hook |
| **3. Name / namespace** | What is it called and what does the name promise? | `class Payroll::Infrastructure::Factories::CompanyAllowanceFactory` → "builds CompanyAllowance objects". Service → `execute`. Repository → `find_one_by_*`. `useXxx` → hook. `XxxContainer` → data owner |
| **4. Public surface** | What can *others* call? | Ruby: methods before `private`; `class << self` → class methods. TS: `export`s |
| **5. Types / signatures** | What goes in, what comes out? | Sorbet `sig { params(...).returns(...) }`; TS `type Props`, function parameter & return types |
| **6. Collaborators** | Who does it call, who calls it? | Constants it references (`::HrMaster::...`, `API.apiP...`). Grep the name to find callers |
| **7. Tests & docs** | What behaviour is promised? | `spec/` mirror path; `*.test.ts(x)`, `*.stories.tsx`; comments; `docs/knowledge` |

Only **after** these 7 steps, read method bodies. You'll understand them much faster because you already know their job.

**Tips:**
- **Comments in Japanese are gold.** They usually explain *why*. Paste them into a translator if needed.
- **Ignore what you don't need.** If you're debugging one method, you don't need to understand the whole file.
- **Find a caller** to see real inputs: `grep -rn "CompanyAllowanceFactory.build" app`.
- **Legacy vs modern:** `new(...).execute` services, `DEPRECATED_` folders, flux, CoffeeScript are all old but alive. Read them, and don't copy their style into new code.

---

## 2. Drills (≈30 min)

For each drill: open the file, apply the 7 steps, then answer the questions. Answers: **Answers33.md, Part 0**.

### Drill 1 ⭐: `app/controllers/concerns/error_handleable.rb` (70 lines)

Key excerpt (lines 1-24, abbreviated):
```ruby
module ErrorHandleable
  extend ActiveSupport::Concern

  # @override
  def handle_error(original_error, opts = {})
    wrapped_error =
      case original_error
      when ::HrErrors::Base
        original_error
      when ::Errors::PayrollError
        ::HrErrors::UserError.new(original_error.message, original_exception: original_error, status: original_error.status)
      when ::ActiveRecord::RecordInvalid
        ::HrErrors::UserError
          .new('入力内容に誤りがあります。修正のうえ、再度お試しください。', severity: :info, original_exception: original_error, notify_error: true)
          .tap { |error| error.add_invalid_errors(original_error.record) }
      ...
      else
        ::HrErrors::SystemError.new(system_message: original_error.message, original_exception: original_error)
      end

    super(wrapped_error, opts)
  end
```
*(the `PayrollError` branch is folded onto one line here; open the file for the exact code)*

Questions:
1. Is this a class or a module? What does `extend ActiveSupport::Concern` let it do?
2. What does `super(wrapped_error, opts)` call? (Hint: who has the `rescue_from StandardError, with: :handle_error`?)
3. An unexpected `NoMethodError` happens in a controller that includes this. What class does the client's error become, and what HTTP status?
4. What does `.tap { |error| error.add_invalid_errors(...) }` return: the result of the block, or `error`?
5. Look at `render_error` (lines 55-69). Why is there both `format.json` and `format.any`?

### Drill 2 ⭐⭐: `lib/database_selector.rb` (61 lines)

```ruby
    def connected_to_reader(enable_write_forwarding: false, &)
      if enable_write_forwarding
        ...
      else
        begin
          ActiveRecord::Base.connected_to(shard: :default, role: :reading, &)
        rescue ActiveRecord::ReadOnlyError
          ::ReachableChecker.execute(message: 'リードレプリカに書き込もうとしています。')

          # 書き込みクエリがあった場合は writing に向ける
          yield
        rescue => e
          raise e
        end
      end
    end
```
(lines 28-59, abbreviated)

Questions:
1. `module DatabaseSelector` + `class << self`: how do you call this method?
2. What does the lone `&` in the parameter list mean, and where is the block used? (Two places.)
3. Walk through what happens when the block accidentally performs an `UPDATE` in **development** vs **production**. (Remember what `ReachableChecker` does in each.)
4. According to the header comment (lines 6-12), name two situations where you must **not** use this method.

### Drill 3 ⭐⭐⭐: `app/modules/payroll/infrastructure/factories/company_allowance_factory.rb`

Look at lines 1-10, 37-61, and 116-148.

```ruby
class Payroll::Infrastructure::Factories::CompanyAllowanceFactory
  # calc_type と amount_source の組み合わせが成立しない場合に投げる。
  # API 境界で rescue して 400 に差し替えるため、narrow に rescue できるクラスとして定義する。
  class InvalidCombinationError < StandardError
  end
  ...
    def resolve_calculation_setting_class(calc_type, amount_source)
      if fixed_calculation?(calc_type, amount_source)
        ::Payroll::Domain::Models::Allowance::FixedCalculationSetting
      elsif ...
      elsif Omega::FeatureFlag.active?('shukei1_mins_tag_ph1_enabled', namespace: :payroll) &&
            attendance_mins_tag_system_charge_calculation?(calc_type, amount_source)
        # shukei1_mins_tag_ph1_enabled 削除時はフラグ条件のみ外す
        ::Payroll::Domain::Models::Allowance::AttendanceMinsTagSystemChargeCalculationSetting
      ...
      else
        raise InvalidCombinationError, "invalid combination: calc_type=#{calc_type}, amount_source=#{amount_source}"
      end
    end

    sig { params(calc_type: String, amount_source: String).returns(T::Boolean) }
    def fixed_calculation?(calc_type, amount_source)
      calc_type == 'fixed' && amount_source == 'direct'
    end
```

Questions:
1. Which **layer** is this and what is a *factory*'s job there? What does `build` return (look at its `sig`, line 35)?
2. `resolve_calculation_setting_class` returns a **class**, not an instance. How does its `sig` (lines 100-115) express that?
3. Why define a custom `InvalidCombinationError` instead of raising `ArgumentError`? Where is it rescued? (grep!)
4. What will a developer do to this method when the `shukei1_mins_tag_ph1_enabled` flag is removed?
5. `build` has many keyword arguments with `: nil` defaults. Which are required?

### Drill 4 ⭐⭐⭐: `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx` (273 lines)

Look at lines 32-83, 176-197, 203-273.

Questions:
1. The file has `export default InsuranceSettingsIndex` (line 197) and `export const InsuranceSettingsIndexContainer` (line 203). What's the role of each? Which one would a Storybook story render?
2. `TABS[index]?.key ?? 'health_insurance'` (line 81): why is `?.` needed on an array index? (Hint: `tsconfig.json` option.)
3. Why is `switchTab` wrapped in `React.useCallback(..., [])`?
4. What does the `useEffect` at lines 220-230 do and when does it re-run?
5. In `handleChangeDefaultSettings` (lines 242-262): if the health-insurance change fails, is the welfare-pension change attempted? Why was it written this way (read the comment)?
6. Find the route that renders the Container (grep `InsuranceSettingsIndexContainer` in `front/javascripts`). What URL shows this screen?

### Drill 5 ⭐⭐⭐⭐: `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts` (199 lines)

Look at lines 27-30, 96-199.

```ts
export type AttritionRiskSummaryState =
  | { kind: 'error'; message: string }
  | { kind: 'loading' }
  | { kind: 'ok'; summary: AttritionRiskAiSummary };
```

Questions:
1. What TS pattern is `AttritionRiskSummaryState`? How would a component safely read `summary`?
2. `useState<AttritionRiskSummaryState>(() => summaryCache.get(employeeId) ?? { kind: 'loading' })`: why pass a **function** to `useState`?
3. What are `requestInputsRef` and `inFlightRef` for? Why `useRef` and not `useState`?
4. The effect's dependency array is `[employeeId, sendMessage]`, *not* `row`. Why (read lines 106-107)?
5. What problem does `if (inFlightRef.current !== employeeId) return;` inside `finalize` prevent?
6. `fetchSummary().catch((err: unknown) => ...)`: why type `err` as `unknown`, and how is it narrowed?

---

## 3. Debugging toolbox

### Backend (Ruby / Rails)

| Tool | How |
|---|---|
| **`binding.b`** | Put `binding.b` on a line (`debug` gem, in `Gemfile` group `:development, :test`). The server/test **pauses** there and you get a console: inspect variables (`p employee`), `n` (next line), `s` (step into), `c` (continue), `info` (locals). **Remove before committing.** |
| **Stack trace** | Read from the **top**. The first line in *your* code (`app/…`, `lib/…`) is usually where to look; lines in `gems/…` are library internals. Example shape: `app/modules/hr_master/infrastructure/repositories/member_repository.rb:50:in 'find_one_by_employee_id'` → `app/controllers/api/private/employee_display_names_controller.rb:9:in 'show'`. It reads *"raised at repo line 50, which was called from controller line 9"*. |
| **Rails log (development)** | `log/development.log` / the server terminal shows each request: `Started GET "/api/p/..."`, `Processing by Api::Private::XxxController#show`, `Parameters: {...}`, the SQL queries, `Completed 200 OK in 45ms`. Great for "which controller handled this?" and N+1 detection (the `bullet` gem is in the Gemfile's development group). |
| **`bin/rails console`** | Try queries: `Company.first.employees.count`. (Careful: real dev DB.) |
| **Run one spec** | `bin/rspec spec/controllers/api/private/employee_display_names_controller_spec.rb:16` |
| **Production errors** | They go to **Bugsnag** via `Omega::Bugcap.notify(e)` (and `ReachableChecker` in prod). Never use `Rails.logger` for this (Hr/UsedRailsLogger cop; logger is disabled in prod). The `.claude/skills/investigate-bugsnag-error` skill helps investigate. |

### Frontend (TypeScript / React)

| Tool | How |
|---|---|
| **`debugger;`** | Put the statement in TS code. With Chrome DevTools open, execution pauses there. Inspect variables and step through. **Remove before committing.** |
| **Console** | `console.log(value)` for a quick look (don't commit). The Console tab also shows red errors with a clickable file:line. |
| **Network tab** | Every `/api/p/` call: method, status, payload, response. Filter by `api/p`. |
| **React DevTools** (browser extension) | Components tab: see the component tree, **props and state** of each component, which hooks hold what. |
| **Storybook** | `yarn storybook` → develop and inspect a presentational component in isolation. |
| **Vitest** | Run a single test file to reproduce logic bugs quickly. |

---

## 4. Your first PR: the workflow

1. **Ticket + branch.** Trunk-based: branch off `main`, keep it short-lived, and name it with your username prefix (README: `ユーザー名/` so you never overwrite someone else's branch). e.g. `joshua/HRXXX-123_fix_display_name`.
2. **Risky change?** Protect it with a **feature flag** (README rule; `docs/knowledge/general/coding-standards/feature-flag.md`). First choice is freee-feature-flag (`Omega::FeatureFlag.active?('x', namespace: :payroll)`). Unfinished work merges hidden behind the flag.
3. **New API?** Schema first: `*_controller.swagger.yaml` → `make generate-root-swagger` → `make front/packages/@freee-payroll/api/auto-generated` → route → **`config/permission.yaml` entry** → controller (`include ErrorHandleable`) → **request spec** (`spec/controllers/...`, `type: :request`).
4. **Write/adjust tests.** RSpec (`default_company`, DDT style, no explicit ids) and Vitest/Stories for the frontend. Danger warns if an `app/` file has no matching `spec/` file.
5. **Lint & type-check locally** as needed: `bin/rubocop`, `bin/stree`, `yarn lint`, Sorbet, tsc.
6. **Commit with gitmoji.** `.claude/skills/commit-with-gitmoji/SKILL.md` is the repo's way to do it. The skill reads the staged diff, picks an emoji based on `gitmoji -l`, writes a **short title about purpose/intent/reason (not a transcription of the code)**, and *proposes* the message. You approve it before the commit runs. Real examples from `git log`: `🐛 Fix stale form values overwriting edits during postal code lookup`, `✨ Add edit mode to labor consultant detail page`, `♻️ [NEN-2495] レビュー指摘対応: …`.
7. **Open the PR** against `main`, filling `.github/PULL_REQUEST_TEMPLATE.md`: Tickets / Summary / Out of Scope / Test Cases / Note.
8. **CI must pass.** Jobs in `.circleci/config.yml`: `prepare`, `rspec`, `rspec_with_release_flag_true` (all flags ON!), `front_test`, `prettier`, `syntax_tree`, `rubocop`, `ruby_type_check`, `private_api_schema`, `permission_config`, `danger`, `outbox-capturer-lint`, and more. **Danger** (`Dangerfile`) posts automatic comments, e.g. missing spec (`warn_on_not_correspond_impl_to_spec`), missing change reason, migrations mixed with other changes (`fail_on_migration_with_other_changes`), explicit record locks, and so on.
9. **Review.** A code review is required, and for files listed in `.github/CODEOWNERS` (e.g. `/Gemfile @C-FO/hr_quality`) the **code owner** must approve too. Per README, code owners check two things: **consistency with spec/design policy** and **conflicts with ongoing development**. They don't necessarily review implementation details. Choose whom to ask for review based on how unsure you feel.
10. **Merge, then later remove the flag** (keep flags short-lived; grep the flag name to find every branch point).

---

## 5. Glossary index: where each term was taught

| Term(s) | Lesson |
|---|---|
| client/server, HTTP, status code, JSON, API, database, framework, multi-tenant, company (事業所) | 1 |
| terminal, Git, commit, branch, PR, trunk-based, gitmoji, Gemfile, package.json, linter, formatter, CI, Danger | 2 |
| Integer/String/Symbol/nil, method, `?`/`!`, interpolation | 3 |
| `if/unless/case`, truthiness, `&.`, `\|\|=`, Array, Hash, Range, `present?` | 4 |
| block, `each/map/select/group_by/index_by/each_slice`, `&:x`, lambda | 5 |
| class, object, `initialize`, `@ivar`, `attr_reader`, `class << self`, `private`, inheritance, `super` | 6 |
| module, namespace `::`, `include/extend`, Concern, keyword args, `raise/rescue/ensure`, `freeze` | 7 |
| Sorbet, sigil, `sig`, `T.nilable`, `T::Array`, `T.must`, `T::Struct`, `T::Enum` | 8 |
| MVC, convention over configuration, Zeitwerk, environments, request lifecycle | 9 |
| table/row/column, primary/foreign key, index, SQL, migration, schema.rb, replica | 10 |
| ActiveRecord, CRUD, `where/find/find_by/pluck`, lazy relation, `save!` | 11 |
| `belongs_to/has_many`, validations, callbacks, `scope`, `default_scope` | 12 |
| soft delete `DEFAULT_DELETED_AT`, enumerize/enum/aasm, LeakCheck, N+1, `find_each`, transaction, `connected_to_reader` | 13 |
| routes, `resources/resource`, `member/collection`, controller, params, strong params, `before_action`, `render json:` | 14 |
| Private API, OpenAPI/swagger, committee, `ErrorHandleable`, `HrErrors`, serializer | 15 |
| ERB, layout, helper, `JS_FILE_MAP`, `$FREEE_DATA`, feature flags | 16 |
| service (`execute`), repository naming, value object, presenter, decorator, policy specs, cops | 17 |
| DDD, domain, entity, use case, factory, query service, layers, `Dependency/OverBoundary`, DAG | 18 |
| ActiveJob, queue, Resque, microservice, gRPC/protobuf, `Omega::Bugcap`, `ReachableChecker`, encryption | 19 |
| RSpec, `describe/context/it`, `let/let!`, `subject`, matchers, `change`, `raise_error` | 20 |
| FactoryBot, trait, `default_company`, request spec, mocks/stubs, `have_received`, WebMock, `travel_to`, DDT, shared_examples | 21 |
| `const/let`, arrow functions, destructuring, spread, `import/export` | 22 |
| `===`, `?.`, `??`, `map/filter/reduce`, Promise, `async/await` | 23 |
| TS types, `type` vs `interface`, union, literal types, `any/unknown` | 24 |
| generics, `Pick/Omit/Partial/Record`, narrowing, `as const`, `.d.ts`, tsconfig, zod | 25 |
| component, JSX, props, `key`, conditional rendering, events | 26 |
| state, `useState`, `useEffect`, deps, cleanup, Rules of Hooks | 27 |
| `useMemo/useCallback/useRef`, custom hook, Context, `useReducer`, Redux ducks | 28 |
| feature directory, entrypoint, Vite, Yarn workspaces, Vibes, i18n | 29 |
| `@freee-payroll/api`, configured-axios, react-router v3, Formik, legacy (flux/Backbone/CoffeeScript) | 30 |
| Vitest, Testing Library, Storybook snapshots, ESLint, Prettier | 31 |
| end-to-end trace, finding a screen's code, find-route | 32 |
| reading method, debugging, PR workflow | 33 |

---

## 6. What's next: how to keep learning

1. **`docs/knowledge/`**: the official rules. Re-read `general/architecture/overview-backend.md`, `overview-frontend.md` and `coding-standards/*` now; they'll make much more sense.
2. **The directory guides**: `app/README.md`, `front/README.md`, `spec/README.md`, `app/controllers/api/private/README.md`, `app/modules/README.md`.
3. **`.claude/skills/`** (49 skills): each `SKILL.md` is a step-by-step recipe written by the team (e.g. `find-route`, `gen-private-api`, `run-related-tests`, `review-pr-checklist`). Reading them teaches you *how the team works*.
4. **Read merged PRs** in your team's area every week. Read the diff with the 7-step method, then the review comments.
5. **Pair / shadow**: ask a teammate to share their screen while they trace a bug. Watch which files they open first.
6. **Your team's docs** in `docs/teams/<team>/`.
7. **Redo the traces** in Lesson 32 on a feature you'll actually work on, and write them into `Learning.md` → "My Notes".
8. Ubiquitous language: when you meet a romanized Japanese identifier (`geppen`, `nencho`, `shaho`), look it up in the HR ユビキタス言語辞書 linked from `docs/knowledge/general/architecture/ubiquitous-language.md`.

---

## 7. Summary

- Read files in 7 steps: **header → location → name → public surface → types → collaborators → tests**, and only then bodies.
- Debug with `binding.b`, stack traces (top down, find your code), the Rails log; `debugger`, the Network tab, React DevTools; production errors live in Bugsnag.
- PR flow: branch `username/…` → flag if risky → schema first for APIs → permission.yaml → specs → gitmoji commit → PR template → CI + Danger → review (+ CODEOWNERS) → merge → clean up the flag.
- You now know every layer. Keep reading real code daily. That's how "able to read" becomes "able to write".

➡️ Now take the **final exam: Test33.md**. 🎓
