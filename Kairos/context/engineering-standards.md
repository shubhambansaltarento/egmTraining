# Engineering standards

## Repository layout
- backend/   API, domain logic, migrations, tests
- frontend/  single-page application
- docs/      designs and decisions

## Backend
- Layers: router -> service -> repository. Routers contain no business rules.
- All input validated at the API boundary.
- Errors use one JSON shape: { "code", "message", "details" }.
- Every endpoint has at least one happy-path and one failure test.

## Frontend
- One folder per feature. Shared components in components/.
- No direct fetch calls in components. Use the API client module.

## Commands (must exist after ST-01)
- make up      start everything locally
- make test    run all tests
- make lint    lint and type-check

## Git
- One commit per sub-task, message starts with the sub-task ID.