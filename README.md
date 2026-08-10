# codegen

A Python code generator that parses a SQL DDL file containing `CREATE TABLE`
statements and generates a **runnable prototype**: a Spring Boot backend
(JPA + springdoc) and a React + TypeScript + Tailwind frontend with CRUD pages
per table, preloaded with demo data.

The output is not just source files — it ships with `run.sh` / `run.ps1` that
build the frontend into the backend and serve the whole thing on one port.

## Quick start

```bash
pip install -e .
codegen inspect --ddl tests/fixtures/sample.sql
codegen generate --ddl tests/fixtures/sample.sql \
                 --out ./generated \
                 --package com.example.demo \
                 --app-name DemoApp
cd generated && ./run.sh
```

Then open <http://localhost:8080>.

| | |
|---|---|
| UI | <http://localhost:8080/> |
| API | <http://localhost:8080/api/> |
| Swagger UI | <http://localhost:8080/swagger-ui.html> |
| H2 console | <http://localhost:8080/h2-console> |

`run.sh --dev` runs Spring Boot on :8080 and Vite with hot reload on :5173
instead. `run.sh --backend-only` skips Node entirely.

Requires **Java 21+**, **Maven**, and **Node 18+** on the machine running the
generated app. The generated project has no Maven wrapper committed; add one
with `cd generated/backend && mvn -N wrapper:wrapper` if you want `./mvnw`.

## Demo data

`generate` writes `backend/src/main/resources/data.sql` with 15 rows per table
by default. Values are picked from the column name first (an `email` column
gets addresses, `price` gets money, `is_active` gets 0/1) and fall back to the
mapped Java type. Foreign keys reference real parent rows, and tables are
inserted parent-first.

Generation is deterministic, so regenerating the same schema produces the same
`data.sql`. It is **fake data for demos** — none of it is real.

```bash
codegen generate ... --rows 40   # more rows
codegen generate ... --rows 0    # no data.sql at all
```

Identifiers are emitted double-quoted, matching Hibernate's
`globally_quoted_identifiers`, so tables named `user`, `order` or `group` work
without renaming.

## What the generated app includes

Each table gets a JPA entity, Spring Data repository, service, DTO + mapper,
a paginated REST controller, and React list + form pages. The frontend opens
on a dashboard with per-entity row counts.

**Foreign keys.** FK columns resolved against another generated table become
real relations: the entity gains a read-only `@ManyToOne` association next to
the writable scalar id, the DTO carries a server-populated `<assoc>Label`
derived from the target's display column (`name`, `title`, `sku`, ... with
fallbacks), list pages show that label instead of the raw id, and forms render
a dropdown of target rows instead of a number input. FKs pointing outside the
schema, at composite PKs, or at non-PK columns stay plain scalars.

**List pages.** Server-side column sorting (Spring's `sort` param), a
debounced search box backed by a generated `LIKE` query over the table's
string columns, and a row count. Secret-ish columns (`password`, `secret_*`,
`token`) are excluded from search and from display-label selection.

## Verifying the output

```bash
codegen verify --out ./generated
```

Compiles the backend with Maven and builds the frontend with npm, printing a
pass/fail line per step and exiting non-zero on failure — suitable for CI.
Flags: `--offline` (Maven `-o`), `--skip-backend`, `--skip-frontend`.

## Web UI

```bash
codegen serve --host 127.0.0.1 --port 5000
```

Open <http://127.0.0.1:5000/> and the form lets you:

- upload a `.sql` DDL file (drag-and-drop supported),
- set **Project Name**, **Group Id**, **Package**, and **SEAL Id**,
- pick a **Deployment type** (Jenkins, Jules, AWS ECS, AWS EKS),
- click **Generate** and download a ZIP of the backend + frontend.

> The deployment type and SEAL id are collected and threaded through the
> generator context but no template consumes them yet.

## Running the tests

```bash
pip install -e ".[dev]"
pytest -q
```

## Project layout

```
codegen/
  cli.py            click entry point: inspect / generate / verify / serve
  ddl_parser.py     sqlglot wrapper -> RawTable list
  schema.py         Table / Column / Schema pydantic models + build_schema
  type_mapper.py    SQL -> Java / TypeScript type dictionaries
  seed.py           deterministic demo-data -> data.sql
  generator.py      Jinja2 environment + Generator orchestrator
  templates/
    java/           entity, repository, service, controller, dto, mapper,
                    pom, application.yml, OpenApiConfig, SpaWebConfig,
                    ApiExceptionHandler
    react/          list/form pages, api clients, types, shell, Vite config
    project/        run.sh, run.ps1, README, .gitignore for the output
tests/              pytest suite for parser, schema, seed, generator, web, cli
```

## Known gaps

- Regeneration overwrites unconditionally; hand edits to generated files are
  lost. Keep customisations outside the output directory for now.
- Input is SQL DDL only — no YAML business objects or requirements text yet.
- One-to-many navigation (child lists on a parent's page) is not generated;
  relations surface on the many side only.
