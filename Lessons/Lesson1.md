# Lesson 1 — How Web Apps Work + A Map of freee-payroll

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Know the basic vocabulary of web software and be able to say, in plain words, what each big folder of `freee-payroll/` is for.
**You need:** Nothing except this file. For the practice parts, open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. The very basic words

| Term | Meaning in plain words | Example |
|---|---|---|
| **Program / software** | A set of instructions a computer follows. | freee人事労務 (freee HR) is a program. |
| **Code / source code** | The text of those instructions, written by people in a *programming language*. | `def execute ... end` |
| **Programming language** | A strict "language" computers can follow. Each one has its own grammar (**syntax**). | Ruby, TypeScript |
| **File** | One document on disk. The **extension** (the end of the name) tells you what's inside. | `employee.rb` (Ruby), `Button.tsx` (TypeScript + React) |
| **Directory / folder** | A container of files and other folders. | `app/models/` |
| **Path** | The address of a file, made of folders separated by `/`. | `app/models/employee.rb` |
| **Repository (repo)** | A project folder whose full history of changes is tracked by **Git**. | `freee-payroll/` is a repo. |
| **Bug** | A mistake in code that makes the program behave wrongly. | Salary shown as ¥0 |

### File extensions you will see in freee-payroll

| Extension | Language / meaning |
|---|---|
| `.rb` | Ruby |
| `.erb` | HTML with Ruby mixed in (a *template*) |
| `.ts` | TypeScript |
| `.tsx` | TypeScript + JSX (React components) |
| `.js` | JavaScript |
| `.coffee` | CoffeeScript (old, legacy JavaScript dialect) |
| `.scss` | Sass: a nicer way to write CSS (styles/colors/layout) |
| `.yml` / `.yaml` | YAML: a settings/data file format |
| `.json` | JSON: a data format (see below) |
| `.md` | Markdown: formatted text (like this lesson) |
| `.tsp` | TypeSpec: a language for describing APIs |

---

## 2. How a web app works

### 2.1 Client and server

- **Client**: the program *asking* for something. For us, it's usually the **browser** (Chrome).
- **Server**: a computer (and the program on it) that *answers*. freee-payroll's server program is written in **Ruby on Rails**.

```
 You (user)
   │ click
   ▼
┌────────────┐   request  ─────────────▶  ┌──────────────┐        ┌──────────┐
│  Browser   │                            │   Server     │ ─────▶ │ Database │
│ (client)   │  ◀─────────────  response  │ (Rails app)  │ ◀───── │ (MySQL)  │
└────────────┘                            └──────────────┘        └──────────┘
```

### 2.2 Frontend and backend

- **Frontend**: the part that runs *in the browser*, meaning what you see and click. In freee-payroll: **TypeScript + React**, in the `front/` folder.
- **Backend**: the part that runs *on the server*: business rules, saving data, calculations. In freee-payroll: **Ruby + Rails**, mainly in `app/`, `lib/`, `config/`, `db/`.

> "Business logic" = the rules of the business. Example: "overtime after 8 hours is paid at 1.25×". Most of these rules live in the backend.

### 2.3 HTTP: the language browsers and servers speak

**HTTP** (HyperText Transfer Protocol) is the set of rules for sending messages on the web.

- A **request** is a message from client → server.
- A **response** is the answer from server → client.

Every request has:

1. A **URL**: *which* thing you want. Example: `https://p.secure.freee.co.jp/api/p/employee_display_names/1`
   - `https://`: the protocol (secure HTTP)
   - `p.secure.freee.co.jp`: the **domain** (which server)
   - `/api/p/employee_display_names/1`: the **path** (which thing on that server)
   - Anything after `?` is a **query string**, e.g. `?year=2026&month=9` (extra options)
2. An **HTTP method** (a "verb"): *what* you want to do.

| Method | Meaning | Example |
|---|---|---|
| `GET` | Read / fetch something | Show employee #1 |
| `POST` | Create something new | Create a new employee |
| `PATCH` / `PUT` | Update something | Change an employee's name |
| `DELETE` | Remove something | Delete a comment |

