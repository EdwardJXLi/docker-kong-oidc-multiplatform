# docker-kong-oidc

> Builds an AMD64 and ARM64 Docker image (<ghcr.io/edwardjxli/docker-kong-oidc-multiplatform>) from base Kong + [revomatico/kong-oidc](https://github.com/revomatico/kong-oidc) plugin (based on zmartzone/lua-resty-openidc, based on nokia/kong-oidc)
> 
## Notes

- Overriding numeric values like ports via env vars: due to a limitation in the lua templating engine in openresty, they must be quoted twice: `KONG_X_VAR="'1234'"`.
- Dockerfile will patch `nginx_kong.lua` template at build time, to include `set $session_secret "$KONG_X_SESSION_SECRET";`
  - This is needed for the kong-oidc plugin to set a session secret that will later override the template string
  - See: <https://github.com/nokia/kong-oidc/issues/1>
- A common default session_secret must be defined by setting env `KONG_X_SESSION_SECRET` to a string
- To enable the plugins, set the env variable for the container with comma separated plugin values:
  - `KONG_PLUGINS=bundled,oidc`
- Default: `KONG_X_SESSION_NAME=oidc_session`

## Session: Cookie

- This is the default, but not recommended. I would recommend **shm** for a single instance, lightweight deployment.
- If you have too much information in the session (claims, etc), you may need to [increase the nginx header size](https://github.com/bungle/lua-resty-session#cookie-storage-adapter):
  - `KONG_NGINX_LARGE_CLIENT_HEADER_BUFFERS='4 16k'`
- You can also enable [session compression](https://github.com/bungle/lua-resty-session#pluggable-compressors) to reduce cookie size:
  - `KONG_X_SESSION_COMPRESSOR=zlib`

## Session: Memcached

> Instead of actual memcached, Hazelcast (that is Kubernetes aware), with memcache protocol enabled should be used.
> See <https://docs.hazelcast.org/docs/latest-dev/manual/html-single/#memcache-client>.

- Reference: <https://github.com/bungle/lua-resty-session#memcache-storage-adapter>
- To replace the default sesion storage: **cookie**, set
  - `KONG_X_SESSION_STORAGE=memcache`
- Memcached hostname is by default **memcached** (in my case installed via helm in a Kubernetes cluster)
  - Set `KONG_X_SESSION_MEMCACHE_HOST=mynewhost`
  - Alternatively, set up DNS entry for **memcached** to be resolved from within the container
- Memcached port is by default **11211**, override by setting:
  - `KONG_X_SESSION_MEMCACHE_PORT="'12345'"`
- KONG_X_SESSION_MEMCACHE_USELOCKING, default: off
- KONG_X_SESSION_MEMCACHE_SPINLOCKWAIT, default: 150
- KONG_X_SESSION_MEMCACHE_MAXLOCKWAIT, default: 30
- KONG_X_SESSION_MEMCACHE_POOL_TIMEOUT, default: 10
- KONG_X_SESSION_MEMCACHE_POOL_SIZE, default: 10
- KONG_X_SESSION_MEMCACHE_CONNECT_TIMEOUT, default 1000 (milliseconds)
- KONG_X_SESSION_MEMCACHE_SEND_TIMEOUT, default 1000 (milliseconds)
- KONG_X_SESSION_MEMCACHE_READ_TIMEOUT, default 1000 (milliseconds)

## Session: DSHM (Hazelcast + Vertex)

> This lua-resty-session implementation depends on [grrolland/ngx-distributed-shm](https://github.com/grrolland/ngx-distributed-shm) dshm.lua library.
> Recommended: Hazelcast with memcache protocol enabled (see above).

- Reference: <https://github.com/bungle/lua-resty-session#dshm-storage-adapter>
- To replace the default sesion storage: **cookie**, set
  - `KONG_X_SESSION_STORAGE=dshm`
- X_SESSION_DSHM_REGION, default: oidc_sessions
- X_SESSION_DSHM_CONNECT_TIMEOUT, default: 1000
- X_SESSION_DSHM_SEND_TIMEOUT, default: 1000
- X_SESSION_DSHM_READ_TIMEOUT, default: 1000
- X_SESSION_DSHM_HOST, default: hazelcast
- X_SESSION_DSHM_PORT, default: 4321
- X_SESSION_DSHM_POOL_NAME, default: oidc_sessions
- X_SESSION_DSHM_POOL_TIMEOUT, default: 1000
- X_SESSION_DSHM_POOL_SIZE, default: 10
- X_SESSION_DSMM_POOL_BACKLOG, default: 10

## Session: SHM

> Good for single instance. No additional software is required.

- Reference: <https://github.com/bungle/lua-resty-session#shared-dictionary-storage-adapter>
- To replace the default sesion storage: **cookie** with **shm**, set
  - `KONG_X_SESSION_STORAGE=shm`
- KONG_X_SESSION_SHM_STORE, default: oidc_sessions
- KONG_X_SESSION_SHM_STORE_SIZE, default: 5m
- KONG_X_SESSION_SHM_USELOCKING, default: no
- KONG_X_SESSION_SHM_LOCK_EXPTIME, default: 30
- KONG_X_SESSION_SHM_LOCK_TIMEOUT, default: 5
- KONG_X_SESSION_SHM_LOCK_STEP, default: 0.001
- KONG_X_SESSION_SHM_LOCK_RATIO, default: 2
- KONG_X_SESSION_SHM_LOCK_MAX_STEP, default: 0.5

## Exclude IPs from access_log

- `KONG_X_NOLOG_LIST_FILE` could be set to a file path, e.g. `/tmp/nolog.txt`
- File format is `ip 0;`. To exclude for example requests from the kubernetes probes:

    ```
    127.0.0.1 0;
    ```
