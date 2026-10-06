# Practice 06 — Container Security

**Objective:** take a Containerfile that *works* and make it *safe to
ship*: non-root, pinned, minimal, no secrets inside, and still working
when the runtime takes almost every privilege away — the skills from
today's lecture.

**Timebox:** ~30 min of actual work within today's session.

> **How to submit:** same as Practice 05 — on **Maru**. Sign in, click
> **Accept** on "Practice 06", accept the GitHub invite, clone your
> private repo under `weeebdev-edu`, and work there. No PR: commit and
> push to `main` — GitHub Actions grades every push automatically, in
> about a minute.

## Task

Your repo has:

- `main.go` — a tiny Go HTTP server. **Don't touch it.** `GET /` answers
  `INF345 Practice 06 OK uid=<uid> token=<set|missing>` — it reports
  which user it runs as, and whether `API_TOKEN` was given to it (never
  the value). On startup it writes a small file to `/tmp`.
- `.env` — a local-development secret (`API_TOKEN=...`). It's a fake
  token, but treat it exactly like a real one.
- `Containerfile` — builds and runs fine. It is also a security review's
  worst nightmare. Every line marked `PROBLEM` is something to fix.

Harden the `Containerfile` (and add a `.dockerignore` if you need one) so
that:

1. **Pinned base images.** Every `FROM` names a real version — a tag
   such as `golang:1.23-alpine`, or a `@sha256:` digest. No `latest`, no
   missing tag.
2. **Multi-stage, small final image.** Build with the Go toolchain, ship
   only the binary. The final image must be **under 50 MB** (the starter
   is ~1 GB).
3. **Non-root.** The final image has a `USER` with a **non-zero uid**,
   and the process really runs as that uid.
4. **No secret in the image.** The token must not be anywhere in the
   final image — not in `ENV`, not in the image history, not in a copied
   `.env` file. It arrives at **runtime**: `docker run -e API_TOKEN=...`.
5. **Works hardened.** The image still serves requests when run with
   `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges`.

Test it locally before you push:

```bash
docker build -t p06 -f Containerfile .
docker run --rm -p 8080:8080 -e API_TOKEN=dev-token p06
curl localhost:8080          # -> INF345 Practice 06 OK uid=65532 token=set

docker image ls p06                                # size
docker image inspect -f '{{.Config.User}}' p06     # who it runs as
docker image inspect -f '{{.Config.Env}}' p06      # no API_TOKEN here!
docker history --no-trunc p06 | grep -i token      # ...or here

# exactly how the autograder runs it:
docker run --rm -p 8080:8080 --read-only --tmpfs /tmp \
  --cap-drop ALL --security-opt no-new-privileges -e API_TOKEN=x p06
```

(Podman works too: `podman build`, `podman run`, same flags.)

## Definition of done

- [ ] Every `FROM` is pinned to a tag (not `latest`) or a digest
- [ ] Multi-stage build; final image under 50 MB
- [ ] The final image has a non-root `USER`; `GET /` shows `uid=` not 0
- [ ] `API_TOKEN` is nowhere in the final image — `inspect`, `history`
      and the layers are all clean
- [ ] Runs with `--read-only --tmpfs /tmp --cap-drop ALL
      --security-opt no-new-privileges` and answers `token=set`
- [ ] Pushed to `main` before the session ends (this is also your
      attendance signal)

## Common mistakes

- **`FROM scratch` and the app dies on start.** `scratch` is *empty* — no
  `/tmp`, no users, nothing. The app writes to `/tmp` on startup.
  `gcr.io/distroless/static-debian12:nonroot` or `alpine:3.20` give you a
  real `/tmp` and still tiny images.
- **Deleting the secret in a later layer.** `COPY . .` then
  `RUN rm .env` does **not** remove it — the earlier layer still contains
  the file, and anyone can extract it. Never put it in a layer at all:
  copy only what you need (`COPY go.mod main.go ./`), or exclude `.env`
  in a `.dockerignore`.
- **`ARG API_TOKEN=...` in the final stage.** Build args show up in
  `docker history`. Secrets are a runtime concern (`-e`, or your
  orchestrator's secrets), not a build-time one.
- **`USER app` without creating `app`.** A user *name* must exist in the
  image's `/etc/passwd`, or the container fails to start ("unable to
  find user app"). Create it (`RUN adduser -D -u 10001 app` on alpine),
  or use a numeric uid (`USER 65532`) — distroless's `:nonroot` tag
  already sets one for you.
- **Pinning only the final stage.** The builder stage's `FROM` counts too.

## Grading

Real autograding — GitHub Actions builds your image, inspects it, and
runs it the way a security reviewer would, out of 100 points:

| Check | Points |
|---|---|
| Image builds | 10 |
| Serves on `:8080` (`GET /` → `token=set`) | 10 |
| Runs as non-root (`USER` set, process uid ≠ 0) | 20 |
| Every base image pinned (tag ≠ `latest`, or digest) | 10 |
| Token nowhere in the image (env, history, layers) | 20 |
| Works read-only, non-root, with all capabilities dropped | 15 |
| Final image under 50 MB | 15 |

Check the **Actions** tab in your repo for the run, or the workflow
run's **Summary** for the exact score. Workflow:
`.github/workflows/classroom.yml` (in your repo, not this one).

This score is this session's grade within the **Weekly practice
sessions** category — 1 point per practice, your best 10 count (see
`SYLLABUS.md`).

## Stretch (not graded)

Scan your image for known vulnerabilities and compare it with the
starter:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.58.1 image --severity HIGH,CRITICAL p06
```
