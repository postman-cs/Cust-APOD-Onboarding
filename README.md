# APOD - Run Postman collections from GitLab CI

A GitLab pipeline that logs in to Postman with the Postman CLI and runs the
collections in a Postman workspace against that workspace's **mock server** -
either **every collection** in the workspace or **the designated collections**
you list.

It only runs collections that already exist in the workspace. It does not
create workspaces, sync files, or onboard specs.

**Where it runs.** These are GitLab CI files. They are kept in this repository
as a handoff copy and do not run here: GitHub does not read `.gitlab-ci.yml`.
Copy `.gitlab-ci.yml` and the `.gitlab/` folder into the GitLab project that
will run them.

## Current state

Validated on the Postman side against the APOD workspace and its mock server
(not yet run in an independent GitLab environment):

- Resolving the workspace's collections and mock environment, running all
  collections, running a chosen list of collections, and failing on an unknown
  collection ID all behaved as described in this README.
- Of the three collections in that workspace, the baseline and Smoke collections
  pass. The Contract collection **fails** on 6 assertions (2 each on 3
  requests): "Error and encoding conventions match RFC 9457 / RFC 8259 /
  RFC 8288" and "Response satisfies RFC 9110 range, negotiation, and cache
  conventions". These assertions are evaluated against the mock's responses.
  The pipeline is reporting them as intended; the cause has not been
  investigated, and the owner of the spec and mock decides how to address them.
- Not tested: a Postman team hosted in the EU region, and listing the same
  collection ID twice (duplicates are ignored by the code).

```
postman:resolve  ->  postman:collections (child pipeline)
 look up workspace      one job per collection: install CLI, log in,
 collections + mock     warm up the mock, `postman collection run`
 environment
```

## What you need before the first run

| Requirement | Detail |
| --- | --- |
| GitLab project + runner | Any runner that can run the `node:22-bookworm` image and reach the internet (see below). GitLab.com shared runners work. |
| `POSTMAN_API_KEY` | A Postman API key (a service-account key is best) with access to the workspace. Add it under **Settings > CI/CD > Variables** as **Masked** (use **Masked and hidden** if offered). Leave **Protected** off if you want branch pipelines to run, since protected variables are not available to unprotected branches. |
| A Postman workspace | It must contain the collections you want to run. Set its ID in `POSTMAN_WORKSPACE_ID`. |
| A mock environment | The workspace must have an environment whose `baseUrl` is a Postman mock URL (`https://<id>.mock.pstmn.io`). The value must be **synced to the cloud** (a local-only value is invisible to the API). The pipeline will not invent a mock URL. If there is none, or more than one, it fails and lists each environment's ID, name and `baseUrl`; set `POSTMAN_ENVIRONMENT_ID` to choose one. |
| A running mock server | Bound to that environment. If the mock is private, the API key is passed to it as `postmanPrivateMockApiKey`. |

Network access from the runner: `dl-cli.pstmn.io` (CLI install),
`api.getpostman.com` / Postman's API, and the mock host `*.mock.pstmn.io`.
The resolver calls the US API endpoint (`https://api.getpostman.com`), which is
hard-coded in `.gitlab/postman/resolve-resources.mjs`. An EU-hosted Postman team
would need that endpoint changed, and this has not been tested.

## Creating the Postman credential and adding it to GitLab

The pipeline authenticates with one secret, a Postman API key. Use a key that
belongs to a **service account**, not a person, so runs don't depend on one
individual's account.

1. **Get a service account.** A Postman team admin creates one by following
   Postman's [service accounts guide](https://learning.postman.com/docs/administration/service-accounts/).
   Make sure it has access to the workspace the pipeline will run.
