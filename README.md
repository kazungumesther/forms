# Students & Courses API

A small REST API one can practise on: list data, create it, change it, delete it, and follow the relationship between students and the courses they are enrolled in.

There is nothing to install beyond the project's own dependencies — no database, no accounts, no API keys.

## Running it

**1. Install the project's dependencies.** Run this once, in the project folder. It reads [package.json](package.json) and downloads everything listed there into `node_modules`.

```bash
npm install
```

**2. Start the server.** Leave this running — it reloads by itself whenever you save a file.

```bash
npm run dev
```

**3. Check the address.** The terminal prints something like `Local: http://localhost:3000`. The API is at that address with `/api` on the end:

**`http://localhost:3000/api`**

If port 3000 was already taken, Next.js picks another one (3001, 3002…). Use whatever the terminal actually says.

**4. Make your first request.** Open a *second* terminal — the first one is busy running the server — and run:

```bash
curl http://localhost:3000/api/students
```

You should get a wall of JSON back. If instead you see `Connection refused`, the server is not running or is on a different port. You can also just open <http://localhost:3000/api/students> in a browser.

To stop the server, press `Ctrl+C` in the terminal running it.

### Where the data lives

The first request creates a `data` folder at the root of the project with `students.json` and `courses.json` in it, copied from [src/lib/seed.js](src/lib/seed.js). After that, those two files are the real data — anything you POST, PATCH or DELETE is written to them and is still there after you restart the server.

**To reset everything back to the starting data, delete the `data` folder** and make any request. The folder is gitignored, so your experiments never show up in `git status`.

## Reading a response

Every response has the same shape.

**One thing** — `GET /api/students/1`

```json
{ "data": { "id": 1, "firstName": "Amina", "lastName": "Otieno", "courseIds": [1, 3] } }
```

**A list** — `GET /api/students`

```json
{
  "data": [ { "id": 1, "...": "..." } ],
  "meta": { "page": 1, "limit": 10, "total": 8, "totalPages": 1 }
}
```

**Anything that went wrong** — any `4xx` response:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Some fields are invalid.",
    "fields": { "email": ["email must look like name@example.com."] }
  }
}
```

So in JavaScript there is one rule to remember — what you want is always under `data`:

```js
const response = await fetch("http://localhost:3000/api/students");
const { data, meta } = await response.json();

console.log(data);            // the array of students
console.log(meta.totalPages); // how many pages there are
```

And when something fails, read the body — it tells you exactly what was wrong:

```js
const response = await fetch("http://localhost:3000/api/students", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ firstName: "A", lastName: "Njeri", email: "not-an-email" }),
});

if (!response.ok) {
  const { error } = await response.json();
  console.log(error.fields); // { firstName: [...], email: [...] }
}
```

### Status codes

| Code | Meaning |
| --- | --- |
| `200` | It worked. |
| `201` | Something new was created. |
| `400` | A value you sent is wrong. `error.fields` says which. |
| `404` | There is nothing with that id. |
| `405` | That method is not allowed on that path. |
| `409` | Your value is fine, but it clashes with data that already exists — a duplicate email, a duplicate course code, or a student already enrolled. |

The difference between **400 and 409** is worth remembering: 400 means *this value is not acceptable*, 409 means *this value is fine, but it is taken*.

`DELETE` returns `200` with the thing that was deleted, rather than an empty `204`, so `await response.json()` works the same everywhere.

## The data

A **course**:

```json
{
  "id": 1,
  "code": "WEB101",
  "title": "Intro to Web Development",
  "instructor": "Jane Doe",
  "credits": 3,
  "createdAt": "2026-01-05T09:00:00.000Z",
  "updatedAt": "2026-01-05T09:00:00.000Z"
}
```

A **student**, who holds the ids of the courses they are enrolled in:

```json
{
  "id": 1,
  "firstName": "Amina",
  "lastName": "Otieno",
  "email": "amina.otieno@example.com",
  "courseIds": [1, 3],
  "createdAt": "2026-01-05T09:00:00.000Z",
  "updatedAt": "2026-01-05T09:00:00.000Z"
}
```

`id`, `createdAt` and `updatedAt` are set by the API. Sending them is harmless — they are ignored.

## Query parameters

These work on every endpoint that returns a list.

| Parameter | Default | What it does |
| --- | --- | --- |
| `page` | `1` | Which page to return. |
| `limit` | `10` | How many per page, from 1 to 100. |
| `search` | — | Case-insensitive. Students match on name and email, courses on code, title and instructor. |
| `include` | — | `include=courses` on a student endpoint adds the full course objects. |

```bash
curl "http://localhost:3000/api/students?page=2&limit=3"
curl "http://localhost:3000/api/students?search=amina"
curl "http://localhost:3000/api/courses?search=web"
```

A bad `page` or `limit` is a `400` rather than being quietly corrected, so a typo is easy to spot.

## Endpoints

### Students

#### `GET /api/students`

```bash
curl "http://localhost:3000/api/students?page=1&limit=5&search=mwangi"
```

#### `GET /api/students/:id`

Add `?include=courses` to get the full course objects alongside `courseIds`.

```bash
curl "http://localhost:3000/api/students/1?include=courses"
```

#### `POST /api/students` → `201`

`firstName`, `lastName` and `email` are required. `courseIds` is optional and defaults to `[]`.

```bash
curl -X POST http://localhost:3000/api/students \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Njeri",
    "lastName": "Kamande",
    "email": "njeri.kamande@example.com",
    "courseIds": [1, 2]
  }'
