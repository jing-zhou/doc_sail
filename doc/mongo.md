# MongoDB — Developer Guide (Illiad project)

This single reference covers local MongoDB operations for development: installing `mongosh`, Docker usage, helpful `mongosh` queries, export/import, and the project's helper scripts (`scripts/start-mongo-docker.sh`, `scripts/docker-smoke-mongo.sh`).

Paths and defaults in this repository
- Docker Compose mongo service: `docker-compose.yml` (service `mongo`) — image: `mongo:6.0`.
- Dev DB names used in repo:
  - app / CI: `illiad_proxy` (environment SPRING_DATA_MONGODB_URI in compose)
  - developer quick-checks use `illiad` by default; pass `--db` to scripts to override.
- Local data dir (compose): `./docker/mongo/data`

Quick index
- Installing `mongosh` (host)
- Docker fallback (recommended, no host changes)
- Useful `mongosh` queries (interactive & one-liners)
- Docker-based queries and smoke test script
- Helper scripts: `scripts/start-mongo-docker.sh` and `scripts/docker-smoke-mongo.sh`
- Exporting and backups
- Troubleshooting

Installing mongosh (host)

If you prefer using `mongosh` locally on your host, install the official MongoDB Shell. On modern Ubuntu releases you may need `libssl1.1` to run certain prebuilt mongosh binaries.

Option A — manual .deb (recommended when apt repo fails)
1. Visit: https://www.mongodb.com/try/download/shell and pick Platform=Linux, Package=Debian/Ubuntu (.deb), Architecture=x86_64.
2. Download and install:

```bash
# replace <MONGOSH_DEB_URL> with the URL from mongodb.com
curl -fSL -o /tmp/mongosh.deb "<MONGOSH_DEB_URL>"
sudo dpkg -i /tmp/mongosh.deb || sudo apt-get install -f -y
mongosh --version
```

Option B — apt repo (canonical)

Add MongoDB apt repo and install (may fail if your environment blocks pgp.mongodb.com):

```bash
sudo apt-get update && sudo apt-get install -y gnupg curl ca-certificates lsb-release
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg --dearmour -o /usr/share/keyrings/mongodb-server-7.0.gpg
echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg arch=amd64] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get update
sudo apt-get install -y mongosh
mongosh --version
```

Option C — Docker (no host changes): recommended for quick checks

If you want to avoid modifying host libraries, run `mongosh` in a container. This is the recommended approach for development and CI tests.

```bash
# interactive (Linux: host networking)
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad"

# non-interactive ping
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.runCommand({ ping: 1 }))'
```

Useful mongosh queries

Interactive (recommended):

```bash
mongosh "mongodb://127.0.0.1:27017/illiad"
# then inside the shell:
# show current DB
db.getName()
# list collections
show collections
# preview documents
db.getCollection('users').find().limit(10).pretty()
```

Non-interactive one-liners (copy-paste):

```bash
# Find user by username
mongosh "mongodb://127.0.0.1:27017/illiad" --eval 'printjson(db.getCollection("users").findOne({ username: "bob" }))'

# Find user by email
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.getCollection("users").findOne({ email: "alice@example.com" }))'

# Projection: username + email for first 20
ing
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").find({}, { username:1, email:1 }).limit(20).forEach(printjson)'

# Recently created users (if using createdAt)
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").find().sort({ createdAt: -1 }).limit(20).forEach(printjson)'

# Find by _id (replace id)
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.getCollection("users").findOne({ _id: ObjectId("60a7c1d8f2a3b4c5d6e7f890") }))'

# Find users that have active sessions (two-step)
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval '
  const s = db.getCollection("sessions").distinct("userId");
  if (s && s.length) {
    print("Found session userIds:", JSON.stringify(s));
    db.getCollection("users").find({ _id: { $in: s } }).limit(50).forEach(printjson);
  } else {
    print("No session-based userIds found in sessions collection or sessions collection missing.");
  }
'

# List users with token-related fields
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").find({ $or: [{ currentTokenId: { $exists:true } }, { currentToken: { $exists:true } }] }).limit(50).forEach(printjson)'

# Count by condition (example: email domain)
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson({ count: db.getCollection("users").countDocuments({ email: /@example\\.com$/ }) })'
```

Quick repeatable check: tokenId + updatedAt

This small section contains copy-paste commands you can run quickly to verify a user's token identifier and the `updatedAt` timestamp. It provides both a host `mongosh` example and a Docker-based variant (recommended when you didn't install `mongosh` locally).

- Purpose: confirm token generation/update (tokenId/currentTokenId) and when it happened (updatedAt).
- Output: a JSON document containing whichever token field exists and the `updatedAt` field.

Host (mongosh) one-liner:

