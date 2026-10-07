---
theme: seriph
title: "INF345 — Lecture 6: Container Security & Module Wrap-up"
info: |
  INF345 — Fundamentals of DevOps
  Lecture 6 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# INF 345 — Fundamentals of DevOps

## Lecture 6: Container Security & Module Wrap-up

<div class="pt-8 opacity-70">
Adil Akhmetov · Lesson 6
</div>

---
layout: default
---

# Recap — Lesson 5 (Networking & Volumes)

<v-clicks>

- Why can two containers on a user-defined network reach each other by name? <span v-click class="opacity-60">(the network has built-in DNS that resolves service/container names)</span>
- What's the difference between `docker compose down` and `down -v`? <span v-click class="opacity-60">(`-v` also deletes named volumes — the data is gone)</span>
- Why should the database publish no host port? <span v-click class="opacity-60">(only the app needs to reach it, over the internal network — publishing it widens the attack surface for nothing)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Threat model: what an attacker gets when a container is compromised
- [ ] Root in a container — and how to stop being root
- [ ] Capabilities, read-only filesystems, `no-new-privileges`
- [ ] Secrets: why they never belong in an image
- [ ] Supply chain: pinning, minimal bases, scanning
- [ ] Module wrap-up: containers, weeks 3–6 — and Lab 01
- [ ] Straight into today's practice: harden a Containerfile

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Your app has a bug — say, a file-upload endpoint that lets an attacker
run a command inside the container. It happens to every team eventually.
The question isn't <b>whether</b> a container gets compromised.

</div>

<div v-click class="mt-8 text-xl font-bold">
The question is: once they're in, what can they reach? Today is about
making the answer "almost nothing".
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Containers are not VMs

---
---

# What actually isolates a container

<v-clicks>

- A container is **a normal Linux process** with a restricted view:
  namespaces (what it can *see*) and cgroups (how much it can *use*).
- There is **one kernel**, shared with the host and every other
  container. A VM has its own kernel; a container doesn't.
- So security is layered on top: users, capabilities, seccomp,
  SELinux/AppArmor, a read-only filesystem.
- Every one of those layers is something **you** can switch on or off in
  a Containerfile or a `run` flag.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Defense in depth: assume one layer fails. A bug in your app should not
be a bug in the host.
</div>

---
---

# Root in the container = root on the host?

```bash
docker run --rm alpine id
# uid=0(root) gid=0(root) — the default
```

<v-clicks>

- With rootful Docker and no user namespaces, uid 0 inside **is** uid 0
  on the host kernel — just fenced in by namespaces.
- One kernel bug, one mis-mounted path (`-v /:/host`), or one
  `--privileged`, and the fence is gone.
- **Rootless Podman** maps container root to *your* unprivileged user on
  the host — a big reason this course uses Podman.
- Even then: don't run as root inside. Defense in depth.

</v-clicks>

---
---

# Stop being root: `USER`

```dockerfile
FROM alpine:3.20
RUN adduser -D -u 10001 app      # create the user (alpine syntax)
COPY --from=build /out/server /server
USER 10001                       # every later RUN, and the container, run as 10001
CMD ["/server"]
```

<v-clicks>

- Prefer a **numeric uid** — Kubernetes' `runAsNonRoot` can only verify
  numbers, and a name that doesn't exist in `/etc/passwd` fails at start.
- Distroless images ship a ready-made one: `gcr.io/distroless/static-debian12:nonroot` (uid 65532).
- Files the app must write need the right owner: `COPY --chown=10001:10001 ...`
- Ports below 1024 traditionally need root → listen on 8080, publish `-p 80:8080`.

</v-clicks>

---
---

# Check who you really are

```bash
docker image inspect -f '{{.Config.User}}' myimg     # what the image says
docker run --rm myimg id                             # what actually runs
docker run --rm --user 0 myimg id                    # anyone can override — at runtime
```

<div v-click class="mt-6 text-sm">

`USER` sets the **default**. Orchestrators can enforce it: Kubernetes
`securityContext.runAsNonRoot: true` refuses to start a pod whose image
would run as uid 0.

</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your Containerfile ends with <code>USER app</code>. The build succeeds,
but the container exits immediately with
<i>"unable to find user app"</i>. What happened?
</div>

<div v-click class="mt-8 text-lg opacity-70">
The user was never created in the final image — a user <i>name</i> must
exist in that image's <code>/etc/passwd</code>. Create it in the final
stage, or use a numeric uid.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## Least privilege at runtime

---
---

# Capabilities: root, split into pieces

<v-clicks>