```

#### `PUT /api/students/:id`

Replaces the student — send every field.

```bash
curl -X PUT http://localhost:3000/api/students/1 \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Amina",
    "lastName": "Otieno-Odhiambo",
    "email": "amina.otieno@example.com",
    "courseIds": [1]
  }'
```

#### `PATCH /api/students/:id`

Changes only the fields you send.

```bash
curl -X PATCH http://localhost:3000/api/students/1 \
  -H "Content-Type: application/json" \
  -d '{ "lastName": "Otieno-Odhiambo" }'
```

#### `DELETE /api/students/:id`

```bash
curl -X DELETE http://localhost:3000/api/students/8
```

### Enrolment

#### `GET /api/students/:id/courses`

The courses one student is taking. Supports `page`, `limit` and `search`.

```bash
curl http://localhost:3000/api/students/1/courses
```

#### `POST /api/students/:id/courses` → `201`

Enrols the student in one more course and returns the updated student.

```bash
curl -X POST http://localhost:3000/api/students/1/courses \
  -H "Content-Type: application/json" \
  -d '{ "courseId": 2 }'
```

`409` if they are already enrolled, `400` if the course does not exist.

#### `DELETE /api/students/:id/courses/:courseId`

Unenrols the student. The course itself is not touched.

```bash
curl -X DELETE http://localhost:3000/api/students/1/courses/2
```

#### `GET /api/courses/:id/students`

The same relationship read from the other side — everyone taking one course.

```bash
curl http://localhost:3000/api/courses/1/students
```

### Courses

#### `GET /api/courses` and `GET /api/courses/:id`

```bash
curl http://localhost:3000/api/courses
curl http://localhost:3000/api/courses/3
```

#### `POST /api/courses` → `201`

All four fields are required.

```bash
curl -X POST http://localhost:3000/api/courses \
  -H "Content-Type: application/json" \
  -d '{
    "code": "API200",
    "title": "Consuming REST APIs",
    "instructor": "Grace Wambui",
    "credits": 4
  }'
```

#### `PUT` / `PATCH` `/api/courses/:id`

```bash
curl -X PATCH http://localhost:3000/api/courses/3 \
  -H "Content-Type: application/json" \
  -d '{ "credits": 4 }'
```

#### `DELETE /api/courses/:id`

Deleting a course also **unenrols everyone from it**, so no student is left pointing at a course that no longer exists. The response says how many students were affected:

```bash
curl -X DELETE http://localhost:3000/api/courses/4
```

```json
{ "data": { "id": 4, "code": "UX110", "...": "..." }, "meta": { "unenrolledStudents": 2 } }
```

## What gets validated

| Field | Rule |
| --- | --- |
| `firstName`, `lastName` | Text, 2–50 characters. |
| `email` | Looks like `name@example.com`, and no two students may share one. |
| `courseIds` | An array of whole numbers, and every course must exist. Repeats are ignored. |
| `code` | Like `WEB101` — 2 to 6 letters then 2 to 4 numbers. No two courses may share one. |
| `title` | Text, 3–100 characters. |
| `instructor` | Text, 2–60 characters. |
| `credits` | A whole number from 1 to 6. |

Text is trimmed, emails are lowercased and course codes uppercased before being saved. Fields the API does not know about are ignored rather than rejected.

## Calling it from a browser app

CORS is open (`Access-Control-Allow-Origin: *`), so a page on any address — a React app on `localhost:5173`, an HTML file opened directly, an online sandbox — can call this API without extra setup.

```js
const BASE = "http://localhost:3000/api";

// read
const { data: students } = await fetch(`${BASE}/students?include=courses`).then((r) => r.json());

// write
const response = await fetch(`${BASE}/students`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    firstName: "Njeri",
    lastName: "Kamande",
    email: "njeri.kamande@example.com",
  }),
});

