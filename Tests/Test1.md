# Test 1 — How Web Apps Work + A Map of freee-payroll

**Rules:** Don't look at Lesson1.md while answering. Write your answers under each question. Then check `Answers1.md`.
**Total:** 30 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *repository*? b
- a) A database table
- b) A project folder whose history of changes is tracked by Git
- c) A server that stores files
- d) A list of Ruby gems

**2.** Which part of freee-payroll runs *inside the browser*? c
- a) Backend
- b) Database
- c) Frontend
- d) Migration

**3.** Which HTTP method is used to *read / fetch* data? b
- a) `POST`
- b) `GET`
- c) `DELETE`
- d) `PATCH`

**4.** A response has status code `404`. What does it mean? c
- a) Success
- b) The server crashed
- c) The thing you asked for doesn't exist
- d) You need to go to a different URL

**5.** A response has status code `500`. Whose fault is it most likely? b
- a) The user's
- b) The server's (a bug)
- c) The browser's
- d) Nobody's; it means success

**6.** In the URL `https://example.com/api/p/employees?year=2026`, what is `?year=2026`? c
- a) The domain
- b) The HTTP method
- c) The query string
- d) The status code

**7.** What is a *foreign key*? b
- a) A password for the database
- b) A column that points at a row in another table (e.g. `company_id`)
- c) The unique id of a row
- d) A key from another country

**8.** What's the difference between a *library* and a *framework*? b
- a) There is no difference
- b) You call a library; a framework calls your code and gives you a structure to fill in
- c) Libraries are for Ruby, frameworks are for JavaScript
- d) A framework is always smaller than a library

**9.** In Ruby, installable packages are called: c
- a) modules
- b) npm packages
- c) gems
- d) crates

**10.** Which URL prefix is freee-payroll's **Private API** (used by its own React frontend)? c
- a) `/api/v1/`
- b) `/api/internal/`
- c) `/api/p/`
- d) `/private/`

**11.** In freee-payroll's code, what does `company` refer to? a
- a) freee K.K. itself
- b) 事業所: one customer's business (the tenant)
- c) A department
- d) An employee's employer in their resume

**12.** What file extension does a React component written in TypeScript usually have? c
- a) `.rb`
- b) `.ts`
- c) `.tsx`
- d) `.erb`

---

## Part B — Fill in the blanks (1 point each blank)

**13.** Frontend and backend exchange data using the __?____ format, which uses `{ }` for objects and `[ ]` for arrays.

**14.** In a database table, one item (for example one employee) is a __row____, and one attribute (for example `last_name`) is a __column____.

**15.** The unique number identifying each row, usually called `id`, is the __foreign____ key.

**16.** Ruby dependencies are listed in the file named __Gemfile____; JavaScript dependencies are listed in __package.json____.

**17.** The folder where freee-payroll's Ruby tests live is __tests____; the folder with all the frontend code is __front/____.

**18.** Newer backend code organized by business area (payroll, attendance, hr_master...) lives in `app/___database___/`.

---

## Part C — Match the folder to its purpose (1 point each)

Write the letter next to each number.

| # | Folder | | Letter | Purpose | (I placed my answer right after the numbers)
|---|---|---|---|---|
| 19 | B `config/routes.rb` | | A | Database change scripts |
| 20 | C `app/models/` | | B | Map of which URL goes to which controller |
| 21 | A `db/migrate/` | | C | Ruby classes that represent database tables |
| 22 | D `app/controllers/` | | D | Receive HTTP requests and return responses |
| 23 | E `docs/knowledge/` | | E | The team's official rules and knowledge |

---

## Part D — Answer in a sentence or two (2 points each)

**24.** In your own words, explain what happens when you click "Show employee" in freee人事労務, using the words *browser*, *request*, *server*, *database*, *response*.

- The client (browser) makes a request to fetch data from the sevrer's database. They get a response and the data is shown.

**25.** Why does almost every table in freee-payroll have a `company_id` column?

- Because the database is shared across several customers/companies. The company id shows indicates which id belongs to which customer.

**26.** freee-payroll has two backend "styles." Name them and say where each lives.

- oof, idk

---

## Part E — Read it (3 points)

**27.** Explain every part of this exchange:

```
PATCH /api/p/employees/42
Body: { "last_name": "山田" }

→ 422 Unprocessable Entity
{ "message": "姓は20文字以内で入力してください" }
```

(What was the client trying to do? To which thing? Did it work? Why/why not?)

- The client was trying to update an employee's last name. However, it did not work because the validation failed.
