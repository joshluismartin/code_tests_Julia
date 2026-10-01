# Test 14 — Routing and Controllers

**Rules:** Don't look at Lesson14.md. Write your answers under each question, then check `Answers14.md`.
**Total:** 40 points

---

## Part A — Multiple choice (1 point each)

**1.** What is a *route*?
- a) A method inside a model
- b) A rule mapping an HTTP method + URL path to a controller action
- c) A database table of URLs
- d) A JavaScript function

**2.** An *action* is:
- a) Any private method in a controller
- b) A public method in a controller that handles a request
- c) A migration
- d) A before_action filter

**3.** `resources :comments` (without `only:`) can generate how many routes at most?
- a) 1
- b) 4
- c) 7
- d) 10

**4.** Which difference between `member` and `collection` routes is correct?
- a) member routes have `:id` in the path; collection routes don't
- b) collection routes have `:id`; member routes don't
- c) They are identical
- d) member routes are only for POST

**5.** Inside `namespace :api do scope 'p', module: :private do ... end end`, a controller for `resources :holidays` is named:
- a) `HolidaysController`
- b) `Api::HolidaysController`
- c) `Api::Private::HolidaysController`
- d) `P::HolidaysController`

**6.** *Authentication* answers which question?
- a) "Are you allowed to do this?"
- b) "Who are you / are you logged in?"
- c) "Is the JSON valid?"
- d) "Which database to use?"

**7.** What does `head :no_content` send?
- a) Status 200 with JSON
- b) Status 204 with no body
- c) Status 404
- d) An HTML page

**8.** Why do we use strong parameters (`permit`)?
- a) To make requests faster
- b) To whitelist which input keys may be used, preventing mass assignment
- c) To convert JSON to XML
- d) To log parameters

**9.** In freee-payroll, where must every new endpoint's permission be registered?
- a) `config/sekisyo.yml`
- b) `config/permission.yaml`
- c) `config/routes.rb`
- d) `Gemfile`

**10.** `get 'attendances(/*path)', to: 'web/attendances#index'` exists mainly because:
- a) Rails requires it for every page
- b) The page is a React app with its own browser-side router, so every sub-path must serve the same page
- c) It's a security check
- d) It deletes old attendances

---

## Part B — Fill in the blanks (1 point each)

**11.** The main routes file is `config/______.rb`, and it loads split files with the word `______ :web`.

**12.** `resource` (singular) routes have no ______ in the path, but the controller name is still ______ (singular/plural).

**13.** Values in `params` arrive as ______, which is why code often calls `.to_i`.

**14.** `ApplicationController#check_authenticated` just raises `______`, forcing subclasses to override it.

**15.** `helper_method :current_company` makes `current_company` usable in ______ too.

**16.** Every freee-payroll controller must include `______` or `Omega::Controller::ErrorHandler` (checked by a policy spec).

---

## Part C — Fill in the missing code (2 points each)

**17.** Make routes for only `index` and `show` of employees inside the Private API block:

```ruby
scope 'p', module: :private do
  resources :employees, ______
end
```

**18.** Add a collection route `GET /api/p/geofence_locations/summary` and a member route `POST /api/p/geofence_locations/:id/activate`:

```ruby
resources :geofence_locations, only: %i[index] do
  ______ do
    get :summary
  end
  ______ do
    post :activate
  end
end
```

**19.** Permit only `name` and an array `employee_ids`:

```ruby
def group_params
  params.______(:name, ______)
end
```

**20.** Respond with status 201 and no body after creating something:

```ruby
def create
  ...
  ______
end
```

---

## Part D — Write the code (3 points each)

**21.** Write a route and controller (illustrative) so that `GET /api/p/greetings/:id` returns JSON `{ "id": <the id as an integer>, "text": "hello" }`. Follow freee-payroll's conventions (base class, required include, private params helper).

**22.** Write a `before_action` in a controller (illustrative) that raises `HrErrors::UserError` with status `:forbidden` unless `current_company.some_feature_enabled?` is true.

---

## Part E — Answer in a sentence or two (2 points each)

**23.** What's the relationship between the URL segment `api/private/comments`, the class `Api::Private::CommentsController`, and the file path?

**24.** Why do API base controllers respond to "not logged in" with a JSON 401, while web base controllers redirect?

**25.** Why do freee-payroll controllers pass `company_id: current_company.id` to repositories instead of letting the client send `company_id`?

---

## Part F — Read real code (4 points)

**26.** Read `app/controllers/api/private/comments_controller.rb` (lines 1–59 in the repo) and `config/routes.rb:234`. Answer:
- a) List the 3 endpoints (method + URL) this controller serves.
- b) What status does a successful `create` return, and does it send a body?
- c) In `destroy`, what happens if the comment belongs to another user?
- d) What status does a successful `destroy` return?
