# Upstream: PESU Academy

pesu-auth works by driving PESU Academy's web login and scraping HTML. PESU can change that HTML at
any time without notice; most production incidents will start here. All of it lives in
`app/pesu.py`.

## The flow

| Step                  | Operation label | Request                                                                                                                                                                                               | What we read                                                                        |
| --------------------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 1. Pre-login token    | `csrf_fetch`    | `GET https://www.pesuacademy.com/Academy/`                                                                                                                                                            | `meta[name='csrf-token']` `content`                                                 |
| 2. Login              | `login`         | `POST https://www.pesuacademy.com/Academy/j_spring_security_check`, form `_csrf`, `j_username`, `j_password`                                                                                          | Failure if `div.login-form` is present; otherwise the new `meta[name='csrf-token']` |
| 3. Profile (optional) | `profile_fetch` | `GET https://www.pesuacademy.com/Academy/s/studentProfilePESUAdmin` with `menuId=670`, `url=studentProfilePESUAdmin`, `controllerMode=6414`, `actionType=5`, `id=0`, `selectedData=0`, `_=<epoch ms>` | See below                                                                           |

The client is `httpx2.AsyncClient(follow_redirects=True, timeout=10.0)`. Cookies (the session) live
on the client, which is why each login gets its own client and why the prefetched client is handed
to exactly one request.

## Profile page parsing

- Container `div.elem-info-wrapper`, then its `div.form-group` nodes; **at least 7**, else
  `ProfileParseError` (`reason="page_structure"`).
- For each of the first 7 nodes: key from `label.lbl-title-light`, value from
  `label.lbl-title-light + label`. Missing key/value → `ProfileParseError` (`key_missing` /
  `value_missing`).
- Keys are mapped by `PESUAcademy.PROFILE_PAGE_HEADER_TO_KEY_MAP`: `Name`→`name`, `PESU Id`→`prn`,
  `SRN`→`srn`, `Program`→`program`, `Branch`→`branch`, `Semester`→`semester`,
  `Section`→`section`. An unknown key → `ProfileParseError` (`unknown_field`). That is deliberate: a
  renamed label means the page changed, and failing loudly beats returning wrong data.
- `email` from `#updateMail`'s `value`, `phone` from `#updateContact`'s `value` (optional).
- `campusCode` from the PRN (`PES1...` → 1/`RR`, `PES2...` → 2/`EC`); any other digit is logged,
  counted (`unknown_campus_code`) and left out, not fatal.
- An empty profile → `ProfileParseError` (`no_data`).
- Field filtering happens after parsing, only if `fields` differs from `DEFAULT_FIELDS`.

## Recognising a PESU-side change

Signals, in the order they usually appear:

- `profile_parse_errors_total{reason}` rising, or 422s on `/authenticate` with `profile: true`.
- `CSRFTokenError` in `errors_total{type}` (502s): the token tag moved or the page is an error page.
- `upstream_responses_total{status}` showing non-200s, or `upstream_requests_total{outcome="error"}`
  rising: PESU is down or blocking us; nothing in our code changed.
- Every login failing with 401 while credentials are right: the failure marker (`div.login-form`)
  now appears on success, or the form fields changed.

To investigate safely, use the test account from `.env` only, one session at a time, and save
**redacted** HTML (strip names, numbers, emails) to reason about selectors. Never commit a real
profile page. See the `update-scraper` skill.

## Verifying against the real site

- `uv run pytest tests/functional -v` and `uv run pytest tests/integration -v` with `TEST_*`
  credentials: both log in for real (one run at a time).
- Or run the app (`uv run python -m app.app --debug`) and POST to `/authenticate` with the test
  account.
- `scripts/benchmark/unauthenticated_csrf_token_expiry.py` measures how long a prefetched token
  stays valid; it informs `CSRF_TOKEN_REFRESH_INTERVAL_SECONDS` (45 minutes today).

## Etiquette

PESU Academy is a university system we do not own. Keep request volume minimal: one CSRF fetch per
login, no retries in loops, no polling. Benchmarks against a local instance still hit PESU for real;
keep `--num-requests` small.
