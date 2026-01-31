# Hasura DDN Schema Update Guide

This guide documents the process for updating your Hasura GraphQL schema when the upstream Neon PostgreSQL database schema changes.

## TL;DR - Quick Steps

### Local Development
```bash
# 1. Introspect the database to get latest schema
ddn connector introspect neon_mtbl

# 2. Start the connector service
docker compose -f compose.yaml --env-file .env up -d app_neon_mtbl

# 3. Update connector link and add all new resources
ddn connector-link update neon_mtbl --add-all-resources

# 4. Build the supergraph with new metadata
ddn supergraph build local

# 5. Rebuild and restart all services
docker compose down
docker compose -f compose.yaml --env-file .env build engine
docker compose -f compose.yaml --env-file .env up -d

# 6. Verify the update
curl -X POST http://localhost:3280/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { queryType { fields { name } } } }"}'
```

### Deploy to Cloud
```bash
# After local testing is successful, deploy to Hasura DDN cloud
# Option 1: Create and apply in one step (recommended for production)
ddn supergraph build create --apply

# Option 2: Create build first, then apply separately
ddn supergraph build create
ddn supergraph build apply <build-version>

# Test the STABLE production endpoint (this URL never changes!)
curl -X POST 'https://oriented-mosquito-4344.ddn.hasura.app/graphql' \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ leagues { leagueId seasonId } }"}'
```

## Detailed Step-by-Step Process

### Prerequisites

- Docker Desktop must be running
- `ddn` CLI installed (`ddn update-cli` to update)
- Access to Neon PostgreSQL database (connection URI in `.env`)

### Step 1: Introspect the Database

This command connects to your Neon database and updates the connector configuration with the latest schema (tables, columns, relationships, etc.).

```bash
ddn connector introspect neon_mtbl
```

**What it does:**
- Scans all tables, columns, types, and relationships in the database
- Updates `app/connector/neon_mtbl/configuration.json` with the new schema
- Does NOT require Docker to be running for the introspection itself

**Expected output:**
```
Checking Postgres DB connectivity...
Postgres DB connectivity check successful
Connector data source introspected successfully
```

**Troubleshooting:**
- If you get "Docker daemon not running" - that's OK at this stage, the introspection succeeds before the build fails
- The important part is that `configuration.json` gets updated

### Step 2: Start the Connector Service

The connector needs to be running for the next step to fetch the schema.

```bash
docker compose -f compose.yaml --env-file .env up -d app_neon_mtbl
```

**What it does:**
- Builds the connector Docker image with the updated `configuration.json`
- Starts the PostgreSQL connector service on port 8003
- Loads the new schema into the connector

**Expected output:**
```
Container mtblq-app_neon_mtbl-1  Creating
Container mtblq-app_neon_mtbl-1  Started
```

**Troubleshooting:**
- Wait 5-10 seconds after starting for the connector to be fully ready
- Check logs: `docker logs mtblq-app_neon_mtbl-1`
- Connector must be healthy before proceeding

### Step 3: Update Connector Link and Generate Metadata

This command fetches the schema from the running connector and generates all the Hasura metadata files.

```bash
ddn connector-link update neon_mtbl --add-all-resources
```

**What it does:**
- Connects to the running connector at `http://local.hasura.dev:8003`
- Fetches the full NDC (Native Data Connector) schema
- Updates `app/metadata/neon_mtbl.hml` with the connector schema
- Generates metadata files for:
  - **Models** (GraphQL queries/subscriptions for each table)
  - **Commands** (Insert/Update/Delete mutations for each table)
  - **Relationships** (Foreign key relationships between tables)
- Creates files like `Leagues.hml`, `Teams.hml`, etc. in `app/metadata/`

**Expected output:**
```
DataConnectorLink "neon_mtbl" updated successfully
Adding all Models, Commands and Relationships...
  GENERATE Model Leagues
  GENERATE Model Teams
  GENERATE Model Matchups
  ...
  GENERATE Relationship (Leagues.teams)
  ...
Models, Commands and Relationships added successfully
```

**Troubleshooting:**
- If you get "connection refused", the connector isn't running yet (go back to Step 2)
- If you get "DataConnectorLink schema already up to date" but no new models are generated, the connector may not have reloaded the configuration - rebuild it:
  ```bash
  cd app/connector/neon_mtbl && docker compose build && cd ../../..
  docker compose restart app_neon_mtbl
  ```

### Step 4: Build the Supergraph

This compiles all the metadata into optimized build artifacts that the engine will use.

```bash
ddn supergraph build local
```

**What it does:**
- Reads all `.hml` files from `app/metadata/` and `globals/metadata/`
- Validates the metadata (checks for errors, missing fields, etc.)
- Generates optimized build artifacts in `engine/build/`:
  - `metadata.json` - Full metadata for the engine
  - `open_dd.json` - OpenDD format metadata
  - `auth_config.json` - Authentication configuration