```bash
# Replace USERNAME with the username to check (e.g. imlol)
USERNAME=imlol
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval "const u = db.getCollection('users').findOne({ username: \"${USERNAME}\" }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if (u) printjson(u); else print('USER_NOT_FOUND')"
```

Docker-based one-liner (no host mongosh needed):

```bash
# Replace USERNAME=imlol as needed
USERNAME=imlol
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval "const u = db.getCollection('users').findOne({ username: \"${USERNAME}\" }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if (u) printjson(u); else print('USER_NOT_FOUND')"
```

Compact field-only output (useful for scripts):

```bash
# prints: <tokenId or currentTokenId> <updatedAt ISO string>
USERNAME=imlol
mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval "const u=db.getCollection('users').findOne({ username: \"${USERNAME}\" }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if(u) print((u.currentTokenId||u.tokenId) + ' ' + (u.updatedAt ? u.updatedAt.toISOString() : 'NO_UPDATEDAT')); else print('USER_NOT_FOUND')"
```

Reusable pattern for automation (in a CI script)

```bash
# environment variables: DB_URI (optional), USERNAME
DB_URI=${DB_URI:-"mongodb://127.0.0.1:27017/illiad"}
USERNAME=${USERNAME:-imlol}
# using docker image to avoid host dependencies
docker run --rm --network host mongo:6.0 mongosh "$DB_URI" --quiet --eval "const u=db.getCollection('users').findOne({ username: \"${USERNAME}\" }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if(!u){print('USER_NOT_FOUND'); process.exit(2);} printjson({ token: (u.currentTokenId||u.tokenId||null), updatedAt: u.updatedAt||null });"
```

Docker-based queries (using the official mongo image)

```bash
# Ping (Linux)
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.runCommand({ ping: 1 }))'

# List DBs
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017" --quiet --eval 'printjson(db.adminCommand({ listDatabases: 1 }))'

# List collections in illiad
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.getCollectionNames())'

# Print one user
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").find().limit(1).forEach(printjson)'

# Insert smoke-test user (then verify)
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").insertOne({ username: "__smoke_test__", createdAt: new Date(), test: true }); printjson(db.getCollection("users").findOne({ username: "__smoke_test__" }));'

# Remove smoke-test user
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").deleteMany({ username: "__smoke_test__" }); printjson({ removed: true });'

# Export users to host file (mount current dir)
docker run --rm --network host -v "$(pwd):/work" mongo:6.0 bash -lc "mongoexport --uri='mongodb://127.0.0.1:27017/illiad' --collection=users --out=/work/users.json --jsonArray"
```

Helper scripts in this repo

- `scripts/start-mongo-docker.sh` — idempotent helper to start a local mongo container. Defaults:
  - container name: `illiad-mongo`
  - image: `mongo:6.0`
  - host port: `27017`
  - data dir: `./docker/mongo/data`

  Usage examples:

```bash
# start default local mongo
./scripts/start-mongo-docker.sh

# start and create a dev user (DB 'illiad_proxy')
CREATE_USER=true DB_USER=radar DB_PASS=radar_pass DB_NAME=illiad_proxy ./scripts/start-mongo-docker.sh
```

- `scripts/docker-smoke-mongo.sh` — small smoke-test that inserts -> reads -> deletes a test document. Defaults to DB `illiad` and collection `users`. It auto-detects the project's mongo image from `docker-compose.yml` and falls back to `mongo:6.0`.

  Usage:

```bash
# run smoke test (uses host networking on Linux)
./scripts/docker-smoke-mongo.sh

# run against the same DB the app uses
./scripts/docker-smoke-mongo.sh --db illiad_proxy

# override image explicitly
./scripts/docker-smoke-mongo.sh --image mongo:6.0
```

Exporting / backup

```bash
# export 'users' collection to a JSON file from host (requires mongo tools installed)
mongoexport --uri="mongodb://127.0.0.1:27017/illiad" --collection=users --out=users.json --jsonArray

# or run inside container then copy out
docker exec -i illiad-mongo mongoexport --uri="mongodb://127.0.0.1:27017/illiad" --collection=users --out=/tmp/users.json --jsonArray
docker cp illiad-mongo:/tmp/users.json .
```

Troubleshooting
- If you cannot download the MongoDB GPG key or apt install `mongosh`, use the Docker approach instead.
- If `docker run --network host` fails (macOS/Windows), try `host.docker.internal` or `docker exec` into the project's `illiad-mongo` container.
- If collections appear missing, run `show collections` to check exact names (singular vs plural) and verify the DB name.

Troubleshooting: "USER_NOT_FOUND" when using Docker/mongosh

If a simple Docker-based `mongosh` one-liner prints `USER_NOT_FOUND` but you know the user exists, the most common causes are:

