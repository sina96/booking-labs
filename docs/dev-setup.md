# Dev setup: IntelliJ (Java/Kotlin) + Zed & terminal (Go, Docker, scripts)

## Repo layout

One **evolving system**, not a code copy per lab. IntelliJ handles one multi-project Gradle build well; nine copies of the same project would be a mess to index and run. Each finished lab is a **git tag** instead.

```
booking-labs/
├─ settings.gradle.kts        # includes ONLY the JVM services
├─ build.gradle.kts
├─ gradle/libs.versions.toml  # version catalog: Boot, Kotlin, Testcontainers… in one place
├─ services/
│  ├─ booking/       Java 25           → IntelliJ
│  ├─ payment/       Java 25           → IntelliJ
│  ├─ inventory/     Kotlin            → IntelliJ
│  └─ notification/  Go, own go.mod    → Zed   (not part of the Gradle build)
├─ compose.yaml                         → Zed
├─ infra/            MiniStack init scripts, OTel/Toxiproxy config  → Zed
├─ load/             k6 scripts                                      → Zed
├─ http/             *.http request files (IntelliJ HTTP client)
├─ labs/lab-00/NOTES.md, labs/lab-01/NOTES.md, …
├─ docs/             labs.md · events.md · dev-setup.md
├─ Makefile          shared commands: up, down, test, fmt, load
└─ .editorconfig
```

- **A lab is done** → `git tag lab-01-done`. The judge reviews `git diff lab-00-done..lab-01-done`. You get a clean history showing how the system grew, which also looks good on GitHub.
- **Everything works from the terminal** (`make`, `./gradlew`, `go test`). The IDEs are a convenience, not a requirement, so the judge agent and CI can run the same commands.

## IntelliJ IDEA (2026.2, Java + Kotlin)

- **Open the repo root** as a Gradle project, not a single service folder.
- **JDK 25 everywhere.** Your default `java` is 17. Use a Gradle **toolchain** (`languageVersion = 25`) so the CLI and the IDE agree, and set *Settings → Build Tools → Gradle → Gradle JVM* to 25 as well.
- `services/notification` (Go) shows up as plain files. Right-click → *Mark Directory as → Excluded* to keep search and indexing clean.
- Useful built-ins: **HTTP Client** (`http/*.http`, commit these), running Testcontainers tests from the gutter, the Kotlin K2 mode. Docker/compose is handled in Zed + terminal, not IntelliJ.
- Gitignore `.idea/`. Shared settings go in `.editorconfig` and Gradle.

## Zed (Go, compose, scripts, docs)

- **Open the repo root** too, so compose, infra, Go and docs are in one window. gopls finds `services/notification/go.mod` by itself. Add a `go.work` at the root only if you ever have more than one Go module.
- Install the Dockerfile / Docker Compose / Makefile / TOML extensions from the extensions panel.
- Project settings in `.zed/settings.json` (e.g. format on save with goimports) are fine to commit.

## Formatting shouldn't depend on the editor

Two editors means two formatters unless the *build* owns formatting:
- Java/Kotlin: **Spotless** in Gradle (e.g. palantir-java-format or google-java-format, and ktfmt/ktlint for Kotlin). `./gradlew spotlessApply` before committing. Make IntelliJ match (install the matching formatter plugin) or just run Spotless.
- Go: `gofmt`/`goimports`, which is standard.
- `.editorconfig` covers indentation/newlines for YAML, Markdown and scripts. Both editors respect it.

## Creating services with IntelliJ's New Project wizard

The Spring Boot wizard (Initializr) generates a **standalone** project: its own `settings.gradle.kts`, `gradlew`, `gradle/wrapper/`, `.gitignore`. Inside a multi-project build, that gives you nested builds that IntelliJ and `./gradlew` treat as separate projects. To avoid that:

1. **Create the root first:** *New Project → Gradle* (Kotlin DSL, JDK 25) at the repo root. This root owns the one wrapper, `settings.gradle.kts` and `gradle/libs.versions.toml`.
2. **Add each service** with *File → New → Module → Spring Boot* into `services/<name>`: Boot 4.x, Java 25, Gradle-Kotlin, Language = Java (booking, payment) or Kotlin (inventory). Use one package root for everything, e.g. `dev.sinabastani.booking…`.
3. **Clean up what the wizard generated in the module:** delete its `gradlew*`, `gradle/wrapper/`, `settings.gradle.kts` (and its `.gitignore` if the root has one). Add `include("services:booking")` to the root settings, then move the versions into the version catalog.
4. **Check:** `./gradlew projects` in the terminal lists every service, and IntelliJ shows **one** Gradle project in the Gradle tool window.

Doing the cleanup by hand is worth it, because it's exactly what teaches you how Gradle multi-project builds fit together. (Alternative: keep each service standalone and connect them with a Gradle *composite build* (`includeBuild`). It works well with the wizard, but sharing versions and conventions takes more effort, so it isn't recommended here.)

**Go service** (terminal/Zed): `go mod init github.com/<your-gh-user>/booking-labs/services/notification`.

## ⚠️ Docker: one runtime, also for Testcontainers

You run Docker from Zed/the terminal, so **Colima is fine**. You can quit Docker Desktop for this repo. The catch is **Testcontainers**: tests run via `./gradlew test` *or* from IntelliJ's gutter look for Docker through `DOCKER_HOST`, `~/.testcontainers.properties` or `/var/run/docker.sock`. They don't use the `docker context` Colima sets. On your machine, `/var/run/docker.sock` points to **Docker Desktop**, so tests would try to use Docker Desktop while your compose stack runs in Colima.

Fix once: create `~/.testcontainers.properties` pointing `docker.host` at `unix:///Users/<you>/.colima/default/docker.sock`, and apply the socket override from the Testcontainers Colima docs (check the current docs, because the setting names have changed between versions). Putting it in the properties file rather than `~/.zshrc` means tests run the same from IntelliJ (which doesn't read your shell config) and from the terminal.

**Check:** with Docker Desktop quit and Colima running, a Testcontainers test passes both from the IntelliJ gutter and from `./gradlew test`.

## Claude Code in both editors

- **Start Claude Code from the repo root** (any terminal, IntelliJ's or Zed's terminal, or Zed's agent panel) so `CLAUDE.md` and `.claude/skills` load.
- The JetBrains Claude Code plugin shows diffs in IntelliJ. In mentor mode those should be rare.