2. **Generate its API key.** Generate an API key for that service account (see
   [managing API keys](https://learning.postman.com/docs/administration/managing-your-team/managing-api-keys/)
   for expiration and revocation). Copy the key as soon as it is shown; Postman
   may not show it again.
3. **Add it to GitLab.** In your GitLab project go to **Settings > CI/CD >
   Variables > Add variable** and set:
   - **Key:** `POSTMAN_API_KEY`
   - **Value:** the key you copied
   - **Visibility:** *Masked* (or *Masked and hidden* if offered), so it never
     appears in job logs
   - **Protected:** off if branch pipelines should run, since protected
     variables are only available on protected branches and tags
   - **Expand variable reference:** off, so a `$` in the key is not interpolated
4. **Never commit the key.** Don't put it in `.gitlab-ci.yml`, the README, a
   shell history or a chat. A *Masked and hidden* variable can't be viewed
   again after saving, so keep the key in your password manager if you may need
   it later.
5. **Rotate it** per your organization's secret policy: generate a new key,
   update the variable, then revoke the old key.

To check the setup, run the pipeline. If `postman:resolve` fails with a message
like `GET /workspaces/... returned HTTP <status>`, check that the key is
correct, has not expired and belongs to an account with access to that
workspace.

## How authentication works

Each run job installs the Postman CLI and authenticates it with
`postman login --with-api-key "$POSTMAN_API_KEY"`. The resolver calls the
Postman API with the same key. The key is read only from the masked CI
variable and is never written to the job log or to artifacts.

## Variables

Set defaults in `.gitlab-ci.yml`, or override per run under
**Build > Pipelines > Run pipeline**.

| Variable | Default | Purpose |
| --- | --- | --- |
| `POSTMAN_WORKSPACE_ID` | `1707f60c-e080-4213-9fa2-d11c19bd3b01` | Workspace whose collections and environments are used. |
| `POSTMAN_COLLECTION_IDS` | empty | **Empty = run all collections** in the workspace (no name filter). **Set = run only these collection UIDs**, as a comma-separated list. |
| `POSTMAN_ENVIRONMENT_ID` | empty | Environment UID to use. Empty = the one environment whose `baseUrl` is a `*.mock.pstmn.io` URL. |
| `POSTMAN_RESPONSE_TIME_THRESHOLD` | `5000` | Passed to the collections as `RESPONSE_TIME_THRESHOLD` (ms). |

## Running

**All collections (default):** push to a branch, or run the pipeline manually
with `POSTMAN_COLLECTION_IDS` left empty.

**Designated collections:** run the pipeline manually and set
`POSTMAN_COLLECTION_IDS` to one or more collection UIDs, comma-separated, for
example `58347765-2ecd07bb-...,58347765-6081c4dd-...`. A UID is the full
`<owner>-<uuid>` value shown in Postman's collection info or in the API. Spaces
after commas, empty entries and duplicates are ignored. Every listed UID must be
in `POSTMAN_WORKSPACE_ID`: if any is not, the pipeline fails before running
anything, names the missing UIDs and lists the workspace's collections, so a
typo can never pass as a partial run.

## Jobs

- **`postman:resolve`** queries the Postman API for the collections to run and
  for the mock environment, then generates `collection-runs.gitlab-ci.yml`.
  It fails with a clear message if the workspace has no collections, the
  designated collection is not in it, or the environment is missing or
  ambiguous.
- **`postman:collections`** runs that generated file as a child pipeline with
  `strategy: depend`: one job per collection, named `postman:run:<name-slug>`,
  so you can see which collection failed. Each job installs the CLI, logs in,
  warms up the mock and runs `postman collection run`.

## Failures are meant to be visible

There is no `allow_failure`. If a collection has failing assertions, its job
fails, the child pipeline fails and the parent pipeline is red. That is the
intended signal that the spec, the mock or the collection needs attention.
Do not loosen assertions to get a green run.

## Things to know

- **Mock warm-up.** Mocks often fail the first request, and the CLI has no
  retry option, so each job first sends a few retried `curl` requests to
  `$POSTMAN_BASE_URL/apod-basic`. That path is APOD-specific; change it in
  `.gitlab/postman/cli.gitlab-ci.yml` for another API. Failures here are
  ignored.
- **Requests with absolute URLs bypass the mock.** A request that hard-codes a
  full URL (the APOD RSS feed does) calls that URL directly, not the mock.
- **Run results upload to Postman.** The jobs use `--report-events`, so each run
  also appears in the workspace's run history.

## Files

| File | Purpose |
| --- | --- |
| `.gitlab-ci.yml` | Entry point: variables and the two jobs. |
| `.gitlab/postman/cli.gitlab-ci.yml` | Shared building blocks: CLI install and login, the resolver job, and the per-collection run job. |
| `.gitlab/postman/resolve-resources.mjs` | Node script (no dependencies) that calls the Postman API and writes the environment values and the generated child pipeline. |
