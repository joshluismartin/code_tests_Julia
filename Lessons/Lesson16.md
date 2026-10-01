# Lesson 16 — Views, Helpers, and How Rails Hands Off to React

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand how a freee-payroll **web page** is built: which Rails view renders, how the right JavaScript bundle is chosen, how data reaches React through `window.$FREEE_DATA`, and how **feature flags** switch features on and off.
**You need:** Lessons 14–15. Open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. New terms

| Term | Meaning in plain words |
|---|---|
| **HTML** | The language of web pages: tags like `<div>`, `<body>`, `<script>`. |
| **View** | A Rails **template** file that produces the response body (usually HTML). Lives in `app/views/`. |
| **Template** | A file with "holes" where Ruby fills in values. |
| **ERB** (Embedded RuBy) | Rails' default template language: HTML with Ruby inside special tags. Files end in `.html.erb`. |
| **Layout** | The shared outer frame of every page (`<html>`, `<head>`, scripts). Each page's view is inserted into it. |
| **Partial** | A small reusable template piece. Filename starts with `_` (e.g. `_head.html.erb`). |
| **Helper** | A Ruby method available inside views, defined in `app/helpers/`. |
| **SPA** (Single-Page Application) | A page where one HTML shell loads JavaScript, and JavaScript (React) builds everything the user sees. |
| **JS bundle / entrypoint** | The JavaScript file the browser loads for a page. An **entrypoint** is the starting file that imports everything else. |
| **Mount** | To attach a React app to an element in the HTML (e.g. `<div id="app">`). |
| **Global variable** | A variable anyone can access from anywhere. In browsers, properties of `window` are global. |
| **Feature flag** (a.k.a. **release flag**, リリースフラグ) | An on/off switch that enables or hides a feature **without deploying new code**. |

---

## 2. ERB in 5 minutes

ERB has three tags:

| Tag | Does | Output appears in HTML? | Example |
|---|---|---|---|
| `<%= ... %>` | Runs Ruby **and prints** the result | ✅ Yes | `<%= header_message %>` |
| `<% ... %>` | Runs Ruby, **prints nothing** (for `if`, loops, assigning) | ❌ No | `<% if developer_demo_company? %>` |
| `<%# ... %>` | A **comment**, ignored | ❌ No | `<%# erblint:disable AllowedScriptType %>` |

Real example from the main layout:

```erb
<%# app/views/layouts/application.html.erb:40-42 %>
<% if developer_demo_company? %>
  <%= render 'layouts/footer_for_developer_demo_company' %>
<% end %>
```
- `<% if ... %>` / `<% end %>`: Ruby `if`, not printed.
- `<%= render '...' %>`: renders a **partial** and prints its HTML.
- `developer_demo_company?` is a **helper** method.

(illustrative example) With `@name = '太郎'`:
```erb
<p>こんにちは、<%= @name %>さん</p>      →  <p>こんにちは、太郎さん</p>
<% 3.times do |i| %><span><%= i %></span><% end %>   →  <span>0</span><span>1</span><span>2</span>
```

> ⚠️ `<%= %>` **escapes** HTML automatically (turns `<` into `&lt;`). This protects against **XSS** (Cross-Site Scripting: an attacker injecting a `<script>` into your page).

---

## 3. Layouts, `yield`, and partials

The main layout wraps every normal page:

```erb
<%# app/views/layouts/application.html.erb (lines 1-4 and 29-45) %>
<!DOCTYPE html>
<html lang="ja">
  <%= render partial: 'layouts/head' %>
  <body>
  ...
    <%= render 'layouts/freee_js_include' %>
    <%= render 'layouts/javascript_include' %>
    <% if current_auth_user && current_company && @advisor_role.present? %>
      ...
    <% end %>
    <%= render 'shared/google_map' %>
    <%= yield %>
    <% if developer_demo_company? %>
      <%= render 'layouts/footer_for_developer_demo_company' %>
    <% end %>
    <%= render 'layouts/third_party_js' %>
  </body>
</html>
```