3. Optionally a **body**: the data you send (e.g. a form's contents), usually as JSON.

Every response has a **status code**: a number that says how it went.

| Code | Name | Meaning |
|---|---|---|
| `200` | OK | Success |
| `201` | Created | Success, something new was made |
| `204` | No Content | Success, nothing to send back |
| `302` | Found (redirect) | "Go look over there instead" |
| `400` | Bad Request | Your request was malformed |
| `401` | Unauthorized | You're not logged in |
| `403` | Forbidden | Logged in, but not allowed |
| `404` | Not Found | That thing doesn't exist |
| `422` | Unprocessable Entity | The data was invalid (e.g. validation failed) |
| `500` | Internal Server Error | The server crashed (a bug) |

> Rule of thumb: **2xx = success, 3xx = go elsewhere, 4xx = the client's fault, 5xx = the server's fault.**

### 2.4 JSON: the data format

**JSON** (JavaScript Object Notation) is how the frontend and backend exchange data. It looks like this:

```json
{
  "id": 1,
  "display_name": "給与太郎",
  "is_deleted": false,
  "tags": ["full_time", "tokyo"],
  "department": null
}
```

- `{ }` = an **object** (a set of *key: value* pairs)
- `[ ]` = an **array** (an ordered list)
- values can be: string `"..."`, number `1`, boolean `true`/`false`, `null` (nothing), object, array

This exact response (`{ id: 1, display_name: '給与太郎' }`) comes from a real freee-payroll endpoint; we'll trace it fully in Lesson 15.

### 2.5 API

An **API** (Application Programming Interface) is a set of URLs that a program (not a human) can call to get or change data. Each URL + method is called an **endpoint**.

freee-payroll has several kinds of API (you'll see these folder names under `app/controllers/api/`):

| Kind | URL starts with | Who calls it |
|---|---|---|
| **Private API** | `/api/p/` | freee-payroll's own frontend (React) |
| **Internal API** | `/api/internal/` | other freee services (server-to-server) |
| **Public API** | `/api/v1/` | outside developers / customers' programs |

### 2.6 Database

A **database** stores data permanently. freee-payroll uses **MySQL**, a *relational* database, where data is stored in **tables**:

`employees` table (simplified; the real one has many more columns):

| id | company_id | num | last_name | first_name | entry_date |
|---|---|---|---|---|---|
| 1 | 10 | EMP001 | 給与 | 太郎 | 2020-04-01 |
| 2 | 10 | EMP002 | 労務 | 花子 | 2021-10-01 |

- **Table**: like one spreadsheet sheet (`employees`)
- **Row / record**: one item (one employee)
- **Column / field**: one attribute (`last_name`)
- **id**: a unique number for each row (the **primary key**)
- **company_id**: points at a row in *another* table (`companies`); this is called a **foreign key**

freee-payroll is **multi-tenant**: many customer companies share the same database, so almost every table has `company_id` to say whose data each row is. In the code, "company" is **事業所 (jigyōsho)**, one customer's business.

### 2.7 Framework, library, package, dependency

- **Library**: code someone else wrote that you can *use* (call) in your code. Example: `date-fns` for dates.
- **Framework**: a bigger "skeleton" that *calls your code*: you fill in the blanks following its rules. **Rails** is a backend framework; **React** is a frontend library that feels framework-like.
- **Package**: a library bundled so it can be installed. Ruby packages are called **gems**; JavaScript packages are **npm packages**.
- **Dependency**: a package your project needs. Listed in `Gemfile` (Ruby) and `package.json` (JavaScript).
- **Package manager**: the tool that installs them: **Bundler** (Ruby, `bundle install`) and **Yarn** (JS, `yarn install`).

### 2.8 Services and microservices

freee has many products (accounting, HR, sign, ...). Some shared functions live in *separate server programs* called **microservices**. freee-payroll talks to them over the network. In the code you'll see their nicknames, e.g. **Sekisyo** (permissions), **nest-auth** (members/users), **Wolf**, **Ohmu**, **Delibird** (notifications). Don't worry about them yet; just know that "not everything is in our database."

---

## 3. The big picture of freee-payroll

**freee-payroll** is the repo for **freee人事労務 (freee HR & Payroll)**: employee records, attendance (勤怠), payroll (給与計算), year-end adjustment (年末調整), social insurance procedures, and more.

```
┌─────────────────────────── Browser ───────────────────────────┐
│  React components written in TypeScript  (front/)              │
│  - get initial data from window.$FREEE_DATA                    │
│  - call the Private API with @freee-payroll/api                │
└──────────────┬─────────────────────────────────────────────────┘
               │ HTTP (JSON)
┌──────────────▼───────────── Rails server (Ruby) ───────────────┐
│ config/routes.rb   → which controller handles which URL        │
│ app/controllers/   → receive request, return response          │
│ app/services/      → business logic ("do this task")           │
│ app/repositories/  → fetch/save data                           │
│ app/models/        → ActiveRecord models = database tables     │
│ app/modules/       → newer, domain-organized code              │
└──────┬───────────────────────────────┬─────────────────────────┘
       │ SQL                           │ gRPC / HTTP
┌──────▼──────┐                 ┌──────▼──────────────────────┐
│   MySQL     │                 │ Other freee microservices    │
└─────────────┘                 └─────────────────────────────┘
```

Tech in one line each:

- **Ruby 3.3** (see `.tool-versions`): the backend language
- **Rails 7.2** (see `Gemfile`: `gem 'rails', '7.2.2.1'`): the backend framework
- **Sorbet**: adds type checking to Ruby (`sig { ... }` lines)
- **RSpec**: Ruby testing tool (`spec/` folder)
- **Node.js 22**: runs JavaScript tools on your machine (build, test)
- **TypeScript 6**: typed JavaScript, the frontend language
- **React 18**: builds the user interface from **components**
- **Vite**: bundles frontend code for the browser
- **Vitest / Storybook**: frontend testing and component catalog
- **MySQL**: the database

---

## 4. Tour of the top-level folders

Open `~/Josh_Files/freee-payroll/` and find each of these:

| Folder / file | What it is | You'll learn it in |
|---|---|---|
| `app/` | The main backend code (Rails application) | Lessons 9–19 |
| `app/controllers/` | Handle HTTP requests | 14–15 |
| `app/models/` | Database tables as Ruby classes (600+ files!) | 11–13 |
| `app/services/` | Business logic, one task per class | 17 |
| `app/repositories/` | Data access classes | 17 |
| `app/values/` | Value objects (small immutable data holders) | 17 |
| `app/modules/` | Newer code grouped by business area (payroll, attendance, hr_master...) | 18 |
| `app/jobs/` | Background jobs (work done later, not during a request) | 19 |
| `app/views/` | HTML templates (`.erb`) | 16 |
| `config/` | Settings. `routes.rb` = URL map; `database.yml` = DB connection | 9, 14 |
| `db/` | `migrate/` (~2,000 DB change scripts) and `schema.rb` (the current DB shape) | 10 |
| `lib/` | Shared helper code not specific to one feature (clients for other services, utilities) | 19 |
| `spec/` | Ruby tests (RSpec) | 20–21 |
| `front/` | All frontend code (TypeScript/React) | 22–31 |
| `front/javascripts/` | One folder per feature (e.g. `employees`, `yearend_adjustments`) | 29 |
| `front/packages/` | Shared internal JS packages (`@freee-payroll/api`, `@freee-payroll/components`...) | 29–30 |
| `docs/knowledge/` | The team's official rules and knowledge (in Japanese) | throughout |
| `sorbet/` | Sorbet type files | 8 |
| `Gemfile` | List of Ruby gems | 2 |
| `package.json` | List of JS packages + scripts | 2 |
| `CLAUDE.md` / `README.md` | Guides for developers (and AI tools) | 2 |
| `Makefile` | Shortcuts for common commands (`make generate-root-swagger`) | 15 |
| `.rubocop.yml`, `.eslintrc.js`, `.prettierrc.js` | Code style rules | 2 |
| `.circleci/` | **CI** config: automatic checks run on every PR | 2 |

> **Two backend styles live side by side** (from `README.md`):
> 1. `app/modules/` uses a **layered architecture** (organized by business domain).
> 2. Everything else is **"Rails-extended"**: normal Rails + extra folders like `services/`, `repositories/`, `values/`.
> You'll learn both. For now just remember: *both are normal here*.

---

## 5. Some Japanese domain words you'll see in code

freee-payroll uses English for general ideas but **romanized Japanese** for concepts specific to Japanese HR law (rule from `docs/knowledge/general/architecture/ubiquitous-language.md`). Examples:

| In code | Japanese | Meaning |
|---|---|---|
| `company` | 事業所 | A customer's business (the tenant) |
| `employee` | 従業員 | Employee |
| `department` | 部門 | Department (English because a normal English word exists) |
| `payroll` / `salary` / `bonus` | 給与 / 給与 / 賞与 | Payroll / monthly salary / bonus |
| `yearend_adjustment`, `nencho`, `nenmatsu_chosei` | 年末調整 | Year-end tax adjustment |
| `attendance`, `work_record` | 勤怠 / 勤務記録 | Attendance / a day's work record |
| `geppen` | 月変（月額変更届） | Social-insurance monthly-change report |
| `shaho` | 社保（社会保険） | Social insurance |
| `cps` | company_payroll_statement | A company's payroll run for one month |
| `paid_holiday` | 有給休暇 | Paid leave |

---

## 6. Practice (15 min)

**Practice 1: find it.** In the repo, open these and write one sentence about what you see (don't worry about understanding the code):

1. `.tool-versions`: which Ruby and Node versions?
2. `Gemfile`: find the line with `rails`. What version?
3. `package.json`: find `"react"`. What version?
4. `app/controllers/api/`: list the folder names you see. Which one matches `/api/p/`? (Hint: `private`)
5. `front/javascripts/`: pick three feature folder names and guess what screen each is for.

**Practice 2: classify.** For each item, write *frontend*, *backend*, or *database*:
- a) `app/models/comment.rb`  b) `front/javascripts/employees/`  c) the `employees` table  d) `config/routes.rb`  e) a `.tsx` file

**Practice 3: read a request.** Look at this and say in words what's happening:

```
GET /api/p/employee_display_names/1
→ 200 OK
{ "id": 1, "display_name": "給与太郎" }
```

*(Answers for practice: 2 = a) backend, b) frontend, c) database, d) backend, e) frontend. 3 = "The frontend asked the Private API to read the display name of employee #1; the server answered successfully with JSON containing id 1 and name 給与太郎.")*

---

## 7. Summary

- Browser (**client**) sends **HTTP requests** to the **server**; the server sends **responses** with a **status code** and usually **JSON**.
- **Frontend** = `front/` (TypeScript + React). **Backend** = `app/`, `lib/`, `config/`, `db/` (Ruby + Rails).
- Data lives in **MySQL tables**; almost every row belongs to a **company** (`company_id`).
- `/api/p/` = **Private API** used by our own frontend.
- `app/modules/` = layered architecture; the rest of `app/` = Rails-extended. Both are normal.

➡️ Now take **Test1.md**.