**Expected output:**
```
Supergraph built for local Engine successfully
Build artifacts exported to "engine/build"
```

**Troubleshooting:**
- **Validation errors about missing columns**: This happens when old metadata references columns that no longer exist in the database
  - Solution: Delete the affected metadata files and regenerate:
    ```bash
    rm -f app/metadata/Players.hml app/metadata/InsertPlayers.hml
    ddn connector-link update neon_mtbl --add-all-resources
    ddn supergraph build local
    ```
- **Type errors**: The schema may have changed in incompatible ways - check the database schema

### Step 5: Rebuild and Restart All Services

The engine Docker image bakes in the metadata files, so it needs to be rebuilt.

```bash
# Stop all services
docker compose down

# Rebuild the engine image with new metadata
docker compose -f compose.yaml --env-file .env build engine

# Start all services
docker compose -f compose.yaml --env-file .env up -d
```

**What it does:**
- Stops all running containers
- Rebuilds the `engine` image, copying `engine/build/*` into the image
- Starts all services:
  - `app_neon_mtbl` - PostgreSQL connector
  - `engine` - Hasura GraphQL engine (port 3280)
  - `otel-collector` - OpenTelemetry collector

**Expected output:**
```
Container mtblq-engine-1  Created
Container mtblq-app_neon_mtbl-1  Created
Container mtblq-otel-collector-1  Created
...
Container mtblq-engine-1  Started
```

**Important:**
- The engine must be **rebuilt**, not just restarted, because the metadata is baked into the Docker image
- If you just restart without rebuilding, the old schema will still be served

### Step 6: Verify the Update

Check that the new tables are available in the GraphQL API.

```bash
# List all available queries
curl -X POST http://localhost:3280/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __schema { queryType { fields { name } } } }"}' \
  | python3 -m json.tool

# Test a specific query
curl -X POST http://localhost:3280/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ leagues { leagueId seasonId numTeams } }"}' \
  | python3 -m json.tool
```

**Expected output:**
- You should see all your new tables in the query list
- Queries should return data from your database

## Deploying to Hasura DDN Cloud

After updating your local schema successfully, deploy to the cloud environment.

### Understanding Build URLs vs Production URL

Hasura DDN provides two types of endpoints:

1. **Build-specific URLs** (for testing/preview):
   - Format: `https://oriented-mosquito-4344-<BUILD_VERSION>.ddn.hasura.app/graphql`
   - Each build gets its own unique URL
   - Use these to test new builds before applying them to production

2. **Stable Production URL** (for your app):
   - Format: `https://oriented-mosquito-4344.ddn.hasura.app/graphql`
   - **This URL never changes!**
   - Points to whichever build you've "applied"
   - Use this in your production applications

### Deployment Workflow

#### Option 1: Create and Apply (Recommended for Production)
```bash
# Single command to build and apply to production
ddn supergraph build create --apply
```

**What it does:**
- Builds the supergraph for cloud
- Uploads connector images and metadata
- Creates a new build version
- **Immediately applies it to your stable production URL**

**Expected output:**
```
+---------------+---------------------------------------------------------------------------+
| Build Version | 612bb411e0                                                                |
+---------------+---------------------------------------------------------------------------+
| API URL       | https://oriented-mosquito-4344.ddn.hasura.app/graphql                     |
+---------------+---------------------------------------------------------------------------+
| Console URL   | https://console.hasura.io/project/oriented-mosquito-4344/build/612bb411e0 |
+---------------+---------------------------------------------------------------------------+

SupergraphBuild created and applied successfully
```

#### Option 2: Create, Test, Then Apply (Safer for Production)
```bash
# Step 1: Create a build (gets a preview URL)
ddn supergraph build create

# Output shows:
# Build Version: abc123def
# API URL: https://oriented-mosquito-4344-abc123def.ddn.hasura.app/graphql

# Step 2: Test the preview build
curl -X POST 'https://oriented-mosquito-4344-abc123def.ddn.hasura.app/graphql' \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ leagues { leagueId } }"}'

# Step 3: If tests pass, apply it to production
ddn supergraph build apply abc123def

# Now your stable URL serves the new build:
# https://oriented-mosquito-4344.ddn.hasura.app/graphql
```

### Your Production Endpoint

**Use this URL in your application code:**
```
https://oriented-mosquito-4344.ddn.hasura.app/graphql
```

This stable URL will always point to your currently applied build, so you never need to update your frontend/app code when you deploy schema changes!

**Verify Production Deployment:**
```bash
curl -X POST 'https://oriented-mosquito-4344.ddn.hasura.app/graphql' \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ leagues { leagueId seasonId numTeams } }"}'
```

### Rollback Strategy

If you need to rollback to a previous build:

```bash
# List all builds
ddn changelog list

# Apply an older build version
ddn supergraph build apply <older-build-version>
```

The production URL immediately switches to serve the older build!

### Best Practices

1. **Always test locally first** before creating cloud builds
2. **Use preview URLs** to test cloud builds before applying to production
3. **Keep your stable URL** (`https://oriented-mosquito-4344.ddn.hasura.app/graphql`) in your app
4. **Never hardcode build-version URLs** in production code
5. **Document build versions** with changelogs for tracking changes

**Important Notes:**
- Build-specific URLs remain accessible forever (great for debugging old versions)
- You can create unlimited preview builds without affecting production
- Only one build can be "applied" at a time
- Applying a new build instantly updates the production endpoint

## Common Issues and Solutions

### Issue: "NDC Collection not found" warnings

**Symptom:**
```
WARNING NDC Collection not found for pattern leagues
```

**Cause:** The connector hasn't loaded the new schema from `configuration.json`

**Solution:**
```bash
# Rebuild the connector image
cd app/connector/neon_mtbl
docker compose build
cd ../../..

# Restart the connector
docker compose restart app_neon_mtbl

# Try the update again
ddn connector-link update neon_mtbl --add-all-resources
```

### Issue: New tables don't appear in GraphQL schema

**Symptom:** After all steps, `curl` shows the old schema with only a few queries

**Cause:** The engine wasn't rebuilt, so it's still using old metadata

**Solution:**
```bash
# Must rebuild the engine image
docker compose down
docker compose build engine
docker compose up -d
```

### Issue: Validation errors about unknown columns

**Symptom:**
```
error: unknown target column name display_name for field displayName
```

**Cause:** Database schema changed (columns renamed/removed) but old metadata still references them

**Solution:**
```bash
# Delete the affected metadata files
rm -f app/metadata/Players.hml app/metadata/InsertPlayers.hml app/metadata/UpdatePlayersByIdEspn.hml

# Regenerate them
ddn connector-link update neon_mtbl --add-all-resources

# Rebuild
ddn supergraph build local
docker compose down
docker compose build engine
docker compose up -d
```

### Issue: "DataConnectorLink schema already up to date" but nothing changes

**Cause:** The connector is serving a cached schema that doesn't match `configuration.json`

**Solution:**
```bash
# Force a fresh introspection and rebuild
rm -f app/connector/neon_mtbl/schema.json
ddn connector introspect neon_mtbl

# Rebuild connector from scratch
docker compose down
cd app/connector/neon_mtbl
docker compose build --no-cache
cd ../../..

# Start connector and update
docker compose -f compose.yaml --env-file .env up -d app_neon_mtbl
sleep 10
ddn connector-link update neon_mtbl --add-all-resources
ddn supergraph build local
docker compose build engine
docker compose up -d
```

## File Structure Reference

```
.
├── app/
│   ├── connector/
│   │   └── neon_mtbl/
│   │       ├── configuration.json    # Database schema (updated by introspect)
│   │       ├── connector.yaml        # Connector configuration
│   │       └── compose.yaml          # Connector Docker setup
│   ├── metadata/                     # GraphQL metadata (auto-generated)
│   │   ├── neon_mtbl.hml            # Connector link schema
│   │   ├── neon_mtbl-types.hml      # Type definitions
│   │   ├── Leagues.hml              # Leagues model
│   │   ├── Teams.hml                # Teams model
│   │   └── ...                       # One file per table/command
│   └── subgraph.yaml                # Subgraph config (includes metadata/)
├── engine/
│   ├── build/                        # Built metadata (generated)
│   │   ├── metadata.json            # Engine metadata
│   │   ├── open_dd.json             # OpenDD format
│   │   └── auth_config.json         # Auth config
│   └── Dockerfile.engine            # Engine image (COPYs build/)
├── compose.yaml                      # Docker Compose for all services
├── supergraph.yaml                   # Supergraph config
└── .env                              # Environment variables (DB connection, etc.)
```

## Understanding Hasura DDN Architecture

1. **PostgreSQL Database (Neon)** - Your source of truth
2. **Connector** (`neon_mtbl`) - Translates between Postgres and Hasura's NDC protocol
3. **Metadata** (`app/metadata/*.hml`) - Defines what's exposed in GraphQL
4. **Build** (`engine/build/*`) - Compiled metadata artifacts
5. **Engine** - Serves the GraphQL API using the built metadata

**The flow:**
```
Neon DB → introspect → configuration.json → connector → connector-link update
→ metadata/*.hml → supergraph build → engine/build/* → engine Docker image → GraphQL API
```

## Tips

