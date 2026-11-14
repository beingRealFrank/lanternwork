# Tiltfile built-ins: a quick reference

Source of truth: the [Tilt Tiltfile API reference](https://docs.tilt.dev/api.html). This page is a short, practical summary of the functions you reach for most often. The official reference has every argument and the full list.

A Tiltfile is written in Starlark, a small Python-like language. Tilt runs it top to bottom to work out what to build, deploy and watch.

## Build images

| Function | What it does | Example |
|---|---|---|
| `docker_build(ref, context, …)` | Builds an image from a Dockerfile when its files change. Useful keyword arguments: `dockerfile`, `build_args`, `target`, `only`, `ignore`, `live_update`, `platform`. | `docker_build('myorg/api', '.', dockerfile='api/Dockerfile')` |
| `custom_build(ref, command, deps, …)` | Builds an image with your own command. `deps` lists the files that trigger a rebuild; `tag` and `disable_push` control the result. | `custom_build('myorg/api', 'make image EXPECTED_REF=$EXPECTED_REF', ['./src'])` |
| `docker_compose(configPaths, …)` | Brings in services from a Docker Compose file. | `docker_compose('docker-compose.yml')` |
| `dc_resource(name, …)` | Adjusts how one Compose service is shown and built. | `dc_resource('db', labels=['data'])` |

### Live update

Pass `live_update=[…]` to `docker_build` or `custom_build` to copy changed files into a running container instead of rebuilding the image. The steps are:

- `sync(local_path, remote_path)` copies files into the container.
- `run(cmd, trigger=[…])` runs a command in the container, optionally only when certain files change.
- `fall_back_on(files)` forces a full rebuild when those files change.
- `restart_container()` restarts the container's process after the sync.
- `initial_sync()` copies all files when the container first starts.

```python
docker_build(
    'myorg/web', '.',
    live_update=[
        fall_back_on('package.json'),
        sync('./src', '/app/src'),
        run('npm run build', trigger=['./src']),
    ],
)
```

## Deploy to Kubernetes

| Function | What it does | Example |
|---|---|---|
| `k8s_yaml(yaml, allow_duplicates=False)` | Deploys Kubernetes objects from a YAML file path or from a blob of YAML. Tilt matches the images in the YAML to your `docker_build` calls. | `k8s_yaml('k8s/deploy.yaml')` |
| `k8s_resource(workload, new_name, …)` | Configures one deployed workload: `port_forwards`, `resource_deps`, `labels`, `trigger_mode`, `auto_init`, `links`, `objects`. | `k8s_resource('api', port_forwards=8080, resource_deps=['db'])` |
| `k8s_custom_deploy(name, apply_cmd, delete_cmd, deps, …)` | Deploys with your own commands when the built-in apply does not fit. | `k8s_custom_deploy('api', apply_cmd='./apply.sh', delete_cmd='./delete.sh', deps=['k8s'])` |
| `helm(pathToChartDir, name, namespace, values, set, …)` | Renders a Helm chart to YAML (pass the result to `k8s_yaml`). | `k8s_yaml(helm('charts/api', values=['values-dev.yaml']))` |
| `kustomize(pathToDir, …)` | Renders a Kustomize directory to YAML. | `k8s_yaml(kustomize('overlays/dev'))` |
| `k8s_kind(kind, …)` | Teaches Tilt how to find images and pods for a custom resource kind. | `k8s_kind('MyCRD', image_json_path='{.spec.image}')` |
| `k8s_namespace(…)` / `k8s_context(…)` | Read the namespace or context Tilt is using. | `k8s_context()` |
| `allow_k8s_contexts(contexts)` | Lets Tilt run against the named contexts. Tilt refuses unknown contexts by default, to protect production clusters. | `allow_k8s_contexts('my-dev-cluster')` |
| `default_registry(host, …)` | Renames built images so they point at a registry. | `default_registry('registry.example.com/team')` |

## Run local commands

| Function | What it does | Example |
|---|---|---|
| `local_resource(name, cmd, deps, serve_cmd, …)` | A resource Tilt runs on your machine: `cmd` runs when `deps` change, and `serve_cmd` keeps a long-running process alive. Also takes `resource_deps`, `labels`, `trigger_mode`, `allow_parallel`, `auto_init`, `ignore`, `readiness_probe`. | `local_resource('lint', cmd='make lint', deps=['src'])` |
| `local(command, quiet=False, env={}, dir='', stdin=None)` | Runs a command while the Tiltfile is evaluated and returns its output as a blob. | `branch = str(local('git rev-parse --abbrev-ref HEAD')).strip()` |

## Control updates

- `trigger_mode=TRIGGER_MODE_MANUAL` (on a resource) makes Tilt wait for you to click update instead of rebuilding automatically. `TRIGGER_MODE_AUTO` is the default.
- `resource_deps=['other']` makes a resource wait for another to be ready.
- `labels=['group']` groups resources in the Tilt UI.
- `auto_init=False` skips starting a resource until you ask for it.
- `watch_file(path)` rebuilds the Tiltfile when a file the Tiltfile itself reads changes.
- `update_settings(…)` sets global limits, for example `max_parallel_updates`.

## Split a Tiltfile into pieces

| Function | What it does | Example |
|---|---|---|
| `load(path, symbol, …)` | Imports functions from another Tiltfile. A path starting with `ext://` loads a Tilt extension. | `load('ext://namespace', 'namespace_create')` |
| `load_dynamic(path)` | Like `load`, but loads at run time and returns a dictionary of symbols. | `syms = load_dynamic('./other/Tiltfile')` |
| `include(path)` | Runs another Tiltfile as if it were part of this one. | `include('./backend/Tiltfile')` |

## Files, data and environment

| Function | What it does |
|---|---|
| `read_file(path, default=None)` | Reads a file as a blob and watches it. |
| `read_json(path)`, `read_yaml(path)` | Parse a JSON or YAML file. |
| `decode_json(s)`, `decode_yaml(s)`, `encode_yaml(obj)`, `encode_json(obj)` | Convert between strings and data. |
| `listdir(directory, recursive=False)` | Lists files in a directory (and watches it). |
| `os.getenv(key, default)`, `os.putenv(key, value)`, `os.getcwd()` | Environment and working directory. |
| `os.path.join(…)`, `os.path.exists(path)`, `os.path.basename(path)`, `os.path.abspath(path)` | Path helpers. |
| `blob(contents)` | Wraps a string as a blob. |
| `fail(msg)`, `warn(msg)`, `print(…)` | Stop with an error, show a warning, or print. |

## Command-line options

```python
config.define_string('env')
cfg = config.parse()
if cfg.get('env', 'dev') == 'prod':
    fail('not against prod')
```

`config.define_string`, `config.define_bool`, `config.define_string_list` and `config.parse` let you pass `tilt up -- --env dev` style arguments into the Tiltfile.

---

Editing Tiltfiles in a JetBrains IDE? [Loopfile](https://plugins.jetbrains.com/plugin/36122-loopfile) adds highlighting for these built-ins, completion with short documentation, and Cmd+click on `load()` and `include()` paths.
