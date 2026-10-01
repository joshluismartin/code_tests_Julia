# Lesson 2: Your Tools (Terminal, Git, and Reading a Repo)

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Be able to move around the repo in a terminal, understand the Git words your team uses, know what the "config/manifest" files at the top of `freee-payroll/` are for, and find any class or method in the codebase.
**You need:** A terminal (the macOS "Terminal" app or the one inside your editor) and `~/Josh_Files/freee-payroll/`.

---

## 1. The terminal

| Term | Meaning in plain words |
|---|---|
| **Terminal** | A window where you type commands instead of clicking. |
| **Shell** | The program inside the terminal that reads your commands. On your Mac it's **zsh**. |
| **Command** | One instruction, e.g. `ls`. Often followed by **arguments** (what to act on) and **options/flags** (how), e.g. `ls -la app`. |
| **Working directory** | The folder you are "standing in" right now. Commands act relative to it. |
| **Prompt** | The text before your cursor (e.g. `~/Josh_Files %`) that means "ready for a command". |

### 1.1 Paths

A **path** is a file's address.

| Kind | Example | Meaning |
|---|---|---|
| **Absolute path** | `/Users/joshualuisb.martin/Josh_Files/freee-payroll/Gemfile` | Starts with `/`: the full address from the top of the disk. Works from anywhere. |
| **Relative path** | `app/models/comment.rb` | Starts from the working directory. |
| `~` | `~/Josh_Files` | Shortcut for your home folder (`/Users/joshualuisb.martin`). |
| `.` | `./bin/rspec` | "This folder". |
| `..` | `cd ..` | "The parent folder" (one level up). |

### 1.2 The commands you'll use every day

| Command | What it does | Example |
|---|---|---|
| `pwd` | **P**rint **w**orking **d**irectory: where am I? | `pwd` |
| `cd` | **C**hange **d**irectory: move into a folder | `cd ~/Josh_Files/freee-payroll` |
| `ls` | **L**i**s**t files in a folder | `ls app` / `ls -la` (show hidden files + details) |
| `cat` | Print a whole file | `cat .tool-versions` |
| `head` / `tail` | Print the first / last lines | `head -20 Gemfile` |
| `less` | Scroll through a long file (`q` to quit) | `less config/routes.rb` |
| `grep` | Search *inside* files for text | `grep -rn "DEFAULT_DELETED_AT" config` |
| `find` | Search for files *by name* | `find app -name "comment*.rb"` |
| `wc -l` | Count lines | `ls db/migrate \| wc -l` |
| `\|` (pipe) | Send one command's output into the next | `ls app/models \| head` |

**`grep` flags to remember:**
- `-r` = recursive (search all files in all sub-folders)
- `-n` = show line numbers
- `-i` = ignore upper/lower case
- `-l` = only list the file names

```bash
cd ~/Josh_Files/freee-payroll
grep -rn "DEFAULT_DELETED_AT =" config
# config/constants.rb:51:DEFAULT_DELETED_AT = Time.utc(1000, 1, 1)
```

That output means: *in file `config/constants.rb`, on **line 51**, there is this text.* The same `file:line` format is how lessons (and code reviews) point you to code.

> Files whose names start with `.` (like `.rubocop.yml`, `.tool-versions`) are **hidden files**. `ls` doesn't show them; `ls -a` does.

---

## 2. Git: the history of the code

| Term | Meaning in plain words |
|---|---|
| **Git** | A tool that records every change to the files in a repo, so you can see who changed what, when, and why, and go back if needed. |
| **Commit** | One saved "snapshot" of changes, with a **message** explaining it. Each has an ID (hash) like `a56412e8fea`. |
| **Branch** | A separate line of work. You make a branch, commit on it, and the main code isn't affected until you merge. |
| **`main`** | The main branch: the "official" code that gets deployed. |
| **Merge** | Combining a branch's commits into another branch (usually into `main`). |
| **Pull Request (PR)** | A request on GitHub: "please review my branch and merge it into `main`." Teammates comment and approve there. |
| **Code review** | Teammates reading your PR and suggesting improvements before it's merged. |
| **Clone** | Download a copy of a repo from GitHub to your machine. |
| **Pull / Push** | Pull = download new commits from GitHub. Push = upload your commits to GitHub. |
| **Conflict** | When two branches changed the same lines, Git can't merge automatically and asks you to choose. |
| **Diff** | The list of lines added (+) and removed (−) by a change. |

