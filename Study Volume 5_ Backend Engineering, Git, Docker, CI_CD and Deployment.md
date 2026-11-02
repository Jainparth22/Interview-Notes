# Study Volume 5: Backend Engineering, Git, Docker, CI/CD and Deployment

Fifth volume of the in-depth study set, same template as before: the idea, how it works, a diagram, runnable code or real config, the loopholes where people go wrong, interview questions, and a **Revise in 30 seconds** box. This volume is the practical engineering layer that interviewers probe through your projects: how you collaborate with Git, package with Docker, test and ship with CI/CD, run background jobs, observe a live system, and deploy to a platform. Your stack (FastAPI or Flask, PostgreSQL, Redis, Celery with Beat, Docker Compose, Railway, Render, fly.io, Vercel with Neon) is used throughout.

**How this page was checked:** the Git session in Part 1 was actually run and the outputs shown are real. The Python demos in later parts (concurrency test, cache stampede, rate limit, outbox, circuit breaker, JSON logging and metrics, graceful shutdown, config loader) were run and their outputs are quoted. The YAML files were parsed. The Celery configuration was loaded and its tasks and beat schedule registered, but retries were not executed because that needs a running broker and worker. Docker itself could not be run in the authoring environment, so Dockerfile and Compose behaviour is described from standard documented behaviour. Test those two, and the Celery retries, on your machine.

## Contents of this volume

- Part 1: Git in depth: internals, daily workflow, undo and recovery, merge and rebase, bisect, branching strategies, team practice
- Part 2: Docker and Docker Compose: images, containers, layers, Dockerfile, Compose for a FastAPI, Celery, Redis, PostgreSQL stack, debugging
- Part 3: CI/CD with GitHub Actions, deployment strategies, configuration and secrets, migrations
- Part 4: Testing: pytest, the pyramid, a real concurrency test for double booking
- Part 5: Redis caching, Celery background jobs, the outbox pattern, circuit breaker
- Part 6: Observability: logging, metrics, tracing, health checks, SLOs
- Part 7: Cloud deployment, graceful shutdown, performance, troubleshooting runbook

Parts 1 and 2 are in this first section, and the rest follow below.

## Part 1: Git in depth

### 1.1 The mental model: a graph of snapshots

**The idea.** Git is a **content-addressable object database** plus a few movable pointers. It does not store diffs as its primary data, it stores **snapshots**, and names each object by the SHA hash of its content. Understanding four object types explains almost every command.

| Object | What it holds |
| --- | --- |
| **blob** | The contents of one file (no name) |
| **tree** | A directory: names mapped to blobs and sub-trees |
| **commit** | A pointer to a tree (the snapshot), parent commit(s), author, committer, message |
| **tag** (annotated) | A named, signed pointer to a commit |

```text
 commit d606454 ("third")  --parent-->  commit b8a2368 ("second")  --parent-->  commit 402e4cd ("first")
        |                                       |                                      |
      tree (snapshot of the project)          tree                                   tree
```

- A **branch** is just a **movable pointer (a file containing a commit hash)** to a commit. Creating a branch is nearly free. **`HEAD`** points to the current branch (or directly to a commit in 'detached HEAD' state).
- History is a **directed acyclic graph (DAG)** of commits. Each commit knows its parents, never its children. A merge commit has two parents.
- Because the hash covers content **and the parent hash**, changing any commit changes the hash of every descendant. That is why rewriting history (amend, rebase, reset) creates **new commits**, and why force-pushing rewritten history disrupts anyone who already pulled the old ones.
- **Three areas:** the **working directory** (your files), the **staging area or index** (what the next commit will contain), and the **repository** (committed history). `git add` copies working changes into the index, `git commit` turns the index into a commit.

You can see the objects directly. This is real output from the verification session:

```text
$ git cat-file -t HEAD
commit
$ git cat-file -p HEAD
tree 725322682524f6bb2c5ddf2d4c2bcad4adf95c6b
parent b8a236860042d2ed4a44fc8792993b1e61cd05ef
author demo <a@b.c> 1793210874 +0000
committer demo <a@b.c> 1793210874 +0000
```

### 1.2 Daily commands, grouped by purpose

```bash
# start
git clone <url>                 # copy a repository with full history
git init -b main                # new repository

# look
git status                      # what changed, what is staged
git diff                        # working directory vs index (unstaged)
git diff --staged               # index vs last commit (what will be committed)
git log --oneline --graph --decorate --all
git show <commit>               # one commit's message and diff
git blame <file>                # who last changed each line
git log -S 'some_function'      # commits that added or removed that text (the 'pickaxe')

# record
git add -p                      # stage hunks interactively (keeps commits focused)
git commit -m 'Add slot hold expiry sweep'
git commit --amend              # rewrite the LAST commit (only if not pushed)

# branches
git switch -c feature/hold-expiry     # create and switch (older: git checkout -b)
git switch main
git branch -d feature/hold-expiry     # delete a merged branch

# remotes
git fetch                       # download new commits, change nothing locally
git pull --rebase               # fetch, then replay your local commits on top
git push -u origin feature/hold-expiry
git push --force-with-lease     # safer force push: refuses if someone else pushed meanwhile
```

**Fetch vs pull.** `fetch` only downloads. `pull` is `fetch` plus a merge (or a rebase with `--rebase`). Fetch first and look, when you want to be careful.

