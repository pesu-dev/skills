# API contract

What callers depend on, and the rules for changing it. The README (`README.md`, section "How to use
the PESUAuth API") is the public statement of this contract, Swagger at `/` is the machine-readable
one, and `tests/unit/test_openapi_docs.py` keeps the two honest. A change to any of the three must
update the others.

## Endpoints

| Route           | Method | Tag            | Success                                             | Errors                            | Notes                                                         |
| --------------- | ------ | -------------- | --------------------------------------------------- | --------------------------------- | ------------------------------------------------------------- |
| `/`             | GET    | (Swagger UI)   | 200 HTML                                            |                                   | `docs_url="/"`                                                |
| `/authenticate` | POST   | Authentication | 200 `ResponseModel`                                 | 400, 401, 422, 500, 502           | Body `RequestModel`                                           |
| `/health`       | GET    | Monitoring     | 200 `{status: true, message: "ok", timestamp}`      | 500                               | Must stay unauthenticated: uptime monitors and Render call it |
| `/metrics`      | GET    | Monitoring     | 200 Prometheus text (default) or JSON (`?fmt=json`) | 400 (bad `fmt`), 401 (token), 500 | Protected only when `METRICS_TOKEN` is set                    |
| `/readme`       | GET    | Documentation  | 308 to `https://github.com/pesu-dev/auth`           | 500                               |                                                               |
| `/openapi.json` | GET    |                | 200                                                 |                                   | Generated, with the phantom-422 override                      |

## `/authenticate`

Request (`app/models/request.py`, `strict=True`, `extra="forbid"`, camelCase aliases):

| Field      | Type                         | Default           | Rules                                                           |
| ---------- | ---------------------------- | ----------------- | --------------------------------------------------------------- |
| `username` | str                          | required          | Stripped; empty is a 400. SRN, PRN, email or phone              |
| `password` | str                          | required          | Stripped; empty is a 400                                        |
| `profile`  | bool                         | `false`           | Strict bool: `"true"` is a 400                                  |
| `fields`   | `list[ProfileField]` or null | null (all fields) | Non-empty; each value must be in `ProfileField` (`app/pesu.py`) |

Response (`app/models/response.py`): `status`, `message`, `timestamp` (ISO 8601 with `+05:30`), and
`profile` (`app/models/profile.py`) only when requested and the login succeeded. `None` values are
omitted (`exclude_none=True`), so a field that could not be scraped is **absent**, not null.

Profile fields (`ProfileField` Literal, `ProfileModel`, README `ProfileObject` table, the docs
examples in `app/docs/authenticate.py`, and the live tests must all agree): `name`, `prn`, `srn`,
`program`, `branch`, `semester`, `section`, `email`, `phone`, `campusCode` (1 or 2), `campus`
(`RR` or `EC`).

## Error body and status codes

Every non-2xx that this API renders is:

```json
{"status": false, "message": "<human readable>", "timestamp": "2024-07-28T22:30:10.103368+05:30"}
```

| Code    | Meaning                                                       | Produced by                                                                                   |
| ------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 400     | Request failed validation (body or query)                     | `validation_exception_handler`; message `Could not validate request data - <loc>: <msg>; ...` |
| 401     | Wrong credentials, or missing/wrong metrics token             | `AuthenticationError`, `MetricsAuthorizationError` (+ `WWW-Authenticate: Bearer`)             |
| 404/405 | Unknown path or method                                        | Starlette router, `{"detail": ...}` body (not ours)                                           |
| 422     | Profile page could not be parsed: PESU changed their page     | `ProfileParseError`                                                                           |
| 500     | Our bug, or a response that failed `ResponseModel` validation | catch-all handler; generic message                                                            |
| 502     | PESU Academy unreachable or answered unexpectedly             | `CSRFTokenError`, `ProfileFetchError`                                                         |

This API never returns FastAPI's default 422 validation body. Do not reintroduce it.

## How documentation is enforced

`tests/unit/test_openapi_docs.py` fails the build unless, for every route in the schema:

- it is documented, tagged, and has a summary and a description (the route docstring);
- it documents a success response and a 500;
- every documented response has an example and a schema;
- JSON examples validate against the model they name (`ResponseModel` or `MetricsModel`);
- request examples validate against `RequestModel` and cover the username forms and field filtering;
- documented 400/401/200 examples match what the running app actually returns;
- no phantom `HTTPValidationError` 422 appears, and the real profile-parse 422 is kept.

So a new route needs an `ApiDocs` module in `app/docs/`, exported from `app/docs/__init__.py`, passed
as `responses=` (and `openapi_extra=` for request examples), plus `tags=[...]` and a docstring.
See the `add-endpoint` skill.

## Compatibility and versioning policy

The project version in `pyproject.toml` is the API version (it is published in `/openapi.json`
`info.version` and shown on the README badges).

- **Every PR bumps the version exactly once** (enforced by `version-check.yaml`).
- **Minor** (`x.Y.0`) is the default for any change, including fixes and docs.
- **Major** (`X.0.0`) for anything backwards-incompatible: removing or renaming a request or response
  field, changing a type, changing a status code a caller could branch on, rejecting input that used
  to be accepted, or changing the default of `fmt`. Mark the commit `feat!:`/`fix!:` and call it out
  in the PR. Precedents: v3.0.0 moved response keys to camelCase (#151); v4.0.0 deprecated "Know
  Your Class and Section" (#155).
- Additive changes are compatible: a new optional request field, a new response field (callers must
  tolerate unknown keys), a new route, a new metric.

## Deprecation pattern

When a field is removed, `extra="forbid"` makes old payloads fail loudly with a 400 naming the key,
instead of being silently ignored. Keep regression tests that send the removed keys and assert the
400 message (see the `*_deprecated_*` and `*_removed_kycas_*` tests in
`tests/integration/test_app_integration.py`). Removing a profile field from `ProfileField` likewise
turns requests for it into a 400; add a test for that too.