### 2.1 How freee-payroll uses Git (from `README.md`)

- **Trunk-based development**: everyone makes short-lived branches and merges into `main` (the "trunk") often. No long-living feature branches.
- Unfinished work is hidden behind a **feature flag** (a switch that turns new behavior on/off). You'll learn this in Lesson 16.
- Branch names start with **your username + `/`**, e.g. `joshua/fix-bonus-label`. This prevents accidentally overwriting someone else's branch, since everyone pushes to the same shared repo (no forks).
- Merging a PR needs **CI to pass** and a **code review**. Files listed in **CODEOWNERS** also need the owner team's review.
- Commit messages start with a **gitmoji**: an emoji showing the *kind* of change (skill: `.claude/skills/commit-with-gitmoji/SKILL.md`).

| Gitmoji | Meaning |
|---|---|
| ✨ | New feature |
| 🐛 | Bug fix |
| ♻️ | Refactor (restructure code without changing behavior) |
| 💬 | Change text/wording |
| 🔒 (`:lock:`) | Security fix |
| ✅ | Add/update tests |
| 🔥 | Remove code/files |

### 2.2 🔎 In freee-payroll: the real history

Run `git log --oneline -12` inside the repo. At the time of writing it included these lines:

```
dd879f630b3 Merge pull request #95649 from C-FO/plg_expansion/PLGE-916-2
bafa698fbee Merge pull request #95687 from C-FO/hr-kyuyo-romu/HRHDK-336
a56412e8fea 💬 入社区分CSVヘッダーの括弧を全角に統一する
c258970e191 ♻️ [NEN-2495] レビュー指摘対応: メッセージ定数名を HOUSING_LOAN_COPY_EXTENSION_* に改名
774fbf0d1bf 🐛 Fix stale form values overwriting edits during postal code lookup
```

How to read it:
- `dd879f630b3` is the commit hash (ID).
- "Merge pull request #95649 from C-FO/plg_expansion/PLGE-916-2" means PR number 95649 was merged. The branch name was `plg_expansion/PLGE-916-2` (prefix/ticket number).
- `💬 入社区分CSVヘッダーの括弧を全角に統一する`: a wording change (💬): "make the brackets in the joining-type CSV header full-width."
- `♻️ [NEN-2495] ...`: a refactor for Jira ticket NEN-2495.
- `🐛 Fix stale form values...`: a bug fix.