- Linux splits root's power into ~40 **capabilities**: `NET_BIND_SERVICE`
  (low ports), `CHOWN`, `SYS_ADMIN` (almost everything), `NET_RAW`
  (raw packets, e.g. ping spoofing)…
- Docker gives containers a default set of ~14. Your web app needs
  **zero** of them.

</v-clicks>

```bash {none|1|2-3|all}
docker run --cap-drop ALL myimg                              # drop everything
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimg   # add back only
                                                             # what you need
```

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
<code>--privileged</code> is the opposite: <b>all</b> capabilities, all
devices, no seccomp. Treat it as "this container is the host".
</div>

---
---

# Read-only root filesystem

```bash
docker run --read-only --tmpfs /tmp myimg
```

<v-clicks>

- The image's filesystem becomes immutable at runtime — an attacker
  can't drop a binary, edit a config, or plant a cron job.
- Apps still need *somewhere* to write → give them exactly that:
  `--tmpfs /tmp` (in memory), or a volume for real data.
- If your app crashes under `--read-only`, it's telling you where it
  writes. That's useful information.

</v-clicks>

---
---

# `no-new-privileges` and the hardened run

```bash
docker run -d -p 8080:8080 \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  myimg
```

<v-clicks>

- `no-new-privileges`: no process in the container can gain privileges
  later — `setuid` binaries like `su` or `sudo` stop working.
- This one command is today's practice's grading run — note there's no
  `--user`: the image's own `USER` has to make it non-root. If your image
  survives it, it is in better shape than most images on Docker Hub.
- In Compose: `read_only: true`, `cap_drop: [ALL]`,
  `security_opt: [no-new-privileges:true]`, `tmpfs: [/tmp]`.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 3
## Secrets

---
---

# Images are not secret

<v-clicks>

- Anyone who can **pull** an image can read **everything** in it: every
  layer, every `ENV`, every build step.
- Images get pushed to registries, cached on CI runners, copied to
  laptops. Assume an image is public.
- So a secret in an image is a leaked secret — the only question is when.

</v-clicks>

```bash {none|1|2|3}
docker image inspect -f '{{.Config.Env}}' myimg     # ENV API_TOKEN=... in plain sight
docker history --no-trunc myimg                     # every ENV/ARG/RUN line, values included
docker save myimg -o img.tar && tar -xf img.tar     # every file, in every layer
```

---
---

# Three ways secrets leak into images

| Leak | Why it's in there | Fix |
|---|---|---|
| `ENV API_TOKEN=...` | ENV is image metadata | Pass at runtime: `-e`, `--env-file` |
| `ARG TOKEN=...` used in `RUN` | Build args are recorded in history | BuildKit secret mounts |
| `COPY . .` copies `.env` | It's just a file in a layer | `.dockerignore`, copy only what you need |

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
<code>RUN rm .env</code> in a later step does <b>not</b> help: the file is
still in the earlier layer. Layers only ever add — deleting just hides it
from the final view.
</div>

---
---

# Doing it right

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**At runtime** — the secret never touches the image:

```bash
docker run -e API_TOKEN="$API_TOKEN" myimg
docker run --env-file .env myimg
```

Kubernetes: a `Secret`, mounted as env or file.

</div>
<div>

**At build time** — when a build *needs* one (private registry, private
git):

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm \
    NPM_TOKEN=$(cat /run/secrets/npm) npm ci
