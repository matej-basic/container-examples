# container-examples

Four hello-world web apps, one per stack: Go, Flask, Django and React. Each directory holds a small app and the `Containerfile` that packages it into an image. The apps do almost nothing on purpose, so the `Containerfile` is the part worth reading.

| Directory | App | Image | Port in the container |
|---|---|---|---|
| `golang/` | Fiber server, `GET /` returns `Hello, World!` | two stages, both on `golang:1.20-alpine` | 3000 |
| `python-flask/` | Flask, `GET /hello` returns `{"message": "Hello, World!"}` | `python:3.9` with nginx and uWSGI | 8080 |
| `python-django/` | Django project `helloworldapi`, `GET /hello` returns the same JSON | same as Flask | 8080 |
| `javascript-react/` | default Create React App page | built on `node:14`, served by `nginx:alpine` | 80 |

## What each directory shows

`golang/` is a multi-stage build. The first stage copies the source and runs `go mod tidy` and `go build`. The second stage starts again from `golang:1.20-alpine` and copies in only the compiled `main` binary. The server listens on port 3000, which is set in `main.go`; the `Containerfile` has no `EXPOSE` line.

`python-flask/` and `python-django/` use the same `Containerfile` and the same `nginx.conf`. On top of `python:3.9` the image installs nginx, adds an unprivileged user called `customuser`, gives it `/app` and nginx's log and state directories, and switches to it. uWSGI runs up to six worker processes behind the Unix socket `/app/app.sock`. nginx listens on 8080 and hands requests to that socket with `uwsgi_pass`. The start command puts uWSGI in the background and keeps nginx in the foreground, so one container runs both. The Django project also routes `/admin/`, but the image never runs migrations, so `/hello` is the only page meant to work there.

`javascript-react/` is the unchanged Create React App starter. A `node:14` stage runs `npm install` and `npm run build`, and the `build/` output is copied into `nginx:alpine`, which serves it on port 80. Any path that is not a file gets `index.html`, so client-side routes still load the app.

## Build and run

Run the commands from the repository root. They use Podman; with Docker, replace `podman` with `docker` and keep the rest. The `-f` flag names the `Containerfile` explicitly so the same line works with both.

Go:

```sh
podman build -t container-examples-go -f golang/Containerfile golang
podman run --rm -p 3000:3000 container-examples-go
curl http://localhost:3000/
```

Flask:

```sh
podman build -t container-examples-flask -f python-flask/Containerfile python-flask
podman run --rm -p 8080:8080 container-examples-flask
curl http://localhost:8080/hello
```

Django needs a secret key from the environment:

```sh
podman build -t container-examples-django -f python-django/Containerfile python-django
podman run --rm -p 8080:8080 -e DJANGO_SECRET_KEY="$(openssl rand -hex 32)" container-examples-django
curl http://localhost:8080/hello
```

`settings.py` reads three variables:

| Variable | Default | Effect |
|---|---|---|
| `DJANGO_SECRET_KEY` | none | Django's `SECRET_KEY`. Required unless `DJANGO_DEBUG` is on. |
| `DJANGO_DEBUG` | off | `1`, `true`, `yes` or `on` turns on `DEBUG`. With debug on and no key set, the app falls back to a public development-only key written in `settings.py`. |
| `DJANGO_ALLOWED_HOSTS` | `localhost,127.0.0.1` | Comma-separated host names Django answers for. A request for any other host gets 400. |

Without `DJANGO_SECRET_KEY` and with debug off, the container still starts, but uWSGI cannot load the app: every request returns 500 and the log shows `ImproperlyConfigured`. To try it quickly on your own machine, `podman run --rm -p 8080:8080 -e DJANGO_DEBUG=1 container-examples-django` works without a key. To reach the app under another name, for example the machine's IP address, add that name to `DJANGO_ALLOWED_HOSTS`.

Flask and Django both listen on 8080. To run them at the same time, map one of them to another host port, for example `-p 8081:8080`.

React:

```sh
podman build -t container-examples-react -f javascript-react/Containerfile javascript-react
podman run --rm -p 8080:80 container-examples-react
```

Then open http://localhost:8080/. nginx listens on 80 inside the container; the example maps it to 8080 because rootless Podman cannot bind host ports below 1024 by default.

## Before you reuse this

These files are kept simple for teaching, and four things in them should change before you build on them:

- The Go runtime stage starts from the full `golang:1.20-alpine` image, so the final image carries the Go compiler and toolchain next to one binary. Build a static binary (`CGO_ENABLED=0`) and copy it into a minimal base such as `alpine` or `scratch`.
- `node:14` is past end of life. Node.js 14 stopped getting updates, security fixes included, in April 2023.
- The Python requirements are not pinned. Both `requirements.txt` files list package names with no versions, so every build installs whatever pip picks that day.
- The React build copies `package.json` without `package-lock.json` before `npm install`, so the committed lockfile is not used and dependency versions can drift from one build to the next. Copy both files and run `npm ci` instead.
