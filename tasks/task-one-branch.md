# Task One (Backend): Branch Master CRUD

> This task is **synced** with `frontend/tasks/task-one-branch.md`.
> Both files share the same **API Contract** (section 2). Do not change the contract on your own. If it must change, update BOTH files and tell your partner.

## 1. Goal

Build the backend for the **Branch Master** (System Setup > Master Data > Branches). The backend should let the admin **create, list, view, update and delete** branches, and the frontend will call these APIs.

No frontend work here. Only model, controller, routes and wiring.

## 2. API Contract (shared with frontend)

Base URL: `http://localhost:5000/api/branches`

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/api/branches` | List all branches |
| GET | `/api/branches/:id` | Get one branch |
| POST | `/api/branches` | Create a branch |
| PUT | `/api/branches/:id` | Update a branch |
| DELETE | `/api/branches/:id` | Delete a branch |

### Branch fields

| Field | Type | Rules |
|-------|------|-------|
| `_id` | ObjectId | Auto from Mongo (this is our "uuid") |
| `branch_code` | String | Required, **unique**, trimmed, stored UPPERCASE (e.g. `CSE`) |
| `branch_name` | String | Required, trimmed |
| `capacity` | Number | Required, integer, min 1. Meaning: annual intake capacity |
| `status` | String | Enum `Active` / `Inactive`, default `Active` |
| `createdAt`, `updatedAt` | Date | Auto via Mongoose `timestamps: true` |

> The POST/PUT body accepts only `branch_code`, `branch_name`, `capacity`, `status`. Any other field is ignored.
>
> **The branch table has no HOD field.** It is kept stable. HOD assignment lives in a separate mapping table (`batch_hod_map`) that will be built in a later task. Do not add `hod_id` to the model.

### Response format (always use this shape)

Success:
```json
{ "success": true, "message": "Branch created successfully", "data": { ...branch } }
```
List success: `data` is an array, and add `"count": <number>`.

Error:
```json
{ "success": false, "message": "Branch code already exists" }
```
For validation errors you may add `"errors": { "capacity": "Capacity must be at least 1" }` (field name -> message) so the frontend can show inline errors.

### Status codes

| Case | Code |
|------|------|
| OK (get, update, delete, list) | 200 |
| Created | 201 |
| Validation failed / bad id | 400 |
| Branch not found | 404 |
| Duplicate `branch_code` | 409 |
| Unexpected error | 500 |

## 3. Folder structure to create

Inside `backend/` create (this is the pattern every later module will follow):

```
backend/
├── server.js                    (edit: connect DB + mount routes)
├── .env                         (create, never commit)
├── .env.example                 (create, commit this)
├── config/
│   └── db.js                    (MongoDB connection)
├── models/
│   └── Branch.js                (schema/model)
├── controllers/
│   └── branchController.js      (logic for each endpoint)
├── routes/
│   └── branchRoutes.js          (URL -> controller mapping)
├── middleware/
│   └── errorHandler.js          (optional but recommended: 404 + central error handler)
└── tasks/
    └── task-one-branch.md
```

## 4. Step by step

### Step 1: Environment and DB connection
1. Create `.env` with `PORT=5000` and `MONGO_URI=<your mongodb connection string>`. Create `.env.example` with the same keys and empty values. Confirm `.env` is in `.gitignore`.
2. Create `config/db.js`. It exports a function `connectDB` that calls `mongoose.connect(process.env.MONGO_URI)`, logs success, and on failure logs the error and exits the process.

### Step 2: Model (`models/Branch.js`)
1. Define the schema with the fields from section 2.
2. Mongo collection name must be **`rgukt_sklm_master_branch`**. Set it explicitly with the `collection` option on the schema (or the third argument of `mongoose.model`), because Mongoose would otherwise pluralize the name.
3. Add schema-level rules: `required` with clear messages, `unique: true` and `uppercase: true` + `trim: true` on `branch_code`, `min: 1` on `capacity`, `enum` on `status`, `timestamps: true`.
4. Make sure `capacity` must be an integer (use a `validate` function).

### Step 3: Controller (`controllers/branchController.js`)
Create one exported function per endpoint. Keep each one small.

- **getBranches**: return all branches sorted by `branch_code`. Respond with `count`.
- **getBranchById**: validate that `:id` is a valid ObjectId (`mongoose.isValidObjectId`), else 400. If not found, 404.
- **createBranch**: pick only the 4 allowed fields from `req.body` (do not pass `req.body` straight into `create`). Create and return 201.
- **updateBranch**: validate id, pick the allowed fields, use `findByIdAndUpdate` with `{ new: true, runValidators: true }`. 404 if not found.
- **deleteBranch**: validate id, delete, 404 if not found.

Error handling inside each function (`try/catch`):
- Mongo duplicate key error (`error.code === 11000`) -> 409 "Branch code already exists".
- Mongoose `ValidationError` -> 400 with the `errors` object (field -> message).
- Anything else -> 500.
Write ONE small helper in the controller file for this mapping so you don't repeat it five times.

> Note for later (do not build now): once Batches and Courses exist, deleting a branch that is in use must be blocked. Leave a `// TODO` comment in `deleteBranch`.

### Step 4: Routes (`routes/branchRoutes.js`)
Create an Express `Router`, map the 5 endpoints from section 2 to the controller functions, and export it. `GET/POST` go on `/`, and `GET/PUT/DELETE` go on `/:id`.

### Step 5: Wire up in `server.js`
1. Call `connectDB()` before `app.listen`.
2. Mount: `app.use('/api/branches', branchRoutes)`.
3. Add a 404 handler and the central error handler after the routes (from `middleware/errorHandler.js`).
4. Keep `cors()` and `express.json()` as they are.

> Express 5 note: async errors thrown in route handlers are forwarded automatically, but we still use try/catch in controllers so we control the response shape.

## 5. How to test (before telling your partner it's done)

Use Postman, Thunder Client or curl. Verify:

- [ ] POST a valid branch (`CSE`, `Computer Science and Engineering`, `120`) -> 201
- [ ] POST `cse` lowercase -> saved as `CSE`
- [ ] POST same code again -> 409
- [ ] POST with missing name or capacity `0` / `-5` / `10.5` -> 400 with `errors`
- [ ] POST with `status: "Banana"` -> 400
- [ ] POST with extra fields (e.g. `hod_id`, `foo`) -> ignored, not saved
- [ ] GET list -> array sorted by code, with `count`
- [ ] GET `/:id` valid -> 200, random valid id -> 404, `abc` -> 400
- [ ] PUT changes name/capacity/status -> 200 with updated data
- [ ] PUT changing code to an existing other code -> 409
- [ ] DELETE -> 200, DELETE again -> 404
- [ ] In MongoDB, the collection is named `rgukt_sklm_master_branch`

## 6. Seed data (add these via POST so the frontend has something to show)

| branch_code | branch_name | capacity | status |
|---|---|---|---|
| CSE | Computer Science and Engineering | 120 | Active |
| ECE | Electronics and Communication Engineering | 120 | Active |
| EEE | Electrical and Electronics Engineering | 60 | Active |
| MECH | Mechanical Engineering | 60 | Active |
| CIVIL | Civil Engineering | 60 | Active |
| CHEM | Chemical Engineering | 60 | Inactive |

## 7. Dependencies

Already installed: `express`, `mongoose`, `cors`, `dotenv`, `nodemon`. **No new packages needed.**

## 8. Definition of done

- All checks in section 5 pass.
- Folder structure matches section 3.
- `.env` is not committed, `.env.example` is.
- Response shapes match the contract exactly.
- Tell the frontend partner the server is running and share the seed data.
