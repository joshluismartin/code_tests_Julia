# Test 2: Your Tools (Terminal, Git, and Reading a Repo)

**Rules:** Don't look at Lesson2.md while answering. Write your answers under each question. Then check `Answers2.md`.
**Total:** 38 points

---

## Part A: Multiple choice (1 point each)

**1.** What does `pwd` do?
- a) Changes your password
- b) Prints the folder you are currently in
- c) Lists the files in a folder
- d) Prints a file

**2.** Which path is **absolute**?
- a) `app/models/comment.rb`
- b) `../spec`
- c) `/Users/joshualuisb.martin/Josh_Files/freee-payroll/Gemfile`
- d) `./bin/rspec`

**3.** What is a *commit*?
- a) A saved snapshot of changes with a message
- b) A request for teammates to review code
- c) A separate line of work
- d) A test

**4.** What is a *Pull Request (PR)*?
- a) Downloading new commits
- b) A request on GitHub to review a branch and merge it into `main`
- c) A Git conflict
- d) A deployment

**5.** "Trunk-based development" means:
- a) Long-lived feature branches that merge once a year
- b) Short-lived branches merged often into `main`, with unfinished work hidden behind feature flags
- c) Everyone commits directly to production
- d) Each developer uses their own fork

**6.** What is the difference between `Gemfile` and `Gemfile.lock`?
- a) There is no difference
- b) `Gemfile` lists wanted gems and version rules; `Gemfile.lock` records the exact versions installed
- c) `Gemfile.lock` is for JavaScript
- d) `Gemfile.lock` is edited by hand to add gems

**7.** A *linter* is:
- a) A tool that runs your program
- b) A tool that reads code and warns about style problems or likely bugs without running it
- c) A database
- d) A Git command

**8.** Which tool is the **Ruby formatter** used in freee-payroll?
- a) Prettier
- b) ESLint
- c) syntax_tree (`stree`)
- d) Vitest

**9.** What does *CI* do?
- a) Automatically runs tests/linters/type checks on every PR
- b) Writes code for you
- c) Stores the database
- d) Translates Japanese docs

**10.** What does the gitmoji 🐛 at the start of a commit message mean?
- a) New feature
- b) Bug fix
- c) Refactor
- d) Security

**11.** In a Gemfile, `gem 'kaminari', '~> 0.17.0'` means:
- a) Exactly 0.17.0 only
- b) Any version
- c) 0.17.0 or newer, but below 0.18
- d) Version 17 or higher

---

## Part B: Fill in the blanks (1 point each blank)

**12.** The command to search *inside* files recursively with line numbers is `grep ___ "text" app`.

**13.** `..` means the ______ folder; `~` means your ______ folder.

**14.** In freee-payroll, branch names must start with your ______ followed by `/`.

**15.** Most `CLAUDE.md` files in this repo only contain `@README.md`, which means: go read the ______ in the same folder.

**16.** The Ruby test tool is ______ and the TypeScript test tool is ______.

**17.** The Ruby type checker is ______; the TypeScript type checker is ______.

---

## Part C: Complete the command (1 point each)

**18.** Run only the test file `spec/models/banner_spec.rb`:
```bash
bin/_____ spec/models/banner_spec.rb
```

**19.** List ALL files in the current folder, including hidden ones, with details:
```bash
ls ___
```

**20.** Count how many files are in `app/models` (combine two commands with a pipe):
```bash
ls app/models _ ______
```

---

## Part D: Name → path (2 points each)

Using the Rails naming rule, write the file path for each class.

**21.** `EmployeeTimeClock` (a model)

**22.** `Api::Private::CommentsController`

**23.** `HrMaster::Infrastructure::Repositories::MemberRepository` (lives under `app/modules/`)

**24.** The **test file** for `app/controllers/api/private/comments_controller.rb`

---

## Part E: Answer in a sentence or two (2 points each)

**25.** Why does freee-payroll ask you to prefix branch names with your username?

**26.** You see this grep output. What exactly does it tell you?
```
config/constants.rb:51:DEFAULT_DELETED_AT = Time.utc(1000, 1, 1)
```

---

## Part F: Read the real repo (3 points)

**27.** Here are real lines from `git log --oneline`:
```
bafa698fbee Merge pull request #95687 from C-FO/hr-kyuyo-romu/HRHDK-336
c258970e191 ♻️ [NEN-2495] レビュー指摘対応: メッセージ定数名を HOUSING_LOAN_COPY_EXTENSION_* に改名
```
For each line, explain: what is the hash, what kind of change it is, and what the other parts (`#95687`, `hr-kyuyo-romu/HRHDK-336`, `[NEN-2495]`) mean.
