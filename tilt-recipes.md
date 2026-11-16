# Tiltfile recipes: common patterns

Source of truth: the [Tilt Tiltfile API reference](https://docs.tilt.dev/api.html) and the [Tilt documentation](https://docs.tilt.dev/). This page collects patterns that come up often; names such as `myorg/api` are examples. For the function list see [Tiltfile built-ins](tiltfile-api).

## Build an image and deploy it

```python
docker_build('myorg/api', '.', dockerfile='api/Dockerfile')
k8s_yaml('k8s/api.yaml')
k8s_resource('api', port_forwards=8080)
```

Tilt matches the image name in the YAML to the `docker_build` call, rebuilds when files in the build context change, and forwards port 8080 to the workload.

## Update a running container without rebuilding (live update)

Copy changed files straight into the container and run a command only when its inputs change. `fall_back_on` forces a full rebuild for files such as a lock file:

```python
docker_build(
    'myorg/web', '.',
    live_update=[
        fall_back_on(['package.json', 'package-lock.json']),
        sync('./src', '/app/src'),
        run('npm run build', trigger=['./src']),
    ],
)
```

## Run local commands as resources

A lint step that runs when sources change, and a long-running local server that Tilt keeps alive:

```python
local_resource('lint', cmd='make lint', deps=['src'])
local_resource('mock-api', serve_cmd='python3 -m http.server 9000', labels=['dev'])
```

## Order and group resources

```python
k8s_resource('api', resource_deps=['db'], labels=['backend'])
k8s_resource('db', labels=['data'])
```

`resource_deps` makes `api` wait until `db` is ready. `labels` group resources in the Tilt UI.

## Charts and overlays

```python
k8s_yaml(helm('charts/api', values=['values-dev.yaml']))
k8s_yaml(kustomize('overlays/dev'))
```

## Manual updates for expensive resources

```python
k8s_resource('migrations', trigger_mode=TRIGGER_MODE_MANUAL, auto_init=False)
```

The resource does not start or update until you trigger it in the UI.

## Settings and safety

Tilt refuses to run against Kubernetes contexts it does not know, to protect production clusters. Allow your own dev context by name:

```python
allow_k8s_contexts('my-dev-cluster')
```

Pass options on the command line (`tilt up -- --env dev`) by defining them in the Tiltfile:

```python
config.define_string('env')
cfg = config.parse()
if cfg.get('env', 'dev') == 'prod':
    fail('not against prod')
```

## Split a big Tiltfile

```python
load('ext://namespace', 'namespace_create')
include('./backend/Tiltfile')
```

`load` imports functions (here from a Tilt extension), and `include` runs another Tiltfile as part of this one.

## Habits that save time

- Keep the build context small with `only=` or `ignore=` on `docker_build`, so unrelated edits do not trigger rebuilds.
- Prefer `live_update` for interpreted languages; keep a full rebuild as the fallback.
- Put slow or rarely needed resources behind `auto_init=False` or manual trigger mode.

---

Editing Tiltfiles in a JetBrains IDE? [Loopfile](https://plugins.jetbrains.com/plugin/36122-loopfile) adds highlighting for these built-ins, completion with short documentation, and Cmd+click on `load()` and `include()` paths.