- Wrong database name. The project uses `illiad_proxy` for the app/CI while many quick-checks historically used `illiad`. Double-check which DB the app is using.
- You're connecting to the wrong Mongo instance (multiple mongod instances or containers running). `--network host` only works as expected on Linux; it will not reach a container that is not bound to the host network. If there are multiple containers or a local `mongod`, you may be talking to a different data directory.
- The collection or field names differ (e.g. `users` vs `user`, or `userName` vs `username`).
- Authentication is required and your one-liner silently returns nothing.
- Shell quoting/expansion issues broke the JS sent to `mongosh` (the host shell may interpret quotes or backslashes before the container sees them).

Quick checks (copy/paste)

1) Confirm which process/container is listening on port 27017 (Linux):

```bash
# shows docker containers and ports
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
# find listener on 27017
sudo ss -ltnp 'sport = :27017' || sudo netstat -ltnp | grep 27017 || true
```

2) Prefer `docker exec` into the project's mongo container (most robust)

If your project uses a container named `illiad-mongo` (or similar) it's safest to run `mongosh` inside the same container so you definitely hit the same data directory:

```bash
# find the container name first
docker ps --filter ancestor=mongo --format 'table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Status}}'
# replace <container> with the name from the previous command
docker exec -it <container> mongosh --quiet --eval 'printjson(db.getCollection("users").findOne({ username: "imlol" }, { currentTokenId:1, tokenId:1, updatedAt:1 }))'
```

3) Safer Docker `run` variants (avoid host-shell quoting pitfalls)

- Use single quotes around the `--eval` argument on the host and double quotes inside the JS literal. This avoids most shell expansions:

```bash
USERNAME=imlol
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'const u = db.getCollection("users").findOne({ username: "'"${USERNAME}"'" }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if (u) printjson(u); else print("USER_NOT_FOUND")'
```

- Or run a small shell in the container so your host shell does not interpret the JS at all:

```bash
USERNAME=imlol
docker run --rm --network host mongo:6.0 bash -lc "mongosh 'mongodb://127.0.0.1:27017/illiad' --quiet --eval \"const u = db.getCollection('users').findOne({ username: '${USERNAME}' }, { currentTokenId:1, tokenId:1, updatedAt:1 }); if (u) printjson(u); else print('USER_NOT_FOUND')\""
```

4) Cross-DB/search-for-username (robust):

This iterates all DBs and collections and prints any matching documents. It's slow on very large deployments but safe for local dev.

```bash
USERNAME=imlol
# Docker (recommended when you didn't install mongosh locally)
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017" --quiet --eval "const NAME='${USERNAME}'; const dbs = db.adminCommand({ listDatabases: 1 }).databases; for (const d of dbs) { try { const my = db.getSiblingDB(d.name); const cols = my.getCollectionNames(); for (const c of cols) { try { const u = my.getCollection(c).findOne({ username: NAME }); if (u) { print('FOUND in db='+d.name+' coll='+c); printjson(u); } } catch(e){} } } catch(e){} }"
```

5) Explicitly check both `illiad` and `illiad_proxy` DBs:

```bash
# check illiad
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'printjson(db.getCollection("users").findOne({ username: "imlol" }, { currentTokenId:1, tokenId:1, updatedAt:1 }))'
# check illiad_proxy
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad_proxy" --quiet --eval 'printjson(db.getCollection("users").findOne({ username: "imlol" }, { currentTokenId:1, tokenId:1, updatedAt:1 }))'
```

6) If the document exists but fields differ, print a full sample to inspect the schema:

```bash
# print one user doc to inspect field names
docker run --rm --network host mongo:6.0 mongosh "mongodb://127.0.0.1:27017/illiad" --quiet --eval 'db.getCollection("users").find().limit(1).forEach(printjson)'
```

Notes and common root causes

- DB name mismatch: we intentionally use `illiad_proxy` for the application; many quick-check snippets use `illiad` for convenience. If your app wrote users into `illiad_proxy` and you query `illiad`, you'll get `USER_NOT_FOUND` even though the user exists in another DB.
- Multiple mongod/mongo containers: be sure the command targets the same instance the app writes to; `docker exec` into the specific container is the most reliable way.
- Quoting/escaping: when building JS passed into `--eval`, prefer single quotes on the host and double quotes in the JS, or use `bash -lc` inside the container so the host does not mangle the string.
- Authentication: if auth is enabled you must include credentials in the URI.

If you'd like I can:
- Add a short `doc/quick-checks.md` file with the exact copy-paste commands you used and annotated results (I can capture output if you allow running docker commands here).
- Update any helper scripts under `scripts/` to default to `illiad_proxy` and to prefer `docker exec` when a named container exists.
