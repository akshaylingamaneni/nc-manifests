Supabase self-host quick setup (Kubernetes)

Namespaces and conventions follow the existing repo: everything lives in the `services` namespace, secrets are synced from Infisical.

What’s included
- Postgres (`supabase/postgres:17.6.1.029`)
- Auth (GoTrue), REST (PostgREST), Realtime, pg-meta, Studio, Kong router
- Storage component is currently disabled (commented out) and the Kong /storage route is disabled.
- Ingress at `supabase.neurocollab.in` and Studio at `supabase-studio.neurocollab.in`

Apply manifests
1) Create the Infisical-managed secret mapping:
   - `kubectl apply -f services/supabase-secrets-infisical.yml`
2) Deploy Supabase services:
   - `kubectl apply -f services/supabase.yml`
   - `kubectl apply -f services/supabase-ingress.yml`

DNS
- Point `supabase.neurocollab.in` and `supabase-studio.neurocollab.in` to your ingress controller IP.

Required secrets (Infisical → project `uio-bd-ns`, env `prod`, path `/supabase`)
- `SUPABASE_DB_PASSWORD`             Postgres password for user `postgres`
- `SUPABASE_DB_URL`                  e.g. `postgresql://postgres:<pass>@supabase-db.services.svc.cluster.local:5432/postgres`
- `SUPABASE_JWT_SECRET`              Random HS256 secret (32+ bytes)
- `SUPABASE_ANON_KEY`                JWT signed with `SUPABASE_JWT_SECRET` and claim `role=anon`
- `SUPABASE_SERVICE_ROLE_KEY`        JWT signed with `SUPABASE_JWT_SECRET` and claim `role=service_role`
- `REALTIME_SECRET_KEY_BASE`         Random 64+ char string (Phoenix secret)
// Optional (only if you re-enable Storage)
- `S3_ACCESS_KEY_ID`                 SeaweedFS S3 access key
- `S3_SECRET_ACCESS_KEY`             SeaweedFS S3 secret key

Generating values
- Database password: any strong string. Example: `openssl rand -base64 24`
- JWT secret: `openssl rand -hex 32`
- Static JWTs for anon/service_role (no external deps, python stdlib):

  python3 - <<'PY'
import base64, json, time, hmac, hashlib, os

def b64url(b):
    return base64.urlsafe_b64encode(b).rstrip(b'=')

secret = os.environ.get('JWT_SECRET', 'change-me')
now = int(time.time())
exp = now + 10*365*24*3600  # ~10 years

def make_token(role):
    header = b64url(json.dumps({"alg":"HS256","typ":"JWT"}).encode())
    payload = b64url(json.dumps({
        "role": role,
        "iss": "supabase",
        "iat": now,
        "exp": exp
    }).encode())
    signing_input = header + b'.' + payload
    sig = hmac.new(secret.encode(), signing_input, hashlib.sha256).digest()
    token = signing_input + b'.' + b64url(sig)
    return token.decode()

print("ANON_KEY=", make_token("anon"))
print("SERVICE_ROLE_KEY=", make_token("service_role"))
PY

Usage:
- Run with `JWT_SECRET=<your SUPABASE_JWT_SECRET> python3 script_above.py`
- Paste outputs into Infisical for `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY`.

Notes
- Storage is disabled for now. To enable later, uncomment the `supabase-storage` Deployment/Service in `services/supabase.yml` and the `/storage/v1` Kong route in the ConfigMap.
- GoTrue is set to `GOTRUE_MAILER_AUTOCONFIRM=true` for signups without SMTP. Set to `false` and supply SMTP envs when ready.
- Kong is declarative and routes `/auth/v1`, `/rest/v1`, `/realtime/v1`, `/storage/v1` under `supabase.neurocollab.in`.

Listing files in Studio / Storage UI
- Files uploaded via Supabase Storage API appear in Studio because metadata is stored in Postgres.
- Existing objects already in the S3 bucket (uploaded outside Supabase) won’t auto‑appear; they need importing (e.g., small script to list from S3 and upload through Storage API so metadata is created).
