# gradle-jfrog-private-test

Gradle counterpart of [maven-jfrog-private-test](https://github.com/Backline-playground/maven-jfrog-private-test) for **BKLN-2092** — validates that the JFrog Maven URL configured at the tenant level is what Gradle resolves through, with credentials supplied by the runner's `~/.gradle/init.gradle` injection.

## Topology

```
this repo (build.gradle)
   └── com.backline.test:demo-lib:1.0.0          ← lives in JFrog: maven-test-local
          └── org.apache.logging.log4j:log4j-core:2.14.1   ← Log4Shell (CVE-2021-44228)
                                                              served via the maven-test virtual repo
                                                              (Gradle resolves Maven-layout registries)
```

The JFrog setup is **shared with the Maven test repo** — same `maven-test-local` / `dummy-maven` / `maven-test` virtual. No new artifact uploads needed; `demo-lib:1.0.0` (vulnerable) and `demo-lib:2.0.0` (fixed log4j) are reused.

## The repository is declared, the credentials are not

`build.gradle` declares the `maven-test` virtual repo and nothing else — no `mavenCentral()`, because that virtual already proxies Central. So every artifact this build needs, public ones included, comes through the private registry, and a broken registry configuration fails the build outright rather than silently falling back to a public mirror.

Credentials stay out of the file. Backline's runner injects `$HOME/.gradle/init.gradle` at job-start time, which attaches the tenant's configured credentials to any repository on that host:

```groovy
allprojects {
    repositories.all { repo ->
        if (repo instanceof MavenArtifactRepository &&
                repo.url?.host == 'backlineai.jfrog.io') {
            repo.credentials {
                username = '<tenant username>'
                password = '<jfrog access token>'
            }
        }
    }
}
```

That callback comes from `renderGradleInit` in the runner (`services/runner/internal/model/package_registry_maven.go`). It deliberately does **not** declare a repository of its own: a project-level repo injected from an init script would silently disable every settings-level repository in Gradle 6.8+ builds that use `dependencyResolutionManagement`. A build that declares no repositories therefore resolves nothing — which is why this one declares the registry itself.

The same handler also writes `$HOME/.m2/settings.xml`, which the Maven-side tooling reads.

## Local development

Building this repo outside Backline needs the same credentials in your own `~/.gradle/init.gradle`, or passed explicitly:

```bash
./gradlew --init-script ~/.gradle/jfrog-init.gradle dependencies
```

## Why this exercises the private-registry path

1. Trivy scan of `build.gradle` flags `log4j-core:2.14.1` (transitive of `demo-lib`).
2. Environment setup resolves the build, which requires authenticating against JFrog — `demo-lib` exists nowhere else.
3. The coder's registry probe enumerates the repositories this build declares and queries them for available versions of the flagged package.
4. Upgrade analysis hands those repositories to the Java upgrade binary, which lists `demo-lib` versions and reads their POMs from JFrog to find a parent that brings a safe `log4j-core`.
5. Either `demo-lib` is bumped or `log4j-core` is pinned directly, depending on the remediation strategy.

Every one of those steps fails against a public registry, since `com.backline.test:demo-lib` is published only to JFrog.

## Files

- `build.gradle` — Groovy DSL, the JFrog repo plus a single dep on `com.backline.test:demo-lib:1.0.0`
- `settings.gradle` — root project name
- `src/main/java/.../App.java` — calls `Demo.greet()` from the private artifact
- `trivy-report.json` — pre-staged Log4Shell finding