| Line | Meaning |
|---|---|
| `render partial: 'layouts/head'` | Insert `app/views/layouts/_head.html.erb` (note the `_`). It includes the CSS (`stylesheet_link_tag *css_file_path` at `_head.html.erb:13`). |
| `render 'layouts/freee_js_include'` | Insert `_freee_js_include.html.erb`, which creates `window.$FREEE_DATA` (section 6). |
| `render 'layouts/javascript_include'` | Insert `_javascript_include.html.erb`, which adds the `<script>` for this page's JS bundle (section 5). |
| **`yield`** | "**Put the page's own view here.**" The action's template is inserted at this spot. |

Also notice `current_auth_user` and `current_company` used directly in the view. They work because of `helper_method` in `ApplicationController` (Lesson 14).

Other layouts exist in `app/views/layouts/`: `admin.html.erb`, `email.html.erb`, `pdf.pdf.erb`, ...

---

## 4. The web page view is (almost) empty

Most freee-payroll pages are **React SPAs**. The Rails side is tiny:

```ruby
# config/routes/web.rb:46
get 'holidays', to: 'web/holidays#index'
```

```ruby
# app/controllers/web/holidays_controller.rb (full)
# typed: true
class Web::HolidaysController < Web::BaseController
  include ErrorHandleable

  def index
    render template: 'web/app'
  end
end
```

```erb
<%# app/views/web/app.html.erb (full) %>
<%#
  View template for single root
  **DO NOT ADD OTHER ELEMENTS EXCEPT `#app`**
%>

