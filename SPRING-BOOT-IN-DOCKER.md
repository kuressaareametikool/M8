# Spring Boot track — everything in Docker

The Spring Boot lessons assume a JDK on your computer (`./mvnw`, IntelliJ's ▶ button). On a school PC
you may not be allowed to install a JDK, or Windows may refuse to run it. Docker is installed, so we
run **Java and Maven in a container** too. You only need Docker Desktop, Git and an editor.

```
your PC                                    Docker
───────                                    ──────
editor ── edits files in ./billable ──►    app       (Maven + JDK 25, runs the application)
browser ── http://localhost:8080 ─────►      │
                                             ▼
                                           postgres  (PostgreSQL 17, from lesson 01)
                                           mailpit   (from lesson 09)
```

Your project folder is shared with the `app` container, so the code you edit is the code that runs.
Your code is the same as everyone else's; only the commands that start it change.

---

## 1. Create the project (lesson 01, Step 1)

Generate the project on [start.spring.io](https://start.spring.io) with lesson 01's settings, download
the zip and extract it. Skip the `java -version` / `./mvnw -v` checks — you have no local Java; the
container brings its own.

## 2. Add the `app` service to `compose.yaml` (lesson 01, Step 3)

Write `compose.yaml` as lesson 01 shows, then add the `app` service and the `maven-repo` volume:

```yaml
# compose.yaml
services:
  postgres:
    # ... unchanged from lesson 01 ...

  # The application itself, for computers without a JDK (SPRING-BOOT-IN-DOCKER.md).
  # Only starts with the "app" profile, so nobody else is affected.
  app:
    image: maven:3.9-eclipse-temurin-25
    profiles: ["app"]
    working_dir: /workspace
    command: mvn -B spring-boot:run
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/billable
      SPRING_DOCKER_COMPOSE_ENABLED: "false"
      SPRING_MAIL_HOST: mailpit
      TESTCONTAINERS_HOST_OVERRIDE: host.docker.internal
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "8080:8080"
    volumes:
      - .:/workspace
      - maven-repo:/root/.m2
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  pgdata:
  maven-repo:
```

What each part does:

1. `image: maven:3.9-eclipse-temurin-25` — an official image with JDK 25 and Maven. We call `mvn`, not
   `./mvnw`: same result, and it avoids Windows line-ending problems with the `mvnw` script.
2. `profiles: ["app"]` — the service starts only when the `app` profile is on. Classmates with a JDK,
   Spring's own Compose support (lesson 01) and CI never start it.
3. `SPRING_PROFILES_ACTIVE: dev` — replaces `-Dspring-boot.run.profiles=dev` (lesson 03).
4. `SPRING_DATASOURCE_URL` — inside Docker the database is reached by its service name, `postgres`,
   not `localhost`. The environment variable overrides `application.properties`.
5. `SPRING_DOCKER_COMPOSE_ENABLED: "false"` — there is no `docker` command in the container, and the
   database is already running, so Spring must not try to start `compose.yaml` itself.
6. `SPRING_MAIL_HOST: mailpit` — the same idea for lesson 09's Mailpit. Harmless before lesson 09.
7. `maven-repo` volume — downloaded dependencies survive restarts. The first start downloads them all
   and takes a few minutes; later starts take seconds.
8. The `docker.sock` mount, `TESTCONTAINERS_HOST_OVERRIDE` and `extra_hosts` — let Testcontainers
   (lesson 01 tests, lesson 14) start its test database from **inside** the container. See section 4.

**Lesson 09 re-lists `compose.yaml`** to add Mailpit. Add only the `mailpit` service; keep your `app`
service and the `maven-repo` volume.

## 3. Turn the profile on

Once per terminal window:

```powershell
$env:COMPOSE_PROFILES = "app"      # PowerShell
```

```bash
export COMPOSE_PROFILES=app        # Git Bash, macOS, Linux
```

Now every `docker compose` command in the lessons includes the `app` service: `docker compose up -d`
starts the database **and** the application, `docker compose down -v` stops both.

## 4. Replace the `./mvnw` commands

| Lesson says | In Docker |
|---|---|
| `./mvnw spring-boot:run` (with or without `-Dspring-boot.run.profiles=dev`), or IntelliJ ▶ | `docker compose up -d`, then `docker compose logs -f app` to watch the log (Ctrl+C stops watching, not the app) |
| Stop the app (Ctrl+C) | `docker compose stop app` |
| Restart after a code change | `docker compose restart app` (Maven recompiles; about 10–20 s) |
| `./mvnw test` | `docker compose run --rm --no-deps app mvn -B test` |
| `./mvnw verify`, `./mvnw spotless:apply`, any other `./mvnw <goal>` | `docker compose run --rm --no-deps app mvn -B <goal>` |
| `./mvnw test -Dtest=MoneyTest` | `docker compose run --rm --no-deps app mvn -B test -Dtest=MoneyTest` |
| `java -version` | `docker compose run --rm --no-deps app java -version` |
| `docker compose exec postgres psql ...` | unchanged |
| Lesson 15 fresh-clone check and README *Quick start* | the same replacements for the `./mvnw` lines |

`run --rm` starts a fresh container for one command and removes it afterwards. `--no-deps` skips the
database: tests bring their own through Testcontainers. Running tests while the app is up is fine.

**Shorter commands (optional).** Define a helper once per terminal:

```powershell
function mvnd { docker compose run --rm --no-deps app mvn -B @args }    # PowerShell
```

```bash
mvnd() { docker compose run --rm --no-deps app mvn -B "$@"; }           # Git Bash
```

Then `./mvnw test` becomes `mvnd test`, `./mvnw verify` becomes `mvnd verify`, and so on.

**How Testcontainers works from inside a container.** The `docker.sock` mount lets the tests talk to
Docker Desktop, which starts the test PostgreSQL **next to** the `app` container (not inside it). The
container's port is published on your PC, and `TESTCONTAINERS_HOST_OVERRIDE=host.docker.internal`
tells the tests to connect through the PC instead of `localhost` (which, inside a container, is the
container itself).

## 5. Editor

Any editor works — the code runs in the container, not in the editor. Without a JDK, VS Code's or
IntelliJ's Java support may show red underlines, but compiling and running are done by Maven in the
container, so trust the `mvn` output.

**Full Java support in VS Code without a local JDK (optional).** Install the *Dev Containers*
extension, then add `.devcontainer/devcontainer.json`:

```json
{
  "name": "Billable",
  "dockerComposeFile": ["../compose.yaml"],
  "service": "app",
  "runServices": ["postgres", "app"],
  "workspaceFolder": "/workspace",
  "customizations": {
    "vscode": {
      "extensions": ["vscjava.vscode-java-pack", "vmware.vscode-boot-dev-pack"]
    }
  }
}
```

**F1 → Dev Containers: Reopen in Container.** VS Code now runs inside the `app` container, uses its
JDK 25, and you can type the lesson's commands with plain `mvn` (instead of `./mvnw`) in its terminal.
`docker compose ...` commands still go in a terminal on your PC, outside VS Code.

---

## Troubleshooting

| Error or symptom | Cause | Fix |
|---|---|---|
| `no such service: app` | The profile is not on in this terminal | `$env:COMPOSE_PROFILES = "app"` (or `export ...`) |
| `docker compose up -d` starts only `postgres` | Same | Same |
| App log: `Connection to localhost:5432 refused` | `SPRING_DATASOURCE_URL` missing in the `app` service | Copy the `environment` block from section 2 |
| App log mentions `DockerComposeLifecycleManager` / `docker: not found` | `SPRING_DOCKER_COMPOSE_ENABLED` missing | Add `SPRING_DOCKER_COMPOSE_ENABLED: "false"` |
| `Could not find a valid Docker environment` in tests | The `docker.sock` volume is missing, or Docker Desktop is not running | Check the `volumes` of `app`; start Docker Desktop |
| Tests hang or fail with `Connection refused` to a random port | Testcontainers connects to `localhost` inside the container | Check `TESTCONTAINERS_HOST_OVERRIDE` and `extra_hosts` |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Another app (or a second `app` container) uses 8080 | `docker compose ps`, stop the other one |
| First start takes minutes, log shows `Downloading from central` | Maven fills the `maven-repo` volume | Wait; it happens once |
| A code change is not visible | The app was not restarted | `docker compose restart app` |
| `target/` files owned by `root` (Linux only) | The container runs as root | `sudo chown -R $USER target`, or add `user: "${UID}:${GID}"` to `app` |

---

## Hand-in

Commit the `app` service with the rest of `compose.yaml` — the `app` profile keeps it out of the way
of everyone who runs the project with a local JDK, and CI does not use it. Write in your README's
*Quick start* that the project also runs with `COMPOSE_PROFILES=app docker compose up -d`.