```

```bash
docker build --secret id=npm,src=.npmrc-token .
```

Mounted only for that one `RUN`; never written to a layer.

</div>
</div>

<div v-click class="mt-4 text-sm opacity-70">
And <code>.dockerignore</code> with <code>.env</code>, <code>.git</code>, keys —
it also makes builds faster.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
A multi-stage build does <code>COPY . .</code> — <code>.env</code> included —
in the <b>builder</b> stage only, and the final stage copies just the
binary. Is the secret in the shipped image?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — only the final stage ships. But the builder's layers stay in your
build cache and CI runner, so add <code>.dockerignore</code> anyway.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Supply chain: what you build on

---
---

# Pin your base images

```dockerfile
FROM golang:latest                      # whatever "latest" means today
FROM golang:1.23-alpine                 # a version — readable, still movable
FROM golang:1.23-alpine@sha256:4a1c…    # exactly these bytes, forever
```

<v-clicks>

- `latest` is just a tag someone moves. Same Containerfile, different
  image next week — non-reproducible builds, surprise breakage.
- A version tag is the usual balance; a **digest** is fully reproducible
  (bots like Renovate/Dependabot bump digests for you).
- Pin **every** stage — the builder's toolchain is part of your supply
  chain too.

</v-clicks>

---
---

# Smaller image, smaller attack surface

| Base | Size | Shell & package manager |
|---|---|---|
| `golang:1.23` | ~800 MB | yes — and a compiler |
| `alpine:3.20` | ~8 MB | `sh`, `apk` |
| `gcr.io/distroless/static-debian12` | ~2 MB | no shell at all |
| `scratch` | 0 MB | nothing — not even `/tmp` |

<v-clicks>

- Every package you ship is a package that can have a CVE.
- No shell means an attacker who gets code execution can't just `sh`
  their way around.
- Multi-stage builds (week 4) are how you get here: build fat, ship thin.

</v-clicks>

---
---

# Scanning: know what you ship

```bash
trivy image --severity HIGH,CRITICAL myimg
```

<v-clicks>

- Scanners (Trivy, Grype, Docker Scout) match the packages in your image
  against public vulnerability databases (CVEs).
- Run them in CI — a new CVE can appear for an image you built months ago.
- Most findings in a fat image are in packages your app never uses.
  Minimal bases make the report short and meaningful.
- Related ideas you'll meet at work: **SBOMs** (a bill of materials for
  an image) and **image signing** (cosign) — proof of what and who.

</v-clicks>

---
---

# The hardening checklist

<v-clicks>

- [ ] Every `FROM` pinned — tag or digest, never `latest`
- [ ] Multi-stage: the final image has the app and nothing else
- [ ] Non-root `USER`, numeric uid
- [ ] No secrets in `ENV`, `ARG`, or copied files — `.dockerignore`
- [ ] Runs with `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges`
- [ ] Scanned in CI

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
This is today's practice, line by line.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Module wrap-up: containers

---
---

# Weeks 3–6 in one slide

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

| Week | You learned to… |
|---|---|
| 3 | run, inspect and stop containers; images vs containers |
| 4 | write Containerfiles; layers, caching, multi-stage |
| 5 | connect containers; volumes; Compose |
| 6 | ship them safely |

</div>
<div v-click>

The whole module, as one sentence:

<div class="mt-4 text-base italic">
"Build a small, pinned, non-root image without secrets; run it with only
the network access and storage it needs; describe all of that in a file
that lives in git."
</div>

</div>
</div>

---
---

# Lab 01 — Red Hat Academy DO188

<v-clicks>

- **Due Sunday, Oct 11, 23:59 (Almaty).** Graded from your DO188
  completion on Red Hat Academy: completion × 15 points, frozen at the
  deadline.
- Maru syncs your RHA progress every hour — the **Red Hat labs** section
  on your dashboard shows your progress, the deadline, and **How to get
  in** if you haven't registered yet.
- Register with your **SDU email**, and your **student ID** as the
  username — otherwise your progress can't reach your grade.
- Next module (week 7+): **automation with Ansible** — Lab 02 is RHA
  RH294, due Wed, Nov 4.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 6
## Today's practice

---
---

# Today's practice — harden a Containerfile

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

Practice 06 gives you a Go app and a Containerfile that **works** — but:

1. `FROM golang:latest` — unpinned, ~1 GB, ships a compiler
2. No `USER` — runs as root
3. `ENV API_TOKEN=...` — a secret baked in
4. `COPY . .` — copies `.env` too

Fix all of it. Don't touch `main.go`.

</div>
<div>

```bash
docker build -t p06 -f Containerfile .
docker run --rm -p 8080:8080 \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -e API_TOKEN=x p06
curl localhost:8080
# INF345 Practice 06 OK uid=65532 token=set
```

<div v-click class="mt-4 opacity-70">
Graded automatically: pinned bases, size under 50 MB, non-root, token
nowhere in the image, works hardened.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain why a root process in a container is a risk to the host
- [ ] Write a Containerfile that runs as a non-root, numeric uid
- [ ] Run a container with no capabilities on a read-only filesystem
- [ ] Find a secret in an image — and keep one out of it
- [ ] Choose and pin a minimal base image, and scan the result

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice if you didn't wrap it up in session — same
      Maru flow as last week
- [ ] **Lab 01 (RHA DO188) is due Sunday, Oct 11, 23:59** — check
      "Red Hat labs" in Maru
- [ ] Start RHA **RH294** — Lab 02 is due in week 10

<div class="mt-8 text-sm opacity-60">
That closes the containers module. Next: automating the machines the
containers run on.
</div>

---
layout: end
---

# Next lecture

Configuration management & Ansible basics — inventories, ad-hoc
commands, and your first playbook.
