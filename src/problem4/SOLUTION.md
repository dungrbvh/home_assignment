## What problems I found

1. `nginx` was proxying `/api/` traffic to `api:3001`, but the API process listens on port `3000`. This caused consistent `502 Bad Gateway` responses for API calls.
2. Startup ordering was too weak (`depends_on` only), so services could start before dependencies were actually ready. That creates intermittent failures during boot and restarts.
3. The root response from Nginx used `application/octet-stream` (default type), which makes browsers download a file instead of rendering plain text.

## How I diagnosed them

- Ran `docker compose up --build` and then tested endpoints with `curl`.
- Confirmed `/` returned `200`, but `/api/users` returned `502`.
- Reviewed Nginx and API configs and found upstream port mismatch (`3001` vs `3000`).
- Reviewed Compose dependencies and saw no health checks to gate startup on readiness.

## The fixes I applied

- Updated Nginx upstream from `http://api:3001` to `http://api:3000`.
- Added `default_type text/plain;` so the root message renders correctly.
- Added health checks for `postgres`, `redis`, and `api`.
- Updated `depends_on` to `condition: service_healthy` so service startup waits for real readiness.
- Added `restart: unless-stopped` on all services for better recovery.
- Removed obsolete Compose `version` key to eliminate warning noise.

## What monitoring/alerts I would add

- Synthetic checks for `/` and `/api/users` every minute.
- Alert on non-2xx/3xx rate for Nginx and `5xx` spikes.
- Container health status alerting (unhealthy or restart loops).
- DB/Redis connectivity error rate alert from API logs.

## How I would prevent this in production

- Add CI smoke tests that boot Compose and verify key endpoints.
- Enforce health checks and explicit readiness dependencies in all service templates.
- Keep service ports centralized in env/config to avoid drift between app and proxy.
- Add pre-deploy canary checks before full rollout.