- **Always introspect first** when the database schema changes
- **Always rebuild the engine** after changing metadata (just restarting won't work)
- **Use git diff** to review what changed before committing:
  ```bash
  git diff app/connector/neon_mtbl/configuration.json
  git status app/metadata/
  ```
- **Test locally first** before deploying to cloud
- **Keep metadata in git** - it's your GraphQL schema definition

## Managing Downstream Consumers

### The Short Answer
**Use the stable production URL in all your applications:**
```
https://oriented-mosquito-4344.ddn.hasura.app/graphql
```

When you deploy a new schema, your apps automatically get the updates without code changes!

### Detailed Strategy

#### For Frontend Applications
```javascript
// ✅ CORRECT - Use stable URL
const GRAPHQL_ENDPOINT = 'https://oriented-mosquito-4344.ddn.hasura.app/graphql';

// ❌ WRONG - Don't hardcode build versions
const GRAPHQL_ENDPOINT = 'https://oriented-mosquito-4344-abc123.ddn.hasura.app/graphql';
```

#### Deployment Workflow with Consumers

1. **Create and test a preview build:**
   ```bash
   ddn supergraph build create
   # Test with preview URL: https://oriented-mosquito-4344-<BUILD>.ddn.hasura.app/graphql
   ```

2. **Coordinate with frontend team** (if needed):
   - Share preview URL for integration testing
   - Validate breaking changes
   - Update GraphQL queries if schema changed

3. **Apply to production when ready:**
   ```bash
   ddn supergraph build apply <build-version>
   ```

4. **Consumers automatically receive updates** through the stable URL!

#### For Mobile Apps

Mobile apps can't be updated instantly, so be careful with breaking changes:

```bash
# Good practice: Add new fields without removing old ones
# This allows old app versions to keep working

# When you must make breaking changes:
# 1. Deploy new fields alongside old ones
# 2. Release new app version
# 3. After most users update, remove old fields in next deploy
```

#### For External API Consumers

If you provide your GraphQL API to external partners:

```bash
# Give them the stable URL:
https://oriented-mosquito-4344.ddn.hasura.app/graphql

# For testing/staging environments, give them preview URLs:
https://oriented-mosquito-4344-staging.ddn.hasura.app/graphql
```

#### CI/CD Integration

```yaml
# GitHub Actions example
env:
  GRAPHQL_ENDPOINT: https://oriented-mosquito-4344.ddn.hasura.app/graphql

# Only update this when you want to point to a different environment
```

### Schema Evolution Best Practices

1. **Additive Changes** (Safe - no consumer updates needed):
   - Adding new tables/types
   - Adding new fields to existing types
   - Adding new queries/mutations

2. **Deprecation Strategy** (For breaking changes):
   - Mark fields as deprecated first
   - Give consumers time to migrate
   - Remove deprecated fields in a later release

3. **Version Communication**:
   ```bash
   # Use changelogs when applying builds
   ddn supergraph build apply <version> \
     --changelog-body "## Added\n- New player stats tables\n\n## Deprecated\n- Old stats field" \
     --published
   ```

## Helpful Commands

```bash
# View all tables in the connector config
jq -r '.metadata.tables | keys[]' app/connector/neon_mtbl/configuration.json

# Count models in metadata
ls app/metadata/*.hml | wc -l

# Check if services are running
docker compose ps

# View connector logs
docker logs mtblq-app_neon_mtbl-1 --tail 50

# View engine logs
docker logs mtblq-engine-1 --tail 50

# Open local console (browser UI)
ddn console --local

# List all builds
ddn changelog list

# View current applied build
ddn project get

# Update DDN CLI
ddn update-cli
```

## Quick Reference

### URLs You Need to Know

| Environment | URL | Use For |
|------------|-----|---------|
| **Production** | `https://oriented-mosquito-4344.ddn.hasura.app/graphql` | Your app/frontend/mobile (stable, never changes) |
| **Preview Builds** | `https://oriented-mosquito-4344-<BUILD>.ddn.hasura.app/graphql` | Testing before applying to production |
| **Local Dev** | `http://localhost:3280/graphql` | Local development and testing |
| **Console** | `https://console.hasura.io/project/oriented-mosquito-4344` | Web UI for managing your project |

### Common Workflows

**Daily Development:**
```bash
# Make database changes in Neon
# Then update local Hasura:
ddn connector introspect neon_mtbl
docker compose up -d app_neon_mtbl
ddn connector-link update neon_mtbl --add-all-resources
ddn supergraph build local
docker compose build engine && docker compose up -d
```

**Deploy to Production:**
```bash
# After local testing passes:
ddn supergraph build create --apply
```

**Emergency Rollback:**
```bash
# Find previous working build
ddn changelog list

# Apply it
ddn supergraph build apply <old-build-version>
```

---

**Last Updated:** January 31, 2026
**Production Endpoint:** `https://oriented-mosquito-4344.ddn.hasura.app/graphql`
