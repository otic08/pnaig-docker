# PNAIG CKAN Docker - AI Agent Instructions

## Project Overview
This is a Docker Compose setup for running CKAN (Comprehensive Knowledge Archive Network), an open-source data management system. The project provides both production and development environments with multiple service containers (CKAN, PostgreSQL, Solr, Redis, NGINX, DataPusher).

## Architecture

### Service Containers
- **ckan/ckan-dev**: Main CKAN application (Python 3.10, uWSGI for prod, Flask dev server for dev)
- **db**: PostgreSQL with three databases: `ckan_db`, `datastore_db`, and test databases
- **solr**: Search indexing using CKAN's pre-configured Solr 9 image
- **redis**: Session and background job management
- **nginx**: Reverse proxy with SSL (production only, port 8443)
- **datapusher**: Automated data import service

### Two Modes with Different Files
- **Production**: Use `docker-compose.yml` → container name: `ckan`, port via NGINX (8443)
- **Development**: Use `docker-compose.dev.yml` → container name: `ckan-dev`, direct port 5000

### Critical Volume Mounts (Dev Mode)
```yaml
./src:/srv/app/src_extensions  # Local extensions auto-installed
pip_cache:/root/.cache/pip
site_packages:/usr/local/lib/python3.10/site-packages  # Persists installed packages
```

## Essential Developer Workflows

### Use bin/ Scripts (Not docker compose directly)
These scripts wrap docker compose with the correct compose file:

```bash
bin/compose build              # Build dev images
bin/compose up                 # Start dev containers
bin/ckan user add admin ...    # Run CKAN CLI commands
bin/shell                      # Bash shell in ckan-dev container
bin/generate_extension         # Create new extension in src/
bin/install_src                # Install all src/ extensions (run after adding deps)
bin/reload                     # Reload CKAN without container restart (kills Python processes)
bin/restart                    # Full ckan-dev container restart
```

**Important**: `bin/reload` is fast (kills Python processes) but doesn't reload `.env` changes. Use `bin/compose up -d` for that.

### Extension Development Pattern
1. Extensions go in `src/ckanext-<name>/` (mounted volume)
2. Add extension to `CKAN__PLUGINS` in `.env` file
3. Run `bin/install_src` to install dependencies and run `setup.py develop`
4. Use `bin/reload` to reload CKAN without full restart
5. For new dependencies: update `requirements.txt` → `bin/install_src` → `bin/reload`

### Configuration System (ckanext-envvars)
Environment variables in `.env` override `ckan.ini` settings:
- Format: `CKAN__SECTION__KEY` (uppercase, double underscores)
- Example: `CKAN__PLUGINS="envvars datastore datapusher"`
- Example: `CKAN__DATAPUSHER__CALLBACK_URL_BASE=http://ckan-dev:5000`
- Use `CKAN___` prefix for keys not starting with "CKAN" (e.g., `CKAN___BEAKER__SESSION__SECRET`)

### Image Customization

#### Dockerfiles
- `ckan/Dockerfile`: Production (based on `ckan/ckan-base:2.11`)
- `ckan/Dockerfile.dev`: Development (based on `ckan/ckan-dev:2.11`)

#### Extension Installation in Dockerfile.dev
```dockerfile
RUN pip3 install -e 'git+https://github.com/org/ckanext-name.git@master#egg=ckanext-name' && \
    pip3 install -r ${APP_DIR}/src/ckanext-name/requirements.txt
```

#### Initialization Scripts
Place scripts in `ckan/docker-entrypoint.d/` - they execute after `prerun.py` but before server starts:
- Example: `ckan/docker-entrypoint.d/01_setup_datapusher.sh` (sets up datapusher API token)
- Scripts run in alphanumeric order
- Use for: DB initialization, token generation, config setup

#### Applying Patches
Place patches in `ckan/patches/<package-name>/`:
```
ckan/patches/
  ckan/
    01_datasets_per_page.patch
  ckanext-harvest/
    01_resubmit_objects.patch
```

### Debugging

#### Remote Debugging (VS Code)
1. Set `USE_DEBUGPY_FOR_DEV=true` in `.env`
2. Run `bin/install_src`
3. VS Code: Attach to Running Container → select `ckan-dev`
4. Create launch.json: Remote Attach, localhost:5678

#### pdb Debugging
Add to `docker-compose.dev.yml` ckan-dev service:
```yaml
stdin_open: true
tty: true
```
Then: `docker attach $(docker container ls -qf name=ckan-dev)`

## Project-Specific Conventions

### Database Setup
Three databases created automatically in `postgresql/docker-entrypoint-initdb.d/`:
- `ckan_db`: Main CKAN data
- `datastore_db`: Read-only data access (has readonly user)
- Test databases: For running CKAN tests

### File Override Pattern
Custom scripts in `ckan/setup/*.override` replace base image scripts:
- `start_ckan.sh.override`: Custom startup logic (production)
- `start_ckan_development.sh.override`: Custom dev startup
- `prerun.py.override`: Database initialization and plugin setup
- Copy in Dockerfile: `COPY setup/start_ckan.sh.override ${APP_DIR}/start_ckan.sh`

### Network Segmentation
Services communicate via named networks (see docker-compose.yml):
- `ckannet`: CKAN ↔ DataPusher, NGINX
- `dbnet`: CKAN/DataPusher ↔ PostgreSQL
- `solrnet`: CKAN ↔ Solr
- `redisnet`: CKAN ↔ Redis
- `webnet`: NGINX ↔ external

### Dev vs Prod Differences
| Aspect | Production | Development |
|--------|-----------|-------------|
| Container | `ckan` | `ckan-dev` |
| Server | uWSGI | Flask dev server |
| Port | 8443 (NGINX) | 5000 (direct) |
| CKAN_SITE_URL | https://localhost:8443 | http://localhost:5000 |
| Extensions | Baked in Dockerfile | Mounted from `src/` |
| Callback URL | http://ckan:5000 | http://ckan-dev:5000 |

## Common Pitfalls

1. **Don't run CKAN CLI directly**: Use `bin/ckan` instead of `docker compose exec ckan-dev ckan`
2. **After .env changes**: Use `bin/compose up -d`, not `bin/reload`
3. **CKAN_SITE_URL mismatch**: Update both `.env` and `CKAN__DATAPUSHER__CALLBACK_URL_BASE` together
4. **Extensions not loading**: Check `CKAN__PLUGINS` in `.env` includes the extension name
5. **Permission issues with src/**: The `bin/generate_extension` script handles user/group correctly