<div id="app"></div>
```

That's the whole view: **one empty `<div id="app">`**. It is shared by most pages (`render template: 'web/app'`). The comment is a rule: *don't add anything else*. React fills it in.

> So when you're looking for "the code of the Holidays screen", the Rails view won't help you. It's in `front/javascripts/holidays/`. The next section shows how Rails knows that.

---

## 5. Choosing the JavaScript bundle: `JS_FILE_MAP`

`app/helpers/application_helper.rb` has a big hash from **controller (or controller#action)** to **JS entrypoint**:

```ruby
# app/helpers/application_helper.rb:1-3, 14, 27
module ApplicationHelper
  # stree-ignore
  JS_FILE_MAP = {
    ...
    'web/docs#remittance_form' => 'front/javascripts/DEPRECATED_remittance_form/index.ts',
    ...
    'web/holidays' => 'front/javascripts/holidays/index.ts',
    ...
```

And a helper picks the right one for the current request:

```ruby
# app/helpers/application_helper.rb:76-86
def js_file_path
  inherent_file_path =
    JS_FILE_MAP["#{params[:controller]}##{params[:action]}"] || JS_FILE_MAP["#{params[:controller]}"]
  if inherent_file_path
    inherent_file_path
  elsif params[:controller].start_with?('admin')
    'front/javascripts/admin/index.ts'
  else
    'front/javascripts/legacy_application/index.ts'
  end
end
```

How it works:
1. First try the exact `'controller#action'` key (e.g. `'web/docs#remittance_form'`).
2. Else try just `'controller'` (e.g. `'web/holidays'`).
3. Else, admin pages → `admin/index.ts`.
4. Else → the **fallback** `legacy_application/index.ts` (old Backbone-era code).

There's a matching `CSS_FILE_MAP` / `css_file_path` for stylesheets (`application_helper.rb:56` and `:88`).

The layout partial uses it:

```erb
<%# app/views/layouts/_javascript_include.html.erb:34 %>
<%= javascript_bundle_tag(js_file_path, crossorigin: '', extname: false, type: 'module') %>
```
→ becomes a `<script type="module" src="...holidays-abc123.js">` tag. (In development, lines 24–33 of that file load from the **Vite** dev server at `localhost:8889` instead, with hot reload.)

### 5.1 The entrypoint mounts React on `#app`

```ts
// front/javascripts/holidays/index.ts (full)
import run from '../initializers/run';

import Routes from './Routes';

run({ withReactRouter: Routes });
```

```tsx
// front/javascripts/initializers/run.tsx:45-48 (shortened)
const el = document.querySelector('#app');
...
createRoot(el!).render(
  <NotificationProvider>
    <Provider store={store}>
      ...
          <Routes />
```
- `document.querySelector('#app')` finds the `<div id="app">` from the view.
- `createRoot(...).render(...)` **mounts** React there (Lesson 26).
- `<Routes />` is the feature's own react-router (Lesson 30), which reads the rest of the URL. That's why `web.rb` has catch-alls like `'attendances(/*path)'` (Lesson 14).

### 5.2 The whole path from URL to screen

```
Browser: GET /holidays
  → config/routes/web.rb:46          get 'holidays', to: 'web/holidays#index'
  → Web::HolidaysController#index    render template: 'web/app'
  → layouts/application.html.erb     (head + $FREEE_DATA + JS tag + yield)
       yield → app/views/web/app.html.erb   <div id="app"></div>
       js_file_path → JS_FILE_MAP['web/holidays'] → front/javascripts/holidays/index.ts
  → Browser runs index.ts → run() → createRoot(#app).render(<Routes/>)
  → React screen appears; it calls /api/p/... for data (Lesson 15)
```

> 🔎 **How to find a screen's frontend code:** URL → `config/routes/web.rb` → controller name (e.g. `web/holidays`) → `JS_FILE_MAP` in `application_helper.rb` → `front/javascripts/<feature>/index.ts`.

---

## 6. `window.$FREEE_DATA`: data baked into the page

Some data is needed by almost every screen: the logged-in user, the company, permissions, constants, feature flags. Instead of an API call for each, Rails **writes it into the HTML** as a global JavaScript object:

```erb
<%# app/views/layouts/_freee_js_include.html.erb:1-4, 9, 19 (shortened) %>
<script>
window.$FREEE_DATA =
    <%= json_escape({
      env: Omega.env.display_name,
      ...
      loginUser: @login_user,
      ...
      currentCompany: current_company.present? ? CompanySerializer.new(current_company, root: false) : nil,
      ...
      ff: feature_flags,          <%# line 256 %>
      ...
    }) %>
</script>
```

- A Ruby hash is turned into JSON and assigned to `window.$FREEE_DATA`.
- `json_escape` makes the JSON safe to put inside a `<script>` tag.
- `ff:` holds the **feature flags** from the helper `feature_flags` (section 7).

On the frontend it has a TypeScript type, so the editor knows every key:

```ts
// front/@types/freee-data.d.ts:192
    ff: FeatureFlags;
```
and `FeatureFlags` is listed in `front/@types/feature-flags.d.ts`.

### 6.1 Team rule: reference `$FREEE_DATA` directly

From `front/CLAUDE.md` (translated):
- Global variables are usually bad practice, but `$FREEE_DATA` is **intentionally** global because it's used everywhere and is stable. It's core knowledge of this codebase.
- **Reference it directly.** ❌ Don't do `const fooEnable = $FREEE_DATA.ff.foo_enabled` and then use `fooEnable`. `$FREEE_DATA.ff.foo_enabled` already shows at a glance that it's a feature flag; a local variable hides where it came from.
- ✅ Exception: to keep a function **pure** or to show states in **Storybook**, pass the value in as a prop/argument. **Never** rewrite `$FREEE_DATA` inside Storybook (it would affect other stories).

Real usage:

```tsx
// front/javascripts/home/components/Home/OnboardingTasksPanel.tsx:210-213
const isAvailableNenchoPlan =
  $FREEE_DATA.ff.nencho_plan_enabled &&
  ($FREEE_DATA.applied_plans.current_purchased_plan_code === 'nencho' ||
    $FREEE_DATA.applied_plans.current_trial_plan_code === 'nencho');
```

---

## 7. Feature flags

### 7.1 Why

freee-payroll uses **trunk-based development** (Lesson 2): everyone merges small changes to `main` often. Unfinished or risky features are merged **hidden behind a flag**, then turned on later. That also lets the team turn a feature **off quickly** if something goes wrong. `README.md`: "影響の大きい変更はフィーチャーフラグで既存挙動を保護すること" (protect existing behavior with a feature flag for high-impact changes).

### 7.2 Three mechanisms

From `docs/knowledge/general/coding-standards/feature-flag.md`:

| Mechanism | Read it with | Good for | Watch out |
|---|---|---|---|
| **freee-feature-flag** (first choice) | `Omega::FeatureFlag.active?('flag_name', namespace: :payroll)` | Company-wide flag platform, YAML-managed (GitOps), can span products | Changes are **not instant** (re-fetched every ~300 seconds) |
| **sysparam** (company release flags) | `current_company.xxx_enabled?` | When you need **instant**, precisely timed switching (it's stored in the DB tables `system_parameters` / `company_parameters`) | May hit the DB on each check, so **don't call it inside loops** |
| **Environment variable** | `ENV['...']` | Settings evaluated when a class loads (e.g. ActiveRecord `ignored_columns`) | Changing it requires a K8s resource sync (clusterops) |

**freee-feature-flag** in real code:

```ruby
# app/modules/payroll/infrastructure/repositories/company_allowance_repository.rb (find_list_by_... method)
# shukei1_mins_tag_ph1_enabled 削除時にこの if を削除する
# フラグOFF中は一覧に時間数タグ手当が表示されないため、並び替えの候補からも除外する
unless Omega::FeatureFlag.active?('shukei1_mins_tag_ph1_enabled', namespace: :payroll)
  scope = scope.where.not(calc_type: 'attendance_mins_tag')
end
```

**sysparam** release flags are methods in a model concern:

```ruby
# app/models/concerns/company/release_flags.rb:1-3
# typed: true
# 単なるリリースフラグを作る場合は `bundle exec rails g release_flag` を実行してね！！
module Company::ReleaseFlags
```
```ruby
# app/models/concerns/company/release_flags.rb:160-163
# 勤怠タグ並び替えの有効化
def tnb_2026_asakatu_kintai_tag_sort_enabled?
  parameter(:tnb_2026_asakatu_kintai_tag_sort_enabled)
end
```
Create one with the generator (`lib/generators/release_flag/USAGE`):
```
rails generate release_flag HogeHogeEnabled "Activate HogeHoge"
```
It edits `db/seeds/system_parameters.yml`, `release_flags.rb`, `js_exports_helper.rb`, and `front/@types/feature-flags.d.ts` for you.

A curious real example: a flag that's been **fixed to `true`**, waiting to be deleted:
```ruby
# app/models/concerns/company/release_flags.rb:165-171
# 過去の年末調整を読み取り専用にする機能の有効化
def readonly_obsolete_nencho_enabled?
  # リリース済みで削除できるリリフラだが、範囲が広い&工数に余裕がないので、
  # 事前準備として値を true に固定する
  # TODO: @c-fo/hr_nencho 工数に余裕ができたらリリフラを削除する
  true
end
```

### 7.3 Passing flags to the frontend

```ruby
# app/helpers/js_exports_helper.rb:110-116 (shortened)
# stree-ignore
def feature_flags
  {
    kintai_shukei_1_ui_updated_enabled: !!current_company&.kintai_shukei_1_ui_updated_enabled?,
    ...
    readonly_obsolete_nencho: !!current_company&.readonly_obsolete_nencho_enabled?,
    ...
```
```ruby
# app/helpers/js_exports_helper.rb:149
    nencho_plan_enabled: Omega::FeatureFlag.active?('nencho_plan_enabled', namespace: :payroll),
```

- Both mechanisms end up in one hash → `ff: feature_flags` → **`$FREEE_DATA.ff.nencho_plan_enabled`** in React.
- `!!x` turns any value into a real `true`/`false`. `current_company&.` is safe navigation (Lesson 4), so there's no crash when there's no company (e.g. on the login page).

### 7.4 How to write flag branches (team guidance)

From `feature-flag.md`:
- Prefer **safety and few steps when the flag is deleted later**, even if the code is a bit redundant.
- If the **logic structure changes**, branch **big**: separate old and new modes so deletion = delete one side.
- If the difference is **one value used in one place**, branch at the **expression** level.
- ❌ Failure pattern: duplicating a long method just because a few values differ. You then have to fix bugs twice.
- When a branch becomes its own method/class, give it a **grep-able name** like the suffix `_legacy`.
- To delete a flag, **grep the flag name** and follow callers/callees. Don't rely on "delete me" comments; they rot.
- **Keep flags short-lived.** In tests, separate old/new behavior into different `context`s.
- CI has a job **`rspec_with_release_flag_true`** (`.circleci/config.yml:320`) that runs specs with **all release flags on** (`RSPEC_ALL_RELEASE_FLAG_ENABLED=1`), so the "new" side is tested too.

(illustrative example)
```ruby
# Expression-level branch: one value differs
limit = current_company.bigger_upload_limit_enabled? ? 10.megabytes : 3.megabytes

# Big branch: logic structure differs
def execute(company:)
  if Omega::FeatureFlag.active?('new_calc_enabled', namespace: :payroll)
    calculate(company:)
  else
    calculate_legacy(company:)
  end
end
```

Operational rules (same doc): announce in Slack before changing staging/production flag values; make sure E2E tests pass on staging with the flag on before enabling it in production; minimize the time staging and production values differ.

---

## 8. Helpers

A **helper** is a method you can call from views. Files in `app/helpers/`:

```
app_open_helper.rb  application_helper.rb  bundle_helper.rb  freee_pdf_helper.rb
homes_helper.rb     js_exports_helper.rb   sign_up_flows_helper.rb  tel_support_introduction_helper.rb
```

| Helper you've seen | From | Does |
|---|---|---|
| `js_file_path` / `css_file_path` | `application_helper.rb` | Picks the page's JS/CSS entrypoint |
| `feature_flags` | `js_exports_helper.rb` | Builds `$FREEE_DATA.ff` |
| `javascript_bundle_tag` | `bundle_helper.rb` | Prints the `<script>` tag for a bundle |
| `render`, `link_to`, `form_tag`, `stylesheet_link_tag`, `json_escape` | Rails built-in | Rendering partials, links, forms, CSS tags, safe JSON |
| `current_company`, `current_auth_user` | controller + `helper_method` | Available in views too |

---

## 9. Practice (15 min)

**Practice 1: ERB output.** What HTML does this produce when `@count = 2`?
```erb
<%# 件数 %>
<p>件数: <%= @count %></p>
<% if @count > 1 %><span>複数</span><% end %>
```

**Practice 2: find the bundle.** Using `JS_FILE_MAP`, which JS file loads for `GET /employees` (`web/employees#index`)? What loads for a hypothetical `web/foo#index` not in the map?

**Practice 3: trace.** Open `config/routes/web.rb` and pick any route. Follow it to the controller, check it renders `web/app`, then find its entrypoint in `JS_FILE_MAP` and open that `index.ts`.

**Practice 4: flags.** You need a flag you can switch **instantly** at an exact time for specific companies. Which mechanism? How do you read it in Ruby and in React?

<details><summary>Practice answers</summary>

1. `<p>件数: 2</p>` `<span>複数</span>` (the `<%# %>` comment prints nothing).
2. `front/javascripts/employees/index.ts`. `web/foo` isn't in the map and doesn't start with `admin` → `front/javascripts/legacy_application/index.ts`.
3. (Your own trace.)
4. **sysparam** (company release flag): `current_company.xxx_enabled?` in Ruby; add it to `feature_flags` in `js_exports_helper.rb` (the generator does it) and read `$FREEE_DATA.ff.xxx` in React.
</details>

---

## 10. Summary

- Views are **ERB** templates: `<%= %>` prints, `<% %>` runs, `<%# %>` comments. `render` inserts **partials** (`_name.html.erb`); `yield` inserts the page's view into the **layout**.
- Most web pages render `web/app` → just `<div id="app"></div>`. **React** does the rest.
- `JS_FILE_MAP` + `js_file_path` (in `application_helper.rb`) pick the page's JS entrypoint (`front/javascripts/<feature>/index.ts`); the fallback is `legacy_application`.
- The entrypoint calls `run()` → `createRoot(#app).render(...)`.
- `window.$FREEE_DATA` (built in `_freee_js_include.html.erb`) carries user, company, permissions, and flags (`ff`). **Reference it directly.**
- Feature flags: **freee-feature-flag** `Omega::FeatureFlag.active?` (first choice, ~300s delay), **sysparam** `current_company.xxx_enabled?` (instant, DB), **env vars**. They're exported to React via `feature_flags` → `$FREEE_DATA.ff`.
- Branch so that deleting the flag is safe and easy; keep flags short-lived; CI runs `rspec_with_release_flag_true`.

➡️ Now take **Test16.md**.