const payload = await response.json();
if (response.ok) {
  console.log("created", payload.data);
} else {
  console.log(payload.error.message, payload.error.fields);
}
```

Don't forget `Content-Type: application/json` on writes — without it the body is not read as JSON.

## How it is built

```text
src/lib/seed.js          the starting data
src/lib/db.js            reads and writes the JSON files
src/lib/http.js          response shapes, CORS, query parsing
src/lib/validation.js    the field rules
src/lib/relations.js     connecting students to courses
src/app/api/...          one route.js per URL

src/components/forms/    the forms - not written yet, see below
```

Every route talks to the data only through the functions exported by [src/lib/db.js](src/lib/db.js). Moving to a real database later means rewriting that one file and nothing else.

## Next: the forms

The API above is only half the job. The other half is a UI that talks to it — a form to add a student, a form to edit a course, and so on. Those go in `src/components/forms/`, one file per form, and they are built with **[React Hook Form](https://react-hook-form.com/)**.

**It is not installed yet — install it yourself.** Adding a package to a project is part of the exercise, so here is the whole loop:

**1. Stop the dev server** with `Ctrl+C`, or open a second terminal. Installing while the server runs usually works, but restarting afterwards avoids odd errors.

**2. Install the package**, from the project folder (the one with `package.json` in it):

```bash
npm install react-hook-form
```

**3. Look at what changed.** Open [package.json](package.json). There is now a new line under `"dependencies"`:

```json
"react-hook-form": "^7.x.x"
```

`npm install <name>` does two things: it downloads the code into `node_modules`, and it records the package in `package.json`. That second part is the important one. `node_modules` is gitignored and never committed — so when a classmate clones this repo, all they get is `package.json`, and their `npm install` rebuilds `node_modules` from that list. A package that is only on your machine and not in `package.json` will break for everyone else.

**4. Commit `package.json` and `package-lock.json`** together. The lock file pins the exact version everyone gets.

**5. Start the server again** with `npm run dev`, and import it in a component.

Useful to know:

- `npm install` with no arguments installs everything already listed — that's step 1 of *Running it* above.
- `npm install --save-dev <name>` (or `-D`) puts a package under `devDependencies` instead: tools you need while building, but that the running app does not. React Hook Form ships to the browser as part of the app, so it is a normal dependency, not a dev one.
- `npm uninstall react-hook-form` reverses all of this.

A form component built with it is a client component, so it starts with `"use client"`, and it sends what the user typed to the endpoints documented above:

```jsx
"use client";

import { useForm } from "react-hook-form";

export function AddStudentForm() {
  const { register, handleSubmit, setError, formState } = useForm();

  async function onSubmit(values) {
    const response = await fetch("http://localhost:3000/api/students", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(values),
    });

    const payload = await response.json();

    if (!response.ok) {
      // The API returns { error: { fields: { email: ["..."] } } }, so a failed
      // save can be shown next to the input that caused it.
      for (const [field, messages] of Object.entries(payload.error.fields ?? {})) {
        setError(field, { message: messages[0] });
      }
      return;
    }

    console.log("created", payload.data);
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("firstName", { required: "firstName is required." })} />
      <p>{formState.errors.firstName?.message}</p>

      <button type="submit" disabled={formState.isSubmitting}>
        Save
      </button>
    </form>
  );
}
```

Two kinds of checking are going on there, and both are needed:

- **In the form** (`required`, and the other rules you pass to `register`) — instant feedback, no network round trip. It is for the person typing.
- **In the API** ([src/lib/validation.js](src/lib/validation.js)) — the real gate. A browser can be bypassed entirely with `curl`, so the server can never trust what it is sent. It is for the data.

Because the API reports failures per field, the two line up: `error.fields` maps straight onto React Hook Form's `setError`, and a `409` (duplicate email) can be shown on the email input exactly like a `400`.

## Things worth knowing

- **This will not work deployed to Vercel or Netlify.** Serverless hosts have a read-only filesystem, so every write fails. It runs locally, or on a host that keeps a normal Node process alive (`next start` on a VPS, Render, Railway, Fly). To deploy it properly, swap `src/lib/db.js` for a real database.
- **It is not safe for many writers at once.** Writes are queued inside a single server process, which is enough for a class. Two servers writing the same `data` folder can still overwrite each other.
- **There is no authentication.** Anyone who can reach the URL can change anything. That is deliberate — it keeps the focus on learning HTTP.
- **Ids get reused.** They are "the biggest id so far, plus one", so deleting the last student and creating a new one gives the new one the same id. Don't treat an id as permanent.