Useful Git commands (you don't have to memorize them all yet):

```bash
git status                 # what did I change?
git switch -c joshua/my-fix # make a new branch and move to it
git add app/models/foo.rb  # stage a file (choose it for the next commit)
git commit -m "🐛 Fix ..."   # save a commit
git push                   # upload to GitHub
git log --oneline -10      # recent history
git blame app/models/comment.rb  # who last changed each line
```

---

## 3. Manifest and settings files at the top of the repo

A **manifest** is a file that lists what a project needs.

| File | What it is |
|---|---|
| `.tool-versions` | Which versions of languages to use. Read by version managers such as **mise** / **asdf**. |
| `Gemfile` | The list of Ruby gems (libraries) the app needs, with version rules. |
| `Gemfile.lock` | The **exact** versions actually installed (generated; don't hand-edit). "Lock" = locked-in versions so everyone gets the same ones. |
| `package.json` | The list of JavaScript packages + **scripts** (named commands like `yarn lint`). |
| `yarn.lock` | Exact JS versions installed (generated). |
| `.rubocop.yml` | Ruby style rules. |
| `.eslintrc.js`, `.prettierrc.js` | JS/TS lint and format rules. |
| `tsconfig.json` | TypeScript compiler settings. |
| `Makefile` | Shortcuts: `make <target>` runs a named recipe. |
| `Dockerfile`, `compose-files/` | Recipes to run the app and its helper services in **containers** (isolated mini-computers). |

### 🔎 In freee-payroll

`.tool-versions:1-2`
```
ruby 3.3.12
nodejs 22.22.2
```

`Gemfile:3`
```ruby
gem 'rails', '7.2.2.1'
```
means "use exactly Rails version 7.2.2.1". Other lines: `Gemfile:53` `gem 'sorbet-runtime'` (no version = any), `Gemfile:116` `gem 'rspec-rails'` (inside `group :test do`, so only installed for tests).

`package.json:137` → `"react": "18.3.1"`, `package.json:78` → `"typescript": "6.0.3"`, and a script at `package.json:11`:
```json
"lint": "eslint --cache --cache-strategy content --quiet front",
```
So typing `yarn lint` runs ESLint over the `front/` folder.

**Version number meaning** (`MAJOR.MINOR.PATCH`, "semantic versioning"): `7.2.2` = major 7, minor 2, patch 2. A major bump may break things; a patch bump is only fixes. `~> 5.0.3` in a Gemfile means "5.0.3 or newer, but below 5.1".

---

## 4. Guidance files: read these before coding

| File | Purpose |
|---|---|
| `README.md` | Front page of the repo: dev rules, architecture overview, links. |
| `CLAUDE.md` | Instructions for AI assistants (and humans!). In this repo most `CLAUDE.md` files just contain `@README.md`, meaning "go read the README next to me". |
| `app/README.md` | Backend rules that differ from normal Rails (important!). |
| `front/README.md` | Frontend rules. |
| `spec/README.md` | Testing rules. |
| `app/controllers/api/private/README.md` | Private API rules. |
| `docs/knowledge/` | The official knowledge base: architecture + coding standards (in Japanese). |
| `.claude/skills/` | Step-by-step procedures (e.g. `commit-with-gitmoji`, `find-route`, `review-pr`). |

> Tip: these docs are in Japanese. Use a translator if needed. They're short bullet lists, and the rules in them are exactly what reviewers will check.

---

## 5. Quality tools: linters, formatters, type checkers, tests, CI

| Term | Plain meaning | freee-payroll tools |
|---|---|---|
| **Linter** | A program that reads your code and warns about bad style or likely bugs, *without running it*. | **RuboCop** (Ruby), **ESLint** (TS/JS), **erb-lint** (HTML templates) |
| **Formatter** | Automatically rewrites spacing/line breaks so everyone's code looks the same. | **syntax_tree** (`stree`, Ruby), **Prettier** (TS/JS) |
| **Type checker** | Checks that values have the right *kind* (e.g. you don't pass text where a number is expected). | **Sorbet** (Ruby, the `ruby_type_check` CI job), **tsc** (TypeScript) |
| **Automated test** | Code that runs your code and checks the result. | **RSpec** (Ruby, `spec/`), **Vitest** (TS, `*.test.ts(x)`) |
| **CI** (Continuous Integration) | A server that runs all the checks automatically on every PR. | **CircleCI** (`.circleci/config.yml`) |
| **Danger** | A bot that comments on PRs with warnings (e.g. "you changed a file but there's no spec for it"). | `Dangerfile` |

### 🔎 In freee-payroll: CI jobs

`.circleci/config.yml` defines jobs including (line numbers in that file): `prepare` (82), `rspec` (251), `rspec_with_release_flag_true` (320, which runs all tests with every feature flag ON), `prettier` (415), `front_test` (426), `syntax_tree` (451), `rubocop` (463), `ruby_type_check` (477), `danger` (500), `private_api_schema` (530), `permission_config` (553).

A Danger rule, `Dangerfile:14`:
```ruby
warn 'ディレクトリの構造をシンプルに保つため、request specはspec/controllers/*に移動したうえでtype: :requestを付与してください。'
```
("To keep the directory structure simple, move request specs to spec/controllers/* and add `type: :request`.") So even the bot enforces team conventions.

### `bin/` scripts

`bin/` contains wrappers that run tools with the right settings: `bin/rails`, `bin/rspec`, `bin/rubocop`, `bin/stree`, `bin/tapioca`, `bin/yarn`, `bin/bundle`, `bin/rake`, `bin/setup`.

```bash
bin/rspec spec/models/banner_spec.rb     # run one test file
bin/rspec spec/models/banner_spec.rb:12  # run the test on line 12
bin/rubocop app/models/comment.rb          # lint one file
bin/stree write app/models/comment.rb      # format one file
yarn lint                                  # lint frontend
```

> The team value "Simple Made Easy" (`docs/knowledge/general/coding-standards/overview.md`): don't hide tools behind your own wrapper scripts; learn the tool itself.

---

## 6. How to search the codebase

You will spend more time **finding** code than writing it. Four techniques:

### 6.1 Search by text (grep / editor search)

```bash
grep -rn "def find_one_by_employee_id" app       # where is this method defined?
grep -rln "find_one_by_employee_id" app spec     # which files use it?
```
In VS Code / your editor: **Cmd+Shift+F** = search across all files; **Cmd+P** = open a file by typing part of its name.

### 6.2 From a class name to its file (the Rails naming rule)

Rails (with its loader called **Zeitwerk**) requires that a class's name matches its file path:

| Rule | Example |
|---|---|
| `CamelCase` class name → `snake_case` file name | `CommentsController` → `comments_controller.rb` |
| Each `::` in the name → a folder `/` | `Api::Private::CommentsController` → `api/private/comments_controller.rb` |
| Look under the right `app/` sub-folder (controllers, models, services...) | → `app/controllers/api/private/comments_controller.rb` ✅ (it exists) |

More:
- `EmployeeTimeClock` (a model) → `app/models/employee_time_clock.rb`
- `HrMaster::Infrastructure::Repositories::MemberRepository` → `app/modules/hr_master/infrastructure/repositories/member_repository.rb` (the `app/modules/` folder is a root, so its sub-folders become namespaces)
- Test for a file: swap `app/` → `spec/` and add `_spec`. `app/controllers/api/private/comments_controller.rb` → `spec/controllers/api/private/comments_controller_spec.rb` ✅ (exists)

### 6.3 From a URL to its code

URL `/api/p/comments` → search `config/routes.rb` for `comments` → `config/routes.rb:234` `resources :comments, only: %i[index create destroy]` inside the `/api/p` block → controller `Api::Private::CommentsController`. (Lesson 14 explains this properly. The repo also has a skill `.claude/skills/find-route`.)

### 6.4 From a line to its history

`git log -p app/models/comment.rb` shows every change to that file; `git blame` shows who last touched each line. Then read the PR to learn *why*.

---

## 7. Practice (15 min)

Do these in the terminal, inside `~/Josh_Files/freee-payroll`.

1. `pwd`, then `ls`, then `ls -a`. What extra files appear with `-a`? Name three.
2. `cat .tool-versions`. Write down the Ruby version.
3. `grep -n "gem 'rails'" Gemfile`. Which line is it on?
4. `ls db/migrate | wc -l`. About how many migrations are there?
5. Use the naming rule: where is the file for `Api::Private::EmployeeDisplayNamesController`? Check with `ls`.
6. `grep -rn "resources :employee_display_names" config` (which file and line?)
7. `git log --oneline -5`. Find one commit that starts with a gitmoji and say what kind of change it is.
8. Open `app/README.md` and find the rule about `Rails.logger`. (Answer: don't use it; the `Hr/UsedRailsLogger` cop enforces this, and notifications go via `Omega::Bugcap.notify`.)

*Practice answers: 1 e.g. `.rubocop.yml`, `.gitignore`, `.tool-versions`, `.circleci` · 2 `3.3.12` · 3 line 3 · 4 about 2,000 · 5 `app/controllers/api/private/employee_display_names_controller.rb` · 6 `config/routes.rb:236` · 7 depends on today's log · 8 see above.*

---

## 8. Summary

- Terminal: `cd`, `ls`, `cat`, `grep -rn`, `find`; paths are absolute (`/...`) or relative; `~` is home.
- Git: commits on a **branch** (`username/...`) → **PR** → review + CI → **merge** into `main` (trunk-based). Commit messages start with a **gitmoji**.
- `Gemfile`/`package.json` list dependencies; `.lock` files pin exact versions; `.tool-versions` pins Ruby/Node.
- Read `README.md`/`app/README.md`/`front/README.md`/`spec/README.md` and `docs/knowledge/` for team rules.
- Linters (RuboCop/ESLint), formatters (stree/Prettier), type checkers (Sorbet/tsc), tests (RSpec/Vitest) all run in **CI** (CircleCI) on every PR; **Danger** comments on PRs.
- Class name ↔ file path: `A::B::FooBar` → `a/b/foo_bar.rb`.

➡️ Now take **Test2.md**.