**Good commit habits:** small, focused commits that each build and pass tests, an imperative subject line under about 70 characters ('Fix double booking on concurrent holds'), a body that explains **why**, not what. **Conventional Commits** (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`) let tools generate changelogs and version numbers.

### 1.3 Undoing things: the table you must know

| Goal | Command | Notes |
| --- | --- | --- |
| Discard unstaged edits in a file | `git restore <file>` | Destructive, cannot be undone |
| Unstage a file | `git restore --staged <file>` | Keeps the edits |
| Fix the last commit (message or content) | `git commit --amend` | Rewrites that commit, only if unpushed |
| Move the branch back, keep the changes staged | `git reset --soft HEAD~1` | Good for 'redo this commit' |
| Move back, keep changes unstaged | `git reset --mixed HEAD~1` (default) |  |
| Move back and **throw the changes away** | `git reset --hard HEAD~1` | Destructive for uncommitted work |
| Undo a commit **that is already shared** | `git revert <commit>` | Adds a new commit that reverses it, history stays intact |
| Shelve work temporarily | `git stash` then `git stash pop` | Includes tracked changes by default |
| Delete untracked files | `git clean -fd` (try `-n` first) | Destructive |

**Rule of thumb: `reset` rewrites history (private work only), `revert` adds history (shared branches).**

**The safety net: `git reflog`.** Git records every place `HEAD` has been. After a bad `reset --hard`, you can find and restore the 'lost' commit. Real session:

```text
$ git reset --hard HEAD~1                    # oops, dropped the 'third' commit
$ git reflog
b8a2368 HEAD@{0}: reset: moving to HEAD~1
d606454 HEAD@{1}: commit: third (will be lost)
b8a2368 HEAD@{2}: commit: second
402e4cd HEAD@{3}: commit (initial): first
$ git reset --hard d606454                   # recovered: the commit was never deleted, only unreferenced
$ git log --oneline
d606454 third (will be lost)
b8a2368 second
402e4cd first
```

Commits become truly unreachable only after the reflog entry expires (about 90 days by default) and garbage collection runs. **Uncommitted changes destroyed by `reset --hard` or `restore` are not in the reflog**, which is the real danger.

`git revert <commit>` in the same session produced `Revert "third (will be lost)"` as a new commit on top, leaving the original in history.

### 1.4 Merge vs rebase

Two ways to combine work from one branch into another.

```text
Before:        main:     A---B---C
                              \
               feature:        D---E

MERGE (main gets a merge commit, history is truthful but branchy):
               A---B---C-------M
                    \         /
                     D-------E

REBASE feature onto main (feature commits are replayed as NEW commits D' and E', history is linear):
               A---B---C---D'---E'
```

- **Merge** keeps the true history and never rewrites commits. If the target has not diverged, Git just moves the pointer (**fast-forward**). `--no-ff` forces a merge commit to record that a feature existed. A **squash merge** collapses the whole feature into one commit on main.
- **Rebase** replays your commits on top of the new base, producing a **linear history** that is easier to read and bisect. It rewrites your commits (new hashes).
- **The golden rule of rebasing: never rebase commits that other people have already pulled** (public or shared branches). Rebase your own local, unpushed work (or your own feature branch, then `--force-with-lease`).
- Verified session: after `main` gained a `hotfix` commit, `git rebase main` on `feature` produced a linear history (`feat` on top of `hotfix` on top of `c6`), and `git merge --ff-only feature` on main then simply moved the pointer.
- **Interactive rebase** (`git rebase -i HEAD~4`) lets you **squash, reword, reorder, drop and edit** commits to tidy a branch before opening a pull request.
- **Which to choose:** a typical team rule is 'rebase your feature branch on main to stay current, then merge via a pull request (squash or merge commit)'. The best answer is to say that the team should pick one convention and apply it consistently.

**Cherry-pick:** `git cherry-pick <hash>` copies one commit onto the current branch, for example a hotfix that must also go to a release branch. The copy has a different hash.

### 1.5 Conflicts

A conflict occurs when both sides changed the same lines. Git stops and marks the file:

```text
<<<<<<< HEAD
timeout = 30
=======
timeout = 60
>>>>>>> feature/slow-api
```

Resolve by editing the file to the intended result and removing the markers, then `git add <file>`, then `git commit` (merge) or `git rebase --continue`. `git merge --abort` or `git rebase --abort` returns you to the state before. **Keep conflicts small:** pull or rebase often, keep branches short-lived, avoid mass reformatting in feature branches. `git rerere` can remember how you resolved a repeated conflict.

### 1.6 Bisect: find the commit that broke something

Binary search over history. You tell Git one good and one bad commit, it checks out the middle, and you say good or bad until it names the first bad commit. With a script it is fully automatic. Verified session, where a `BUG` line first appears at commit 4 of 6:

```text
$ git bisect start HEAD HEAD~5          # bad = HEAD, good = 5 commits ago
$ git bisect run sh -c '! grep -q BUG f.txt'     # exit 0 = good, non-zero = bad
6f5fc92aef85d93ec57f16b10f4d9a77f63f28e7 is the first bad commit
$ git bisect reset
```

For N commits it needs about log2(N) tests. This is the answer to 'a test started failing sometime last week'.

### 1.7 Tags, hooks, ignore files, secrets

- **Tags** mark releases: `git tag -a v1.2.0 -m 'release 1.2.0'` and `git push --tags`. Use **semantic versioning**: MAJOR.MINOR.PATCH (breaking change, new feature, bug fix).
- **Hooks** are scripts Git runs at events. `pre-commit` (lint and format before a commit is made, managed with the `pre-commit` tool), `commit-msg` (enforce message format), `pre-push` (run tests). Hooks are local, so enforce the same checks in CI as well.
- **`.gitignore`** keeps build output, virtual environments (`venv/`, `node_modules/`), caches, and **`.env` files** out of the repository. It does not affect files that are already tracked (`git rm --cached <file>` first).
- **A secret committed once is compromised forever.** Deleting it in a later commit does not remove it from history, and anyone with a clone has it. The correct response: **rotate the secret immediately**, then (optionally) purge history with `git filter-repo`, and add secret scanning (`gitleaks`, GitHub secret scanning) and a pre-commit check. Mention this in security questions.
- **Large files:** do not commit big binaries or datasets (the repository grows forever). Use Git LFS or external storage.

### 1.8 Branching strategies and team workflow

- **Trunk-based development:** everyone merges small changes to `main` at least daily, behind **feature flags** when unfinished. Needs strong CI and tests. Fast, minimal merge pain, the modern default for high-performing teams.
- **GitHub flow:** short-lived feature branch, pull request, review, CI passes, merge to `main`, deploy from `main`. Simple and widely used.
- **Gitflow:** long-lived `develop`, plus `release/*` and `hotfix/*` branches. Suits versioned, scheduled releases (installed software), heavier for web services.
- **Pull request practice:** small and focused (under a few hundred changed lines), a description that says what and why, linked issue, passing CI, at least one reviewer, and respond to feedback with commits, not arguments. **As a reviewer,** check correctness, tests, security, and clarity, and leave style to linters. Protect `main`: require reviews and passing checks, forbid force pushes.
- **Forks vs branches:** open source uses forks (you cannot push to the upstream), teams usually use branches in one repository.
- **Signed commits** (`git commit -S`) prove authorship, useful for supply-chain security.

### Loopholes and traps in Git

- `git pull` with local changes and divergent histories creates surprise merge commits. Prefer `git pull --rebase` or fetch first.
- `git push --force` can overwrite a teammate's work. Use `--force-with-lease`.
- `git reset --hard` and `git restore` destroy uncommitted changes with no reflog safety net.
- 'Detached HEAD': you checked out a commit directly. Commits you make there belong to no branch, so run `git switch -c rescue` before leaving.
- Case-insensitive file systems (macOS, Windows) can hide a file rename that only changes case. Use `git mv`.
- Line endings differ between operating systems. A `.gitattributes` file (`* text=auto`) and an `.editorconfig` avoid noisy diffs.
- Rebasing a branch that others branched from creates duplicate commits and chaos.

### Interview questions on Git

- **What is the difference between merge and rebase? When do you use each?** Merge preserves history with a merge commit, rebase rewrites your commits for a linear history. Rebase private branches only, merge shared ones.
- **What is the difference between reset, revert and restore?** Move the branch pointer (rewrites history), add a commit that undoes another (safe for shared history), discard file changes.
- **How do you recover a commit you deleted?** `git reflog`, find the hash, `git reset --hard <hash>` or `git switch -c rescue <hash>`.
- **A bug appeared sometime last week. How do you find the commit?** `git bisect`, ideally `bisect run` with a failing test.
- **What is a detached HEAD?** HEAD points at a commit instead of a branch. Create a branch to keep new commits.
- **You committed a password. What do you do?** Rotate it first, then remove it from history if needed, add scanning. Deleting it in a new commit is not enough.
- **How would you handle a hotfix?** Branch from the release or main, fix, test, merge, tag, and cherry-pick to other maintained branches.
- **What is a fast-forward merge?** The target has no new commits, so Git just moves the pointer, no merge commit.

### Revise in 30 seconds: Git

- Objects: blob, tree, commit, tag, named by hash. Branch = pointer to a commit, `HEAD` = where you are. Three areas: working directory, index, repository.
- Rewriting history makes new hashes. Merge keeps history, rebase makes it linear but never rebase shared commits. `--force-with-lease`, not `--force`.
- Undo: `restore` (files), `reset` (private history), `revert` (shared history), `stash`. `reflog` recovers lost commits, not uncommitted changes.
- `bisect run` finds the bad commit in log2(N) steps. Conflicts: edit, remove markers, `add`, `continue`.
- A leaked secret is compromised: rotate first. Trunk-based or GitHub flow with protected main, small PRs, CI required.

## Part 2: Docker and Docker Compose

### 2.1 What a container is

**The idea.** A **container** is an **ordinary process (or group of processes) on the host kernel, isolated and resource-limited**, packaged together with its filesystem so it runs the same everywhere ('works on my machine' solved). It is **not a lightweight virtual machine**: there is **no guest operating system**, every container shares the host's Linux kernel.

```text
Virtual machines                         Containers
+---------+ +---------+                  +----------+ +----------+
|  App A  | |  App B  |                  |  App A   | |  App B   |
| libs    | | libs    |                  |  libs    | |  libs    |
| Guest OS| | Guest OS|                  +----------+-+----------+
+---------+-+---------+                  |  Container runtime    |
|     Hypervisor        |                  |  Host OS kernel (shared)|
|     Host OS / HW      |                  +------------------------+
```

Two Linux kernel features make this work:

- **Namespaces** give each container its own **view** of the system: process IDs (`pid`), network interfaces (`net`), mount points (`mnt`), hostname (`uts`), IPC, and users. Inside, your process thinks it is PID 1 on its own machine.
- **cgroups (control groups)** **limit and account** resources: CPU, memory, I/O. A container exceeding its memory limit is killed by the kernel (exit code 137, the OOM kill).
- **Union (layered) filesystems** let many containers share read-only image layers and add a thin writable layer on top.

**Image vs container.** An **image** is a read-only, layered template (a filesystem snapshot plus metadata: default command, environment, exposed port). A **container** is a running (or stopped) instance of an image with a writable layer. One image can start many containers. A **registry** (Docker Hub, GitHub Container Registry, ECR) stores and distributes images, addressed as `name:tag` or immutable `name@sha256:digest`.

**Containers vs VMs:** containers start in about a second and use little memory (shared kernel), VMs give stronger isolation and can run different operating systems. Containers are **not a security boundary as strong as a VM**, so run them as a non-root user with minimal privileges.

### 2.2 Layers, the build cache, and why Dockerfile order matters

Every instruction in a Dockerfile (`RUN`, `COPY`, `ADD`) creates a **layer**. Docker caches each layer, and **a changed layer invalidates the cache for every layer after it**. So you order instructions from **least to most frequently changing**:

1. Base image and system packages (rarely change).
2. Copy **only the dependency manifest** (`requirements.txt`) and install dependencies.
3. Copy the application source last (changes every commit).

If you copy the whole source before installing dependencies, every code edit reinstalls all dependencies, which makes builds slow.

### 2.3 A production-quality Dockerfile for a FastAPI service

```dockerfile
# syntax=docker/dockerfile:1

# ---- stage 1: build dependencies in a throwaway image ----
FROM python:3.12-slim AS builder
WORKDIR /build
ENV PIP_NO_CACHE_DIR=1
COPY requirements.txt .                                   # manifest first: this layer is reused until dependencies change
RUN pip install --prefix=/install -r requirements.txt

# ---- stage 2: small runtime image ----
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1          # no .pyc clutter, logs are not buffered (they appear immediately)
RUN useradd --create-home --uid 10001 appuser             # do NOT run as root
WORKDIR /app
COPY --from=builder /install /usr/local                   # only the installed packages, no compilers or caches
COPY --chown=appuser:appuser . .                          # application source last
USER appuser
EXPOSE 8000                                               # documentation only: it does not publish the port
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)" || exit 1
# exec form (JSON array): the app is PID 1 and receives SIGTERM directly, so it shuts down gracefully
CMD ["gunicorn", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "2", "-b", "0.0.0.0:8000"]
```

Companion `.dockerignore` (keeps the build context small and secrets out of the image):

```text
.git
.venv
__pycache__
*.pyc
.env
tests
node_modules
```

**Best practices and why:**

- **Multi-stage builds** keep build tools out of the final image (smaller, fewer vulnerabilities).
- **Pin versions** (`python:3.12-slim`, pinned `requirements.txt`) for reproducible builds. For maximum reproducibility pin the image digest.
- **Slim vs Alpine:** Alpine is smaller but uses musl instead of glibc, which breaks or slows some Python wheels (NumPy, many ML packages) and forces compilation. `-slim` Debian is the safer default.
- **One main process per container.** Run the web app, the Celery worker and Celery Beat as **separate containers** from the same image, not one container with three processes.
- **Never bake secrets into an image** (`ENV API_KEY=...` or copying `.env`). Anyone with the image can read it from the layers. Inject secrets at runtime.
- **Exec form vs shell form of `CMD`:** `CMD ["gunicorn", ...]` (exec) makes your app PID 1 so signals reach it. The shell form `CMD gunicorn ...` wraps it in `sh -c`, and `docker stop`'s SIGTERM may never reach your app, so it is killed abruptly after the timeout. Use the exec form, or `--init` (a tiny init process that forwards signals and reaps zombies).
- **Log to stdout and stderr**, never to files inside the container. The platform collects them.
- **Run as a non-root user**, drop capabilities, and use a read-only filesystem where possible.

### 2.4 Docker Compose: the whole stack in one file

**Compose** defines and runs a **multi-container application** declaratively. Here is the stack for a FastAPI web app, a Celery worker, Celery Beat, PostgreSQL and Redis, the same shape as your appointment system and TaskFlow:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD in your .env file}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data            # named volume: data survives container removal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7
    command: ["redis-server", "--appendonly", "yes"]   # AOF persistence, so a restart does not lose queued jobs
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  web:
    build: .
    ports:
      - "8000:8000"                                 # host:container
    environment: &app_env                            # YAML anchor: reuse this block below
      DATABASE_URL: postgresql://app:${DB_PASSWORD}@db:5432/app    # 'db' is the service name, not localhost
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  worker:
    build: .
    command: ["celery", "-A", "app.tasks", "worker", "--loglevel=INFO", "--concurrency=2"]
    environment: *app_env
    depends_on:
      - db
      - redis

  beat:
    build: .
    command: ["celery", "-A", "app.tasks", "beat", "--loglevel=INFO"]
    environment: *app_env
    depends_on:
      - redis                                       # run exactly ONE beat, or scheduled jobs fire twice

volumes:
  pgdata:
  redisdata:
```

**Concepts to explain:**

- **Networking:** Compose creates a private network where services reach each other **by service name** (`db`, `redis`), resolved by Docker's built-in DNS. **`localhost` inside a container means that container itself**, so `localhost:5432` from the web container fails. This is the most common Compose bug.
- **`ports` vs `expose`:** `ports: "8000:8000"` publishes a container port on the host. Containers on the same network can already talk to each other without publishing anything. Publish only what the outside world needs (do not publish the database port in production).
- **Volumes:** a **named volume** (`pgdata`) is managed by Docker and persists data. A **bind mount** (`./src:/app`) maps a host folder, ideal for live-reload in development. A **tmpfs** mount keeps data in memory only. **Container filesystems are ephemeral**: data written inside a container's own layer is lost when it is removed.
- **`depends_on` controls start order, not readiness.** Without `condition: service_healthy`, `web` may start before Postgres accepts connections. Even with health checks, **your application must retry its connection on startup**, because dependencies can also restart later.
- **Config:** variables from `.env` (which Compose reads automatically for `${...}` substitution) or `env_file:`. Keep `.env` out of Git and commit a `.env.example`.
- **Environment overrides:** use `compose.override.yaml` for development (bind mounts, debug mode) and a separate production file.
- **Migrations:** run them as a **one-off job**, not inside every web instance's startup: `docker compose run --rm web alembic upgrade head`.
- **Scaling:** `docker compose up --scale worker=3`. You cannot scale a service that publishes a fixed host port, or Beat (must stay single).

**Commands to know:**

| Command | Purpose |
| --- | --- |
| `docker compose up -d --build` | Build images and start everything in the background |
| `docker compose ps` / `docker compose logs -f web` | Status and live logs of a service |
| `docker compose exec web sh` | Shell inside a running container |
| `docker compose run --rm web pytest` | One-off command in a fresh container |
| `docker compose down` / `down -v` | Stop and remove containers (`-v` also deletes volumes, **which deletes your database data**) |
| `docker build -t app:1.0 .` / `docker run -p 8000:8000 app:1.0` | Build and run a single image |
| `docker ps -a`, `docker logs <c>`, `docker inspect <c>`, `docker stats` | Inspect containers and resource use |
| `docker image prune`, `docker system df` | Reclaim disk space |

### 2.5 Debugging containers: symptoms and causes

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| Container exits immediately | The main process ended or crashed (bad command, missing env var) | `docker logs <c>`, check `CMD` and environment, check the exit code |
| Exit code **137** | Killed with SIGKILL, very often the **OOM killer** (memory limit) | Raise the limit, find the memory leak, check `docker stats` |
| Exit code **143** | Stopped with SIGTERM (normal stop) | Usually fine |
| Exit code **139** | Segmentation fault in native code | Debug the native library |
| `connection refused` to `localhost:5432` from the app container | Using `localhost` instead of the service name, or the service is not ready | Use `db:5432`, add health checks and startup retries |
| `port is already allocated` | Another process or container uses that host port | Change the host port or stop the other process |
| App works in the container but not from the host | App bound to `127.0.0.1` inside the container | Bind to `0.0.0.0` |
| Data disappeared after restart | No volume, or `down -v` was used | Use a named volume for stateful services |
| `permission denied` on a mounted volume | UID of the container user does not match the host files | Fix ownership, or run with the matching UID |
| Build is slow on every code change | Source copied before dependencies | Reorder the Dockerfile for caching |
| Image is huge | Build tools and caches left in the final image, no `.dockerignore` | Multi-stage build, `.dockerignore`, slim base |
| Changes to code not visible | Old image still running, or no bind mount | Rebuild (`up --build`) or mount the source in development |

### 2.6 Where Kubernetes fits (the overview you need)

Compose runs containers on one machine. **Kubernetes** runs and manages containers across a cluster. Vocabulary: a **Pod** (one or more containers scheduled together), a **Deployment** (declares N replicas and handles rolling updates), a **Service** (stable address and load balancing for pods), an **Ingress** (HTTP routing and TLS from outside), **ConfigMap and Secret** (configuration), **liveness probe** (restart me if I am stuck) and **readiness probe** (do not send me traffic yet), **HPA** (autoscaling). Platforms like Railway, Render and fly.io hide this and run your containers for you, which is why you can deploy without knowing Kubernetes, but expect the vocabulary in interviews.

### Loopholes in Docker answers

- 'A container is a lightweight VM' is wrong: shared kernel, namespaces plus cgroups.
- `EXPOSE` does not publish a port. `depends_on` does not wait for readiness.
- Containers are ephemeral: state belongs in volumes or external services.
- Secrets in `ENV`, build args, or copied `.env` files end up in image layers.
- `latest` tags are not reproducible. Pin versions.
- A healthcheck that only checks the process is up (not the dependencies) hides real failures. Separate liveness from readiness.

### Interview questions on Docker

- **What is the difference between an image and a container? A container and a VM?** Template versus running instance. Shared kernel with namespaces and cgroups versus a full guest OS.
- **How does Docker layer caching work, and how do you order a Dockerfile?** Least changing first, dependencies before source, because a changed layer invalidates the rest.
- **How do containers talk to each other in Compose?** A private network and DNS by service name. `localhost` is the container itself.
- **How do you persist data?** Named volumes or bind mounts. Container layers are ephemeral.
- **Why use multi-stage builds and a non-root user?** Smaller attack surface and image, least privilege.
- **Why does my container ignore SIGTERM or take ten seconds to stop?** Shell-form `CMD` so the app is not PID 1, or the app does not handle SIGTERM. Use exec form and graceful shutdown (Part 7).
- **How do you run migrations in a Compose or container deployment?** As a one-off job or a release step, not on every replica's start.
- **How do you keep secrets out of images?** Inject at runtime (environment, secret store), never `COPY` or bake them in, scan images.

### Revise in 30 seconds: Docker

- Container = isolated process on the host kernel (namespaces for isolation, cgroups for limits), not a VM. Image = read-only layered template, container = running instance.
- Layers cache. Order: base, dependency manifest and install, source last. Multi-stage builds, slim base, non-root user, `.dockerignore`, no secrets in images.
- Exec-form `CMD` so the app is PID 1 and gets SIGTERM. Log to stdout. One process per container (web, worker, beat separate).
- Compose: services reach each other by name, `localhost` is yourself, `depends_on` is order not readiness (use healthchecks plus startup retries), named volumes for data, `down -v` deletes data.
- Exit 137 = killed (often OOM), 143 = SIGTERM. Run migrations as a one-off job, run exactly one Beat.

## Part 3: CI/CD, configuration, secrets and migrations

### 3.1 CI and CD, defined precisely

- **Continuous Integration (CI):** every push and pull request is **built and tested automatically** in a clean environment. The goal is to find breakage within minutes, while the change is small and the author remembers it. A healthy CI is **fast** (under about 10 minutes), **reliable** (not flaky), and **blocking** (a red build cannot merge).
- **Continuous Delivery:** every change that passes CI produces a **deployable artifact** (an image) and could be released at any time, usually with a manual approval.
- **Continuous Deployment:** every change that passes the pipeline is **released to production automatically**.
- **Build once, deploy the same artifact everywhere.** Build the image once, tag it with the **commit SHA**, and promote that exact image through staging to production. Rebuilding per environment lets 'what you tested' differ from 'what you shipped'.

```mermaid
flowchart LR
    A[Push or pull request] --> B[Checkout and set up]
    B --> C[Lint and type check]
    C --> D[Unit and integration tests with real Postgres and Redis]
    D --> E{On main?}
    E -- no --> F[Report status on the pull request]
    E -- yes --> G[Build image, tag with commit SHA]
    G --> H[Push to registry]
    H --> I[Deploy to staging, smoke tests]
    I --> J[Approval]
    J --> K[Deploy to production, health checks, automatic rollback on failure]
```

### 3.2 A GitHub Actions workflow, line by line

Workflow files live in `.github/workflows/`. The structure: **events** (`on`), **jobs** that run in parallel unless they declare `needs`, each job runs on a **runner** and is a list of **steps** (`uses:` an action, or `run:` a shell command).

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

concurrency:                                  # a new push cancels the older still-running run of the same branch
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read                              # least privilege for the automatic token

jobs:
  test:
    runs-on: ubuntu-latest
    services:                                 # real dependencies as containers next to the job
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s --health-timeout 3s --health-retries 10
      redis:
        image: redis:7
        ports: ['6379:6379']
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-retries 10
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/app_test
      REDIS_URL: redis://localhost:6379/0
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip                          # caches downloaded packages between runs
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .
      - run: pytest -q --maxfail=1

  build-and-push:
    needs: test                               # only runs if the test job passed
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha                # reuse Docker layers between runs
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production                   # can require manual approval and holds environment secrets
    steps:
      - run: echo 'call your platform CLI or webhook here, with the image tagged by the commit SHA'
```

**Things to explain:**

- **Services** start real PostgreSQL and Redis for integration tests, which is far more trustworthy than mocking the database. Because the job runs directly on the runner, the services are reached on `localhost` with the mapped ports.
- **Secrets** (`${{ secrets.NAME }}`) are encrypted, masked in logs, and **not available to workflows triggered from forks' pull requests**, which prevents a malicious PR from exfiltrating them. Never `echo` a secret.
- **Caching** dependencies and Docker layers is the main way to keep CI fast. **Matrix builds** (`strategy.matrix`) test several Python versions or operating systems in parallel.
- **Pin third-party actions** to a version (or better, a commit SHA), because an action is code that runs with your permissions (supply-chain risk). Keep `permissions` minimal.
- **Environments** (`environment: production`) give per-environment secrets, required reviewers, and deployment history.
- Run the **same commands locally** as CI does (put them in a `Makefile` or a script), so failures reproduce on your machine.

### 3.3 Deployment strategies

| Strategy | How it works | Pros | Cons |
| --- | --- | --- | --- |
| **Recreate** | Stop the old version, start the new one | Simplest | Downtime |
| **Rolling update** | Replace instances a few at a time, behind a load balancer | No downtime, little extra capacity | Old and new versions run **together**, so changes must be **backward compatible** (especially the database) |
| **Blue-green** | Run the new version (green) beside the old (blue), switch traffic at once, keep blue for instant rollback | Fast, clean rollback | Needs double capacity, shared database still needs compatibility |
| **Canary** | Send a small percentage of traffic to the new version, watch error rate and latency, then increase | Limits blast radius, real-traffic validation | Needs good metrics and routing |
| **Feature flags** | Ship code dark, enable per user or percentage at runtime | Decouples deploy from release, instant off-switch | Flag debt, more test combinations |

**Rollback plan:** you should always be able to answer 'how do we undo this in under five minutes?'. Redeploying the previous image tag is the usual answer. A **database migration that cannot be reversed** is the thing that breaks this, which leads to the next section.

### 3.4 Database migrations without downtime

**The idea.** The schema changes over time and must change **in step with the code**, reproducibly, in every environment. A migration tool (**Alembic** for SQLAlchemy, Django migrations, **Prisma Migrate**) keeps ordered, version-controlled migration scripts and records which have been applied.

```python
"""add held_until to slots"""
from alembic import op
import sqlalchemy as sa

revision = '20261028_01'
down_revision = '20261015_02'

def upgrade():
    op.add_column('slots', sa.Column('held_until', sa.DateTime(timezone=True), nullable=True))
    op.create_index('ix_slots_held', 'slots', ['held_until'],
                    postgresql_where=sa.text("status = 'held'"))     # partial index, covers only the rows that matter

def downgrade():
    op.drop_index('ix_slots_held', table_name='slots')
    op.drop_column('slots', 'held_until')
```

**The core problem:** during a rolling deploy, **old code and new code run against the same database at the same time**. A migration that the old code cannot tolerate (dropping or renaming a column) causes errors in the old instances that are still running. The solution is the **expand and contract** pattern:

1. **Expand:** add the new column or table as **nullable or with a default**, no code depends on it yet. Deploy the migration.
2. **Deploy code that writes to both** old and new places (and reads the old).
3. **Backfill** existing rows in batches (not one giant `UPDATE` that locks the table).
4. **Deploy code that reads the new place.**
5. **Contract:** after every instance runs the new code, drop the old column in a later migration.

**Dangerous operations on big PostgreSQL tables:**

- `CREATE INDEX` blocks writes. Use **`CREATE INDEX CONCURRENTLY`** (cannot run inside a transaction block).
- Adding a column with a **volatile default** or changing a column type can rewrite the whole table under a lock. Add nullable first, backfill, then add the constraint (`NOT VALID` then `VALIDATE CONSTRAINT`).
- Renaming or dropping columns that live code uses.
- Long-running migrations holding locks queue every other query behind them. Set a **`lock_timeout`** so a migration fails fast instead of stalling production.

**Where to run migrations:** as a **separate release step or one-off job before the new version receives traffic**, never concurrently from every replica's startup (they would race). Back up before destructive changes, and test the migration on a copy of production-sized data.

### 3.5 Configuration and secrets

**Twelve-factor rules that matter most:**

- **Config lives in the environment**, not in code. The same image runs in development, staging and production, and only the environment variables differ.
- **Strict separation of config from code** means you could open-source the repository without leaking credentials.
- **Logs go to stdout.** Processes are **stateless**, state lives in backing services (database, Redis, object storage). **Dev and prod parity:** use the same kinds of services (real PostgreSQL in development, not SQLite, if production uses PostgreSQL).

```python
import os
from dataclasses import dataclass

class ConfigError(Exception): pass

def _required(name):
    try:
        return os.environ[name]
    except KeyError:
        raise ConfigError(f'missing required environment variable {name}') from None

@dataclass(frozen=True)
class Settings:                                   # one typed, validated, immutable object, built once at startup
    database_url: str
    redis_url: str
    debug: bool
    hold_minutes: int

    @classmethod
    def from_env(cls):
        return cls(
            database_url=_required('DATABASE_URL'),                          # fail fast at boot, not at the first request
            redis_url=os.environ.get('REDIS_URL', 'redis://localhost:6379/0'),
            debug=os.environ.get('DEBUG', '0') == '1',
            hold_minutes=int(os.environ.get('HOLD_MINUTES', '10')),
        )
```

In FastAPI projects, **`pydantic-settings`** does the same with type coercion and `.env` support: `class Settings(BaseSettings): database_url: str; debug: bool = False`. The principle to state is **validate configuration at startup and crash early with a clear message**, rather than failing mysteriously at the first request.

**Secrets management:**

- Keep secrets out of the repository, out of images, and out of logs. Inject them at runtime from the platform's secret store (Railway, Render and fly.io variables or secrets, GitHub Actions secrets, cloud secret managers, Vault).
- **Rotate** them on a schedule and immediately after any suspected exposure. Give each environment its own secrets and each service the **least privilege** it needs.
- Commit a **`.env.example`** that lists names with fake values, so teammates know what to set. Never commit `.env`.
- **Encrypt sensitive data at rest** in the database (for example OAuth refresh tokens, using authenticated encryption such as Fernet), keep the key separately from the data, and plan **key rotation** (decrypt with the old key, re-encrypt with the new one).
- Distinguish **build-time** variables (baked into frontend bundles and therefore **public**, e.g. `NEXT_PUBLIC_*` in Next.js) from **runtime** secrets. A secret placed in a client-side variable is exposed to every visitor.

### Loopholes in CI/CD and config

- A pipeline that deploys on every merge without good tests just ships bugs faster.
- **Flaky tests** teach people to ignore red builds. Quarantine and fix them.
- Environment drift (staging differs from production) makes 'it passed in staging' meaningless.
- Running migrations from every replica at startup causes races and partial upgrades.
- `latest` image tags in deployment make rollbacks and audits impossible. Deploy by **commit SHA**.
- Secrets printed in logs, committed in `.env`, or passed as build args remain recoverable.

### Interview questions on CI/CD and config

- **What is the difference between continuous delivery and continuous deployment?** Always releasable with a manual approval versus automatic release of every passing change.
- **How do you deploy without downtime when the schema changes?** Rolling or blue-green deploys plus expand and contract migrations, backward-compatible code, concurrent index creation.
- **How do you roll back?** Redeploy the previous image tag. Migrations are designed to be backward compatible so rollback is safe. Feature flags give an instant off-switch.
- **Why build the image once?** Tested artifact equals deployed artifact.
- **How do you handle secrets in CI?** Encrypted secrets, least-privilege tokens, not exposed to fork PRs, masked in logs, short-lived credentials where possible.
- **What would you put in your pipeline?** Lint, type check, unit tests, integration tests with real services, security and dependency scanning, build, push, deploy, smoke test, rollback on failed health checks.
- **What is a canary release and what do you watch?** Small traffic share on the new version, comparing error rate, latency and saturation against the baseline before widening.

### Revise in 30 seconds: CI/CD and config

- CI = build and test every change automatically (fast, reliable, blocking). CD = releasable artifact, optional auto-deploy. Build once, tag by SHA, promote the same image.
- Actions: events, jobs, steps, services for real Postgres and Redis, secrets not exposed to fork PRs, cache dependencies, pin actions, least permissions, environments with approvals.
- Strategies: recreate, rolling (versions overlap), blue-green, canary, feature flags. Always have a rollback.
- Migrations: expand then contract, backfill in batches, `CREATE INDEX CONCURRENTLY`, `lock_timeout`, run as a release step not per replica.
- Config in environment, validated at startup, `.env.example` committed, secrets from a secret store, encrypt sensitive data at rest, `NEXT_PUBLIC_*` values are public.

## Part 4: Testing

### 4.1 The test pyramid and what each level is for

```text
            /\              few       End-to-end: a real browser or full stack, slow, brittle, covers critical user journeys
           /  \
          /----\            some      Integration: real database, Redis, HTTP layer together, catches wiring and SQL bugs
         /      \
        /--------\          many      Unit: one function or class in isolation, milliseconds, pinpoints the bug
```

- **Unit tests** are fast and precise. They need your code to be testable (dependency injection, injected clock, no hidden globals, see Volume 4).
- **Integration tests** run against a **real PostgreSQL and Redis** (CI services, or containers started by the test suite), because SQL, constraints, transactions and locking behave differently from mocks, and many real bugs live exactly there.
- **End-to-end tests** cover only the few journeys that must never break (sign up, book, pay).
- **Contract tests** verify that a service and its consumers agree on the API shape. **Load tests** (Locust, k6, wrk) find capacity limits and regressions. **Property-based tests** (Hypothesis) generate many inputs to find edge cases.

### 4.2 pytest essentials

- Tests are plain functions named `test_*` with plain `assert` (pytest rewrites assertions to show useful diffs).
- **Fixtures** (`@pytest.fixture`) provide setup and teardown and are injected by parameter name. Scope them (`function`, `module`, `session`) to balance isolation and speed. `tmp_path` gives a fresh temporary directory. `monkeypatch` changes environment variables and attributes safely for one test.
- **Parametrisation** (`@pytest.mark.parametrize`) runs one test over many inputs.
- **Markers** (`@pytest.mark.slow`) select tests (`pytest -m 'not slow'`).
- **Isolation:** each test must set up its own data and not depend on order. For databases, wrap each test in a transaction that is rolled back, or use a fresh schema per test.
- **Mocks, stubs, fakes:** a **fake** (a working in-memory version, like the in-memory repository in Volume 4) is usually better than a heavy **mock** that asserts on every call, because mocks tie tests to implementation details and break on harmless refactors. Mock only at **system boundaries** (third-party HTTP APIs, email, payment), never your own internals.
- **Control time and randomness:** inject a clock or use a time-freezing library, seed random generators, never use `sleep` to wait for things in tests.

```python
# FastAPI route test with the built-in test client (no server needed)
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health():
    r = client.get('/health')
    assert r.status_code == 200
```

### 4.3 Testing the dangerous thing: a real concurrency test for double booking

Concurrency bugs are the ones that pass every ordinary test and fail in production. You can test them deliberately: start many threads, **release them at the same instant with a barrier**, and assert the invariant. This is the module under test (the same check-then-act versus atomic compare-and-swap from Volumes 1 and 2) and the tests, which were run for real with SQLite and passed:

```python
# booking.py
import sqlite3, time

SCHEMA = '''
CREATE TABLE IF NOT EXISTS slots (
    id INTEGER PRIMARY KEY,
    status TEXT NOT NULL DEFAULT 'free' CHECK (status IN ('free', 'held', 'booked')),
    user TEXT
)'''

def connect(path):
    conn = sqlite3.connect(path, timeout=10, isolation_level=None)    # autocommit: each statement is its own transaction
    conn.execute('PRAGMA busy_timeout = 10000')
    return conn

def init(path, n_slots=1):
    conn = connect(path)
    conn.execute(SCHEMA)
    conn.executemany('INSERT INTO slots (id) VALUES (?)', [(i,) for i in range(1, n_slots + 1)])
    conn.close()

def hold_unsafe(conn, slot_id, user, pause=0.0):         # check-then-act: has a race window
    row = conn.execute('SELECT status FROM slots WHERE id = ?', (slot_id,)).fetchone()
    time.sleep(pause)                                     # simulates real work between the check and the act
    if row and row[0] == 'free':
        conn.execute('UPDATE slots SET status = ?, user = ? WHERE id = ?', ('held', user, slot_id))
        return True
    return False

def hold_safe(conn, slot_id, user):                       # atomic compare-and-swap in SQL
    cur = conn.execute('UPDATE slots SET status = ?, user = ? WHERE id = ? AND status = ?',
                       ('held', user, slot_id, 'free'))
    return cur.rowcount == 1                              # exactly one caller can see rowcount 1
```

```python
# test_booking.py
import threading
import pytest
import booking

def run_race(db_path, hold_fn, n_threads=20):
    winners, barrier = [], threading.Barrier(n_threads)
    def worker(i):
        conn = booking.connect(db_path)                   # one connection per thread
        barrier.wait()                                     # release every thread at the same instant
        if hold_fn(conn, 1, f'user{i}'):
            winners.append(i)
        conn.close()
    ts = [threading.Thread(target=worker, args=(i,)) for i in range(n_threads)]
    [t.start() for t in ts]; [t.join() for t in ts]
    return winners

@pytest.fixture
def db(tmp_path):
    path = str(tmp_path / 'test.db')
    booking.init(path, n_slots=1)
    return path

@pytest.mark.parametrize('n_threads', [2, 10, 50])
def test_safe_hold_has_exactly_one_winner(db, n_threads):
    assert len(run_race(db, booking.hold_safe, n_threads)) == 1

def test_unsafe_hold_double_books_when_the_window_is_wide(db):
    slow_unsafe = lambda conn, sid, user: booking.hold_unsafe(conn, sid, user, pause=0.05)
    assert len(run_race(db, slow_unsafe, 10)) > 1         # documents the bug: more than one 'winner'

def test_second_hold_fails_after_first(db):
    conn = booking.connect(db)
    assert booking.hold_safe(conn, 1, 'a') is True
    assert booking.hold_safe(conn, 1, 'b') is False
```

**What the experiment showed** (10 threads, 5 rounds each, real run, exact numbers vary by machine):

```text
unsafe, no artificial pause  : winners per round [6, 8, 8, 7, 8]    <- several users 'got' the same slot
unsafe, 50 ms pause          : winners per round [10, 10, 10, 10, 10]   <- all ten
safe (atomic UPDATE)         : winners per round [1, 1, 1, 1, 1]    <- always exactly one
```

**Lessons to state in an interview:**

- The check-then-act version double books **even with no artificial delay**, and the wider the gap between check and act (a network call, a validation step), the worse it gets. In production that gap is real.
- A single-threaded test would never have caught it. **You need real parallelism and a barrier** to line threads up.
- The fix is not a bigger `if`, it is making the check and the act **one atomic statement** and **checking the affected row count**.
- Races are probabilistic, so a passing run proves little for the unsafe code, but for the safe code the **invariant (exactly one winner)** must hold on every run. Repeat the test many times in CI to build confidence.
- Against PostgreSQL in CI, the same test uses the same SQL, and you would add a test at the HTTP level (many simultaneous `POST /book` calls, expect exactly one `201` and the rest `409`).

### 4.4 Test quality, coverage and flakiness

- **Coverage** tells you which lines ran, not whether behaviour is verified. 100 percent coverage with no assertions is meaningless. Use it to find untested areas, not as a goal.
- **Flaky tests** (pass and fail without code changes) come from time, ordering, shared state, real network calls, and unsynchronised concurrency. Fix the cause (inject a clock, isolate data, wait on conditions not sleeps) instead of re-running until green.
- **Test behaviour, not implementation.** Test through public interfaces, so refactoring does not break tests.
- **Name tests as sentences** (`test_second_hold_fails_after_first`), keep them small, one reason to fail, arrange-act-assert.
- **Test the unhappy paths:** invalid input, timeouts, dependency down (Redis fails open as designed), duplicate requests (idempotency), permissions.

### Interview questions on testing

- **What is the test pyramid?** Many fast unit tests, fewer integration tests, very few end-to-end tests, for speed and reliability.
- **Mock vs fake vs stub? When do you avoid mocks?** Recorded interactions versus a lightweight working implementation versus canned answers. Avoid mocking your own code and databases, mock only external boundaries.
- **How do you test code that depends on time?** Inject a clock, or freeze time, never sleep.
- **How would you test for a race condition?** Many threads released by a barrier, assert the invariant, repeat, and test against the real database engine.
- **What does 90 percent coverage tell you?** Which lines executed, not that they are correct.
- **How do you deal with a flaky test?** Find the nondeterminism (time, order, shared state, concurrency) and remove it.

### Revise in 30 seconds: testing

- Pyramid: many unit, some integration with real Postgres and Redis, few end-to-end. Fixtures, parametrize, `tmp_path`, `monkeypatch`.
- Prefer fakes to mocks, mock only external boundaries, inject the clock, never `sleep` in tests, isolate data per test.
- Concurrency test = threads plus a barrier plus an invariant. Measured: check-then-act let 6 to 10 of 10 threads win, atomic `UPDATE ... WHERE status = 'free'` always exactly 1.
- Coverage is not correctness. Fix flakiness at the cause.

## Part 5: Redis caching, background jobs, the outbox, and resilience

### 5.1 Caching with Redis: what, how, and what goes wrong

**The idea.** A database read costs milliseconds and does real work. A Redis read costs a fraction of a millisecond from memory. Caching stores the result of an expensive or frequent read so most requests skip the database. It trades **freshness and complexity** for **speed and lower load**. Cache only what is read far more often than it changes, and decide **how stale it may be** before you design anything.

**Cache-aside (lazy loading), the default pattern:**

```mermaid
sequenceDiagram
    participant A as App
    participant R as Redis
    participant D as Database
    A->>R: GET user:42
    alt cache hit
        R-->>A: value
    else cache miss
        R-->>A: nil
        A->>D: SELECT ... WHERE id = 42
        D-->>A: row
        A->>R: SET user:42 value EX 300
    end
    Note over A,R: on a write: update the database, then DELETE user:42 so the next read rebuilds it
```

- The application owns the logic, and **the cache can fail without breaking correctness** (the database is still the source of truth).
- Alternatives: **read-through** (the cache layer loads on a miss), **write-through** (write to cache and database together, no stale reads, slower writes), **write-behind** (write to cache and flush to the database later, fast but data can be lost if the cache dies before flushing).

**Invalidation: the hard part.** Options in practice: **TTL expiry** (simple, bounded staleness, always set one), **delete on write** (the usual choice, `DEL` the key after the database commit), and **versioned keys** (`user:42:v7`, bump the version on change so old entries are simply never read). Know the **stale refill race**: reader A misses and reads the old row, a writer updates and deletes the key, then A writes its **old** value into the cache, which now holds stale data until the TTL. Mitigations: short TTLs as a safety net, delete **after** commit, and versioning.

**The failure modes you must name:**

| Problem | What happens | Defence |
| --- | --- | --- |
| **Cache stampede (thundering herd)** | A hot key expires and many requests miss at once, all hitting the database | A rebuild lock (`SET lock:key 1 NX PX 5000`, only the winner rebuilds), serve stale while refreshing, early refresh |
| **Cache avalanche** | Many keys expire at the same moment (all given the same TTL), a wave of misses | **Jitter** the TTLs (`ttl + random`), stagger warm-ups |
| **Cache penetration** | Requests for keys that do not exist always miss and hit the database (an attack or a bug) | Cache a **null marker** with a short TTL, validate input, a Bloom filter in front |
| **Hot key** | One key gets enormous traffic and overloads a single Redis shard | A local in-process cache in front, replicate the key |
| **Stale data** | The cache holds old values | Delete on write, TTLs, versioned keys |
| **Memory pressure** | Redis fills up | Set `maxmemory` and an eviction policy (`allkeys-lru`), TTL everything |

**The stampede defence, run for real** (using the real `redis-py` API against an in-memory Redis server stand-in): 20 threads request the same missing key at the same instant.

```python
import json, random, time

def make_protected(r, loader, ttl=60, lock_ms=5000):
    def get(key):
        raw = r.get(key)
        if raw is not None:
            return json.loads(raw)
        lock_key = f'lock:{key}'
        if r.set(lock_key, '1', nx=True, px=lock_ms):            # exactly one caller wins the right to rebuild
            try:
                value = loader(key)
                r.set(key, json.dumps(value), ex=ttl + random.randint(0, max(1, ttl // 10)))   # jittered TTL
                return value
            finally:
                r.delete(lock_key)
        for _ in range(200):                                     # the others wait briefly for the winner's result
            time.sleep(0.01)
            raw = r.get(key)
            if raw is not None:
                return json.loads(raw)
        return loader(key)                                       # last resort: correctness over protection
    return get
```

```text
database loads for 20 simultaneous misses -> naive cache-aside: 20 | with the rebuild lock: 1
```

**Refinements to mention:** the lock has a **TTL** so a crashed holder cannot block forever. A safe release stores a **unique token** as the lock value and deletes only if it still matches (a small Lua compare-and-delete script), otherwise a slow holder could delete a lock that expired and was taken by someone else. `SET ... NX PX` is atomic, which is why it works (never do `GET` then `SET`, that is the TOCTOU race again).

**Rate limiting with `INCR`** (fixed window), verified output for a limit of 3:

```python
def allow(r, key, limit, window):
    n = r.incr(key)                 # atomic increment, creates the key at 1
    if n == 1:
        r.expire(key, window)       # first hit starts the window
    return n <= limit
# [True, True, True, False, False]
```

**Loophole:** if the process dies between `INCR` and `EXPIRE`, the key never expires and that user is limited forever. Make it atomic with a Lua script, or `EXPIRE key window NX` in a pipeline. Fixed windows also allow up to twice the limit across a window boundary (token bucket and sliding window fix this, Volume 4).

**More Redis facts for interviews:**

- **Single-threaded command execution** means each command is atomic, but a slow command (`KEYS *`, a huge `SORT`) blocks everyone. Use `SCAN`, not `KEYS`.
- **Data structures and uses:** strings (cache, counters, locks), hashes (small objects), lists (simple queues), sets (unique members), **sorted sets** (leaderboards, time-ordered jobs), streams (durable event logs), pub-sub (fire-and-forget messages, not durable).
- **Persistence:** **RDB** snapshots (compact, can lose recent writes) and **AOF** (append-only log, `fsync` policy trades durability and speed). Without either, a restart empties it. Treat Redis as a cache unless you configured durability on purpose, and note that a **Celery broker on Redis loses queued tasks if Redis restarts without persistence**.
- **Fail open:** your appointment system keeps working if Redis is down (cache and rate limiting are skipped). State the trade-off: availability over protection, and for rate limits that means abuse is possible while Redis is down.
- **Serialisation:** JSON is safe and portable. **Never unpickle data from a shared cache that untrusted parties can write to** (pickle executes code).
- **Key design:** namespaced and versioned (`app:v2:user:42`), short, predictable, always with a TTL.

### 5.2 Background jobs with Celery

**The idea.** A web request should return in milliseconds. Slow, unreliable or CPU-heavy work (sending email, calling an AI provider, generating a report, ML inference, calendar sync) belongs **outside the request**. The web process puts a message on a **queue (broker)** and returns immediately, and separate **worker processes** consume and run the jobs. Workers are separate **processes**, so CPU-bound jobs also escape Python's GIL (Volume 1).

```mermaid
flowchart LR
    W[Web app: FastAPI or Flask] -- task.delay(args) --> B[(Broker: Redis or RabbitMQ)]
    BT[Celery Beat: scheduler] -- periodic tasks --> B
    B --> K1[Worker 1]
    B --> K2[Worker 2]
    K1 --> D[(Database)]
    K2 --> E[External APIs]
    K1 -- result --> R[(Result backend, optional)]
```

**Configuration that matters (this app definition loaded and registered its tasks and beat entries in the verification run):**

```python
from celery import Celery

app = Celery('app', broker='redis://redis:6379/0', backend='redis://redis:6379/1')
app.conf.update(
    task_acks_late=True,                  # acknowledge AFTER the task finishes: if a worker dies mid-task it is redelivered
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,         # do not hoard tasks: fairer distribution for long jobs
    task_time_limit=60, task_soft_time_limit=45,     # hard kill and a catchable SoftTimeLimitExceeded: no job runs forever
    timezone='UTC',
    beat_schedule={
        'reclaim-expired-holds': {'task': 'tasks.reclaim_holds', 'schedule': 60.0},        # every 60 seconds
        'weekly-review': {'task': 'tasks.weekly_review', 'schedule': 7 * 24 * 3600.0},
    },
)

@app.task(name='tasks.send_reminder', bind=True,
          autoretry_for=(ConnectionError,), retry_backoff=True, retry_backoff_max=300,
          retry_jitter=True, max_retries=5)
def send_reminder(self, appointment_id):               # pass an ID, not an object: load fresh data inside the task
    ...

@app.task(name='tasks.reclaim_holds')
def reclaim_holds():
    # idempotent by construction: running it twice changes nothing the second time
    # UPDATE slots SET status = 'free', held_until = NULL WHERE status = 'held' AND held_until < now()
    ...
```

**Rules for well-behaved tasks:**

- **Make every task idempotent.** With `acks_late`, delivery is **at-least-once**: a task may run twice (a worker crashes after finishing but before acknowledging, a broker redelivers). A reminder email task needs a 'reminder already sent' marker (a unique constraint or a status column) so a duplicate does nothing.
- **Pass small, serialisable arguments** (IDs), never ORM objects or large payloads. The data may have changed by the time the task runs, so re-read it.
- **Retry transient failures with exponential backoff and jitter**, cap `max_retries`, and send permanently failing tasks somewhere you can see (dead-letter handling, alerts).
- **Set time limits** so one stuck job cannot occupy a worker forever.
- **Do not wait for a task inside another task** (blocking on results from within a worker can deadlock when all workers are waiting, the same thread-pool starvation as Volume 1).
- **Route heavy and light tasks to separate queues** (`task_routes`, `-Q email,default`) so a flood of slow jobs cannot starve urgent ones.
- **Do not store results you never read** (`ignore_result=True`), or the result backend fills up.

**Beat, the scheduler:** a separate process that enqueues tasks on a schedule. **Run exactly one Beat instance**, otherwise every scheduled task fires once per instance. Beat only *enqueues*, so a slow task can pile up if its interval is shorter than its runtime (guard long tasks with a lock or make them skip when already running). Beat's schedule lives in config by default (a database-backed scheduler is available for dynamic schedules). Your 60-second hold reclamation job and weekly ETL are exactly this pattern.

**Failure modes (a checklist for 'my jobs are not running'):**

| Symptom | Likely cause |
| --- | --- |
| Tasks enqueue but nothing happens | No worker running (a worker is a **separate service** to deploy, not part of the web process), wrong queue name, wrong broker URL |
| Scheduled tasks never fire | Beat not running, or it is running but the worker is not consuming that queue |
| Scheduled tasks fire twice | Two Beat instances |
| Tasks vanish after a Redis restart | Broker has no persistence (enable AOF), or tasks were only in memory |
| A task with a long ETA or delay runs repeatedly | Redis broker **visibility timeout** shorter than the task's delay or duration: it was redelivered. Raise the timeout or avoid very long ETAs |
| Worker memory grows | Leaks in tasks, large payloads, result accumulation. Use `worker_max_tasks_per_child` to recycle processes |
| `Task of kind ... is not registered` | The worker did not import the module defining the task |
| Pickle or serialisation errors | Non-serialisable arguments. Use JSON and plain types |

**Canvas** (composing tasks): `chain` (run in sequence, passing results), `group` (run in parallel), `chord` (parallel then a callback). Mention it, but keep real workflows simple and idempotent.

### 5.3 The transactional outbox pattern

**The problem (dual write).** An order is saved in the database and an event must be published to a broker so other services react. These are **two systems with no shared transaction**. If you commit then publish and the process crashes between them, the event is lost. If you publish then commit and the commit fails, a phantom event goes out.

**The solution.** Write the business change **and an 'event to send' row in the same database transaction** (atomic by definition). A separate **relay** reads unsent outbox rows, publishes them, and marks them sent. The guarantee is **at-least-once**, so consumers must be **idempotent**.

```python
import json, sqlite3

def setup(conn):
    conn.executescript('''
      CREATE TABLE orders (id INTEGER PRIMARY KEY, item TEXT);
      CREATE TABLE outbox (id INTEGER PRIMARY KEY AUTOINCREMENT, topic TEXT, payload TEXT, sent INTEGER DEFAULT 0);''')

def place_order(conn, item):
    with conn:                                               # ONE transaction: the order and its event commit together or not at all
        cur = conn.execute('INSERT INTO orders (item) VALUES (?)', (item,))
        conn.execute('INSERT INTO outbox (topic, payload) VALUES (?, ?)',
                     ('order_placed', json.dumps({'order_id': cur.lastrowid})))

def relay(conn, publish):
    rows = conn.execute('SELECT id, topic, payload FROM outbox WHERE sent = 0 ORDER BY id').fetchall()
    for oid, topic, payload in rows:
        publish(topic, payload, oid)                         # publish first ...
        conn.execute('UPDATE outbox SET sent = 1 WHERE id = ?', (oid,))     # ... then mark as sent
        conn.commit()

class Consumer:                                              # idempotent: the message id makes duplicates harmless
    def __init__(self): self.seen, self.effects = set(), []
    def handle(self, topic, payload, message_id):
        if message_id in self.seen: return 'duplicate ignored'
        self.effects.append(payload); self.seen.add(message_id); return 'processed'
```

The verification run made the relay **crash after delivering message 1 but before marking it sent**. The restarted relay re-sent it, and the idempotent consumer ignored the duplicate:

```text
delivery log: ['processed', 'crash: relay crashed', 'duplicate ignored', 'processed']
effects applied once each: ['{"order_id": 1}', '{"order_id": 2}']
```

**Points to make:**

- **At-least-once plus an idempotent consumer = effectively once.** Exactly-once delivery across systems is not something you can simply have.
- In production the consumer records processed message ids in a table with a **unique constraint, in the same transaction as its own effect**, so the dedupe check and the effect are atomic.
- The relay is a **polling job** (a Celery Beat task works) or **change data capture** (Debezium reading the database log). Add an index on unsent rows, delete or archive sent rows, and keep **ordering** in mind (per aggregate, by outbox id).
- This is how your appointment system gets at-least-once email delivery, and the same idea as TaskFlow's outbox. Say the guarantee, the cost (extra table, a relay, duplicates possible) and the alternative (publish after commit and accept loss).

### 5.4 Resilience: timeouts, retries, circuit breakers, fallbacks

Every network call can be slow, fail, or hang. A slow dependency is **worse than a dead one**, because threads pile up waiting and your own service falls over (cascading failure).

- **Always set timeouts** (connect and read) on every outbound call: `requests.get(url, timeout=(3, 10))`. A missing timeout is a bug. Total request budgets should shrink as you go deeper in a call chain.
- **Retry only transient errors, only idempotent (or keyed) operations, with exponential backoff and jitter, and a cap.** Retries multiply load on a struggling service (retry storms), so use a retry budget.
- **Circuit breaker:** after repeated failures, **stop calling** the dependency for a while and fail fast, then test recovery carefully. States: **closed** (normal, counting failures), **open** (reject immediately), **half-open** (let a trial call through, success closes it, failure reopens it). This is what you built around the ML client in TaskFlow. Verified run with a fake clock:

```python
import threading, time

class CircuitOpen(Exception): pass

class CircuitBreaker:
    def __init__(self, failure_threshold=3, reset_timeout=30.0, clock=time.monotonic):
        self.th, self.timeout, self.clock = failure_threshold, reset_timeout, clock
        self.failures, self.state, self.opened_at = 0, 'closed', 0.0
        self._lock = threading.Lock()

    def call(self, fn, *args, **kwargs):
        with self._lock:
            if self.state == 'open':
                if self.clock() - self.opened_at >= self.timeout:
                    self.state = 'half_open'                      # time to probe
                else:
                    raise CircuitOpen('failing fast, dependency is down')
        try:
            result = fn(*args, **kwargs)
        except CircuitOpen:
            raise
        except Exception:
            with self._lock:
                self.failures += 1
                if self.state == 'half_open' or self.failures >= self.th:
                    self.state, self.opened_at = 'open', self.clock()
            raise
        with self._lock:
            self.failures, self.state = 0, 'closed'
        return result
```

```text
breaker: ['fail', 'fail', 'fail', 'open', 'fast-fail', 'prediction', 'closed']
# three failures open it, the next call fails fast, after the timeout one trial succeeds and closes it
```

- **Fallbacks and graceful degradation:** return a cached or default value, or a cheaper alternative, when a dependency is unavailable. Your appointment system uses a **deterministic keyword heuristic when the LLM call times out**, and TaskFlow falls back from WebSockets to HTTP polling. Always decide what the user experiences during a failure.
- **Bulkheads:** separate pools or queues per dependency, so one failing dependency cannot consume every thread or connection.
- **Load shedding and rate limiting:** reject excess work early and cheaply (HTTP 429 or 503 with `Retry-After`) instead of collapsing.
- **Idempotency keys** make retries safe for state-changing calls.

### Loopholes in this part

- Caching data that must be correct (balances, stock counts) without a clear invalidation story.
- Caching per-user data under a shared key (leaks one user's data to another). Include the user or tenant in the key.
- Assuming Celery runs a task exactly once.
- Forgetting that Beat is a single point (and must be a single instance).
- Retrying non-idempotent POSTs, retry loops without a cap or jitter, no timeouts.
- Treating the outbox as exactly-once. It is at-least-once.

### Interview questions on caching, jobs and resilience

- **Explain cache-aside and its invalidation problem.** Read through the cache, load on miss, delete on write, TTL as a safety net, the stale refill race.
- **What is a cache stampede and how do you prevent it?** A hot key expiring under load. Rebuild lock with `SET NX PX`, jittered TTLs, stale-while-revalidate. We measured 20 database loads fall to 1.
- **Why use a task queue? What guarantees does Celery give?** Move slow work off the request and escape the GIL. At-least-once delivery, so tasks must be idempotent.
- **What happens if a worker dies mid-task?** With `acks_late` the task is redelivered, otherwise it may be lost. So make it idempotent.
- **Why run only one Beat?** Each instance schedules every task, so duplicates fire.
- **What is the outbox pattern and what does it guarantee?** Atomic business change plus event row, a relay publishes. At-least-once, idempotent consumers.
- **Explain the circuit breaker.** Closed, open, half-open, fail fast to protect resources and give the dependency time to recover.
- **What do you do when the dependency you call is slow?** Timeouts, circuit breaker, fallback, bulkhead, and load shedding.

### Revise in 30 seconds: caching, jobs, resilience

- Cache-aside: read cache, load on miss, set with TTL, delete on write. Failure modes: stampede (rebuild lock, 20 loads became 1), avalanche (TTL jitter), penetration (null caching), hot key, stale refill race. Always TTL, namespaced keys, never share per-user data under one key.
- Redis commands are atomic. `SET NX PX` for locks (with a token), `INCR` plus `EXPIRE` is racy without a script. Persistence is optional: RDB and AOF. Fail open is a deliberate trade-off.
- Celery: broker, workers, Beat. At-least-once, so idempotent tasks, IDs as arguments, backoff retries, time limits, `acks_late`, separate queues, exactly one Beat.
- Outbox: same transaction writes data and event, relay publishes at-least-once, consumers dedupe by message id.
- Resilience: timeouts everywhere, retry with backoff and jitter only for idempotent calls, circuit breaker (closed, open, half-open), fallbacks, bulkheads, load shedding.

## Part 6: Observability

### 6.1 Why it exists and the three signals

**The idea.** In production you cannot attach a debugger. **Observability** is how well you can understand what your system is doing from the outside, using the data it emits. Monitoring tells you **that** something is wrong, observability helps you find **why**.

| Signal | What it answers | Example |
| --- | --- | --- |
| **Logs** | What happened, in detail, for one event | 'slot 7 hold failed: already booked, request abc123' |
| **Metrics** | How much, how often, how fast, as numbers over time | requests per second, error rate, p95 latency, queue depth |
| **Traces** | Where did the time go across services for one request | a request spent 40 ms in the API, 300 ms waiting on the AI provider, 5 ms in Postgres |

**What to measure, the standard frameworks:**

- **The four golden signals (Google SRE):** **latency** (time per request, separating successes from failures), **traffic** (demand), **errors** (rate of failures), **saturation** (how full the most constrained resource is).
- **RED** for services: **R**ate, **E**rrors, **D**uration. **USE** for resources (CPU, memory, disk, connection pool): **U**tilisation, **S**aturation, **E**rrors.
- **Use percentiles, not averages.** An average hides the slow tail. A p50 of 80 ms with a p99 of 4 s means one user in a hundred waits four seconds. Always chart p50, p95 and p99.

### 6.2 Structured logging, with request correlation

Plain text logs are hard to search. **Structured logs** (one JSON object per line) can be filtered by field. Every line should carry a **request id**, so you can pull up everything one request did across modules, and across services if you pass the id along in a header.

Verified run with FastAPI middleware, `contextvars` (safe across async tasks), and a JSON formatter. The output for a request that carried `x-request-id: abc123`:

```text
{"level": "INFO", "logger": "app", "msg": "request done", "request_id": "abc123", "route": "/tasks/{task_id}", "status": 200}
```

```python
import contextvars, json, logging, time, uuid
from fastapi import FastAPI, Request

request_id = contextvars.ContextVar('request_id', default='-')

class JsonFormatter(logging.Formatter):
    def format(self, record):
        payload = {'level': record.levelname, 'logger': record.name, 'msg': record.getMessage(),
                   'request_id': request_id.get()}                    # every line automatically carries the id
        payload.update(getattr(record, 'extra_fields', {}))
        return json.dumps(payload)

handler = logging.StreamHandler(); handler.setFormatter(JsonFormatter())
log = logging.getLogger('app'); log.addHandler(handler); log.setLevel(logging.INFO); log.propagate = False

app = FastAPI()

@app.middleware('http')
async def observe(request: Request, call_next):
    token = request_id.set(request.headers.get('x-request-id') or uuid.uuid4().hex[:8])   # accept or create an id
    status = 500
    try:
        response = await call_next(request)
        status = response.status_code
        response.headers['x-request-id'] = request_id.get()           # hand it back so clients can quote it in bug reports
        return response
    finally:
        route = request.scope['route'].path if request.scope.get('route') else 'unmatched'
        log.info('request done', extra={'extra_fields': {'route': route, 'status': status}})
        request_id.reset(token)
```

**Logging rules:**

- **Levels:** DEBUG (developer detail, off in production), INFO (normal events), WARNING (unexpected but handled), ERROR (a failure of this operation), CRITICAL (the service is in trouble). Do not log expected situations (a user typing a wrong password) at ERROR, or alerts become noise.
- **Never log secrets or personal data:** passwords, tokens, full card numbers, health details, and avoid whole request bodies. Mask or omit them. This matters especially in healthcare-style data.
- Log to **stdout**, let the platform ship it. Include enough context (ids, route, duration) but not whole objects. **Log once** at the boundary where the error is handled, not at every layer (duplicate stack traces drown the signal).
- Use **sampling** for very high-volume debug logs, and keep **retention** limits.
- For exceptions, log the **stack trace** (`log.exception(...)`) and send them to an **error tracker** (Sentry) that groups and alerts.

### 6.3 Metrics

Three metric types in Prometheus-style systems:

- **Counter:** only goes up (`http_requests_total`). You chart its **rate**.
- **Gauge:** goes up and down (queue depth, active connections, memory).
- **Histogram:** counts observations in buckets, from which you compute **percentiles** (request duration).

```python
from prometheus_client import Counter, Histogram

REQS = Counter('http_requests_total', 'Requests', ['method', 'route', 'status'])
LAT = Histogram('http_request_duration_seconds', 'Latency', ['route'],
                buckets=(0.01, 0.05, 0.1, 0.5, 1, 5))
# in the middleware: REQS.labels(method, route, str(status)).inc();  LAT.labels(route).observe(seconds)
```

The verified run produced these series (note the route is the **template**, not the raw URL):

```text
http_requests_total{method="GET",route="/health",status="200"} 1.0
http_requests_total{method="GET",route="/tasks/{task_id}",status="200"} 2.0
http_requests_total{method="GET",route="unmatched",status="404"} 1.0
```

**Cardinality warning (a classic mistake):** every distinct combination of label values is a separate time series stored in memory. Labelling by the **raw path** (`/tasks/7`, `/tasks/8`, ...) or by user id creates **millions of series** and can take down the monitoring system. Label by the **route template** and a small fixed set of values, and send high-cardinality detail to logs and traces instead. That is why unknown paths are grouped as `unmatched`.

**Business and system metrics worth adding:** queue depth and oldest message age, cache hit ratio, database connection pool usage, slot hold expirations per minute, failed bookings by reason, external API latency and error rate, and job retries.

### 6.4 Tracing

A **trace** follows one request through every service and component. It is a tree of **spans** (one operation each: an HTTP handler, a database query, an outbound call) sharing a **trace id**, each with start time and duration. The trace id travels between services in a header (the W3C `traceparent` header), so spans from different services join up. **OpenTelemetry** is the standard for instrumenting code (auto-instrumentation exists for FastAPI, Flask, SQLAlchemy, Redis, requests) and exporting to a backend (Jaeger, Tempo, a vendor). Tracing is what tells you **which hop is slow** in a chain of services, and it shows N+1 query patterns as hundreds of tiny database spans under one request.

### 6.5 Health checks: liveness, readiness, startup

| Probe | Question | Failure action | What it should check |
| --- | --- | --- | --- |
| **Liveness** `/health` | Is the process alive and not stuck? | **Restart** it | Almost nothing: the app responds. Do **not** check the database here, or a database outage makes the platform restart every healthy instance |
| **Readiness** `/ready` | Can it serve traffic right now? | **Stop sending traffic** (do not restart) | Dependencies it needs: database, Redis, migrations applied |
| **Startup** | Has it finished booting? | Keep waiting, then fail | Slow initialisation (loading models, warming caches) |

Verified: both endpoints returned 200 in the test run, with `/ready` reporting the dependency checks it performed. Keep checks fast (a timeout of about a second), cheap, and without side effects.

### 6.6 SLIs, SLOs, alerts and incidents

- **SLI (indicator):** a measurement ('percentage of booking requests that succeed in under 500 ms'). **SLO (objective):** the target ('99.5 percent over 30 days'). **SLA (agreement):** a contractual promise with penalties, usually looser than the SLO. The **error budget** is the allowed failure (0.5 percent), and it tells you when to slow down features and fix reliability.
- **Availability arithmetic:** 99.9 percent allows about 8.8 hours of downtime per year (43 minutes per month), 99.99 percent about 53 minutes per year.
- **Alert on symptoms users feel** (error rate, latency, failed bookings), not on every cause (CPU 80 percent). Every alert must be **actionable and urgent**, link to a **runbook**, and have an owner. Alert fatigue means real alerts get ignored.
- **Incident flow:** detect, acknowledge, **mitigate first** (rollback, disable a flag, scale up, fail over) before finding the root cause, communicate, resolve, then a **blameless postmortem** (timeline, impact, root cause, what worked, action items with owners).
- **Debugging order in production:** start at the **metrics** dashboard (what is broken, since when, how bad), check **recent deploys and config changes** (most incidents follow a change), use **traces** to find the slow or failing hop, and read **logs** (filtered by request id) for the detail.

### Interview questions on observability

- **Logs vs metrics vs traces?** Detail about one event, aggregated numbers over time, the path and timing of one request across services.
- **What are the golden signals?** Latency, traffic, errors, saturation.
- **Why percentiles over averages?** Averages hide tail latency.
- **How do you debug a slow request in production?** Metrics to scope it, a trace to find the slow hop, logs by request id for detail, and `EXPLAIN ANALYZE` if it is a query.
- **Liveness vs readiness?** Restart versus stop routing. Why must liveness not check the database?
- **What is label cardinality and why does it matter?** Number of distinct label combinations, each a stored series. High cardinality overwhelms the metrics system.
- **What would you alert on for a booking service?** Booking error rate, p95 latency, queue depth and age, hold-reclaim job not running, database connection saturation.
- **What goes in a postmortem?** Timeline, impact, root cause, contributing factors, what went well, action items. Blameless.

### Revise in 30 seconds: observability

- Logs (detail), metrics (numbers over time), traces (request path). Golden signals: latency, traffic, errors, saturation. RED for services, USE for resources. Use p95 and p99.
- Structured JSON logs with a request id (contextvars), stdout, no secrets or personal data, log at the boundary.
- Counter, gauge, histogram. Label by route template, never raw path or user id (cardinality).
- Traces: spans with a shared trace id (`traceparent`), OpenTelemetry. Liveness (restart, no dependency checks) versus readiness (route traffic, check dependencies).
- SLI, SLO, SLA, error budget. Alert on symptoms with runbooks. Mitigate first, then root cause, blameless postmortem.

## Part 7: Cloud deployment, performance and troubleshooting

### 7.1 Platforms you have used, and what to say about them

The details of pricing and limits change often, so confirm current behaviour in each platform's documentation before an interview. The concepts below are stable.

| Platform | Model | Good for | Things to know |
| --- | --- | --- | --- |
| **Railway, Render** | Git-connected platform as a service: build from a Dockerfile or buildpack, run as a service, managed Postgres and Redis add-ons | Quick full-stack deployments (web, worker, beat as separate services), environment variables per service | Free or low tiers may **sleep idle services** (cold start on the first request), set a start command, bind to the **`$PORT`** the platform provides |
| **fly.io** | Containers run as microVMs near users, configured by `fly.toml`, driven by the `fly` CLI | Multi-region apps, long-running processes, volumes | You choose regions and machine sizes, health checks and release commands live in `fly.toml` |
| **Vercel + Neon** | Vercel runs frontends and **serverless/edge functions**, Neon is **serverless PostgreSQL** that scales and can pause when idle | Next.js apps like your InventoryPro | Functions are short-lived and many instances open connections, so use **connection pooling** (Prisma with Neon's pooler). Cold starts add latency, long-running work and background workers do not fit, use queues or a separate service |
| **VM or Kubernetes** | You manage machines or a cluster | Full control, special needs | More to operate: patching, scaling, networking, TLS |

**What a platform does for you:** builds your image, injects environment variables, provides TLS and a domain, load balances, restarts crashed instances, and (with health checks) does rolling deploys. **What you still own:** the app's correctness under restarts, migrations, secrets, backups, monitoring, and cost.

### 7.2 Process model: web workers, worker services, Beat

- Production Python web apps run under a **process manager and server**, not the framework's development server. For FastAPI: **gunicorn with uvicorn workers** (the Dockerfile in Part 2). For Flask: gunicorn.
- **Number of workers:** each worker is a process with its own memory. A common starting point is `2 x CPU cores + 1` for synchronous workers, and fewer for async workers (they handle many connections each). Measure, do not guess. More workers means more memory and more database connections.
- Useful gunicorn options: `--timeout` (kill a worker stuck longer than this), `--graceful-timeout` (time allowed to finish in-flight requests on shutdown), `--max-requests` with `--max-requests-jitter` (recycle workers to contain slow leaks), `--bind 0.0.0.0:$PORT`.
- **Deploy web, worker and beat as separate services** (or separate processes with separate scaling) from the same image. A common mistake is deploying only the web service, so queued tasks never run.
- **Release phase:** run migrations as a one-off step **before** traffic shifts to the new version (platform 'release command' or a pre-deploy job).

### 7.3 Graceful shutdown (what happens on every deploy)

When a platform stops an instance it sends **SIGTERM**, waits a grace period (often 10 to 30 seconds), then **SIGKILL**s it. A well-behaved process **stops accepting new work, finishes in-flight work, releases resources, and exits with code 0**. A process that ignores SIGTERM loses in-flight requests and half-finished jobs on every deploy.

Verified run: the process was sent SIGTERM while it was working, finished its current job, and exited cleanly.

```python
import signal, sys, threading, time

stop = threading.Event()
def on_term(signum, frame):
    stop.set()                                   # only set a flag, do the real work in the main loop
signal.signal(signal.SIGTERM, on_term)
signal.signal(signal.SIGINT, on_term)

while not stop.is_set():
    process_one_job()                            # finish the CURRENT job before looking at the flag again
cleanup()                                        # close connections, flush logs, release locks
sys.exit(0)
```

```text
worker started
finished current job, completed 4, exiting cleanly | exit code 0
```

- **gunicorn and uvicorn handle SIGTERM for you** (they stop accepting connections and drain), provided your container runs them as PID 1 (the **exec form** of `CMD`, Part 2). Set the platform's grace period longer than your slowest normal request.
- **Celery** performs a **warm shutdown** on SIGTERM: it stops taking new tasks and waits for running ones. Combined with `acks_late`, a task killed mid-way is redelivered, which is another reason tasks must be idempotent.
- **Load balancer draining:** the instance should become **not ready first** (fail readiness), so the balancer stops sending it new requests, then finish what is in flight.
- On rolling deploys, **the old and new versions run side by side**, so the code and schema must be backward compatible (Part 3).

### 7.4 Performance work: measure first

**The rule: do not optimise by guessing. Measure, find the bottleneck, fix it, measure again.**

1. **Define the target** (p95 under 300 ms at 200 requests per second).
2. **Reproduce with a load test:** Locust (Python), k6 or wrk. Ramp users, watch p95 and error rate, and find the saturation point.
3. **Find where time goes:** a trace or per-phase timing (`curl -w`), the **slow query log** and `EXPLAIN ANALYZE` (Volume 2), and a **profiler** for CPU-bound code. `cProfile` ships with Python:

```python
import cProfile, pstats, io

pr = cProfile.Profile(); pr.enable()
handler()                                        # the code you suspect
pr.disable()
stats = pstats.Stats(pr, stream=io.StringIO()).sort_stats('cumulative')
stats.print_stats(10)                            # top 10 by cumulative time
```

(Verified output on a toy handler: `handler` 0.189 s cumulative, calling `slow` 50 times at 0.004 s each, which shows where the time is.) For a running production process use a sampling profiler such as **`py-spy`**, which attaches without restarting (`py-spy top --pid <PID>`, `py-spy dump --pid <PID>` to see where every thread is stuck, ideal for hangs and deadlocks).

4. **Fix the biggest cost first.** Typical wins, in order of how often they matter:
   - **N+1 queries and missing indexes** (Volume 2): by far the most common backend slowness.
   - **Caching** hot reads (Part 5), HTTP caching and a CDN for static assets.
   - **Move slow work to background jobs** and answer with `202`.
   - **Pagination and field selection**, smaller payloads, compression.
   - **Connection pooling** and keep-alive for outbound calls.
   - Batch operations (one query for many rows) instead of loops.
5. **Async pitfalls:** in an `async def` FastAPI handler, a **blocking call** (a synchronous database driver, `requests`, `time.sleep`, heavy CPU work) **blocks the whole event loop** and stalls every other request on that worker. Use async libraries, or declare the handler as a plain `def` (FastAPI runs it in a thread pool), or push CPU work to a worker process.

**Connection pool arithmetic (a very common production outage):**

```text
PostgreSQL max_connections = 100
instances (3)  x  gunicorn workers per instance (4)  x  pool size per worker (10)  =  120 possible connections
120 > 100  ->  'too many connections' errors under load
Fixes: smaller pools (pool_size 5 gives 60), fewer workers, or a pooler such as PgBouncer in front of PostgreSQL.
Remember Celery workers and Beat also hold connections, and serverless platforms (Vercel + Neon) multiply connections, which is why a pooler is used there.
```

### 7.5 Security hardening checklist for a deployed backend

- HTTPS everywhere (HSTS), secure cookie flags, correct CORS (Volume 3).
- Authentication on every route that needs it, **object-level authorisation** (no IDOR), RBAC enforced on the server.
- Input validation with schemas, parameterised queries, output encoding.
- Rate limiting and brute-force protection on login and expensive endpoints.
- Secrets from the secret store, rotated, least-privilege database users, no secrets in logs or images.
- Dependency scanning and updates, minimal images, non-root containers.
- Backups with **tested restores**, encrypted sensitive data, audit logs for sensitive actions.
- Do not expose debug mode, stack traces or the database port publicly. Remove default credentials.

### 7.6 Troubleshooting runbook: symptom, check, fix

| Symptom | First checks | Typical fix |
| --- | --- | --- |
| **Build fails on the platform** | Build logs: missing dependency, wrong Python version, Dockerfile path, a file excluded by `.dockerignore` | Pin versions, fix the Dockerfile, build the same image locally |
| **Service crash-loops** | Logs from the first crash, exit code (137 OOM, 1 app error), missing environment variable | Fix config, raise memory, fix the startup error (fail-fast config errors are visible in logs) |
| **Deploy succeeded but the site shows 502 or is unreachable** | Is the app listening on **`0.0.0.0`** and the **`$PORT`** the platform expects? Health check path and timeout | Bind correctly, fix the start command, extend the health check grace period |
| **Works locally, fails deployed** | Environment variables, database URL, region, file paths, case-sensitive file names, read-only filesystem | Compare configs, use the same container image locally |
| **`connection refused` or 'too many connections' to the database** | Database URL host, network and firewall rules, `max_connections` versus pool arithmetic, database paused (serverless) | Correct the URL, shrink pools, add PgBouncer, warm the database |
| **Migration failed or app errors about missing columns** | Was the migration run before the new code received traffic? Is the app pointing at the right database? | Run the release step, expand-contract for compatibility |
| **Tasks queued but not processed** | Is a **worker service** deployed and running? Same broker URL? Same queue names? Queue depth | Start the worker, fix the URL, check Redis persistence and memory |
| **Scheduled jobs missing or doubled** | Beat running? Exactly one instance? Timezone? Worker consuming that queue? | Run a single Beat, fix the schedule and queues |
| **First request after idle is very slow** | Platform sleeping idle services, serverless cold start, a paused serverless database | Wake it beforehand, keep-alive pings if appropriate, or a paid always-on tier |
| **Memory grows until restart** | Unbounded caches, large payloads in memory, leaked connections, Celery result accumulation | Bound caches, stream large data, recycle workers (`--max-requests`, `worker_max_tasks_per_child`) |
| **Random 502s or resets during deploys** | Missing graceful shutdown, keep-alive timeout mismatch with the load balancer, instances killed mid-request | Handle SIGTERM, longer grace period, align timeouts |
| **CORS error in the browser, but `curl` works** | The browser preflight, allowed origin, credentials setting | Fix CORS config and middleware order (Volume 3) |
| **Wrong times or timestamps** | Server timezone, naive datetimes, daylight saving | Store UTC with `timestamptz`, convert at the edge |
| **Disk full or too many open files** | Logs written to files, temp files, leaked file handles or sockets (many `CLOSE_WAIT`) | Log to stdout, clean temp files, close resources, raise limits |
| **Calendar or OAuth integration stops working** | Expired or revoked refresh token, redirect URI mismatch, secret rotated | Re-authorise, handle revocation by prompting the user, check stored encryption key |

**Pre-interview checklist for your deployed projects:** open every live link on the day of the interview and again an hour before, wake sleeping free-tier services, make sure a **demo account and seeded data** exist, test the main flow (sign up or log in, the core action, the background job result), keep the README up to date with architecture diagram, setup steps and env variable list, have logs and the dashboard ready to show, and prepare a one-minute story for the hardest bug you fixed. A dead link or a cold-start timeout in front of an interviewer is a self-inflicted wound.

### Interview questions on deployment and operations

- **Walk me through how you deployed your project.** Source in Git, CI builds and tests, image tagged by commit, platform runs web, worker and beat services with environment variables, migrations as a release step, health checks, rolling deploy, rollback by redeploying the previous tag.
- **What happens to in-flight requests when you deploy?** SIGTERM, drain and finish within the grace period, readiness fails first. Without graceful shutdown they are dropped.
- **Your service is slow. What do you do?** Reproduce, measure (traces, slow query log, profiler), fix the largest bottleneck (usually queries and N+1), verify with a load test.
- **How do you size connection pools?** Instances times workers times pool size must stay under the database `max_connections`, leave room for other clients, use a pooler for serverless.
- **How would you handle a spike in traffic?** Autoscale stateless web instances, cache, queue heavy work, rate limit and shed load, protect the database with pooling.
- **A scheduled job ran twice. Why?** Two Beat instances or an at-least-once redelivery. Fix with a single Beat and idempotent tasks.
- **How do you roll back a bad release?** Redeploy the previous image, keep migrations backward compatible, use feature flags for an instant off-switch.
- **What do you monitor after a deploy?** Error rate, latency percentiles, saturation, queue depth, and the business metric (successful bookings), compared with the baseline.

### Revise in 30 seconds: deployment and operations

- Platform runs the image, you own correctness: web, worker and beat as separate services, bind `0.0.0.0:$PORT`, migrations as a release step, secrets as variables, health checks, rollback by previous tag. Free tiers sleep, serverless cold-starts.
- gunicorn plus uvicorn workers, `--graceful-timeout`, recycle workers. SIGTERM then SIGKILL: stop taking work, finish in-flight work, exit 0. Exec-form `CMD`.
- Performance: measure first (load test, trace, slow query log, `cProfile`, `py-spy`), fix N+1 and indexes, cache, queue, paginate. Never block the event loop in async handlers.
- Pool arithmetic: instances times workers times pool size under `max_connections`, otherwise use a pooler. Many `CLOSE_WAIT` means a leaked connection.
- Runbook order: logs, exit code, environment variables, port and bind address, database connectivity, worker and beat running, recent deploy or config change. Check every live link before an interview.

## What Volume 5 covers, and what comes next

Volume 5 covers the backend, DevOps and deployment items from your checklist in depth: Git internals and workflow (with a real verified session), Docker and Compose for a FastAPI, Celery, Redis and PostgreSQL stack, CI/CD with GitHub Actions, deployment strategies and zero-downtime migrations, configuration and secrets, testing including a real concurrency test, Redis caching and its failure modes, Celery, the outbox pattern, resilience patterns, observability, graceful shutdown, performance work, and a troubleshooting runbook.

Next in the series is **Volume 6: system design (HLD)**: scaling fundamentals, load balancing, caching and CDNs, database replication and sharding, consistency and CAP, queues and Kafka, rate limiting, ID generation, and the ten HLD systems (URL shortener, rate limiter, notification service, chat, file storage, news feed, e-commerce, job scheduler, appointment booking, distributed cache), followed by ML and GenAI, DSA patterns, and the systems and programming volume.