# gradle-jfrog-private-test

Gradle counterpart of [maven-jfrog-private-test](https://github.com/Backline-playground/maven-jfrog-private-test) for **BKLN-2092** — validates that the JFrog Maven URL configured at the tenant level reaches Gradle via the runner's `~/.gradle/init.gradle` injection.

## Topology

```
this repo (build.gradle)
   └── com.backline.test:demo-lib:1.0.0          ← lives in JFrog: maven-test-local
          └── org.apache.logging.log4j:log4j-core:2.14.1   ← Log4Shell (CVE-2021-44228)
                                                              served via the maven-test virtual repo
                                                              (Gradle resolves Maven-layout registries)
```

The JFrog setup is **shared with the Maven test repo** — same `maven-test-local` / `dummy-maven` / `maven-test` virtual. No new artifact uploads needed; `demo-lib:1.0.0` (vulnerable) and `demo-lib:2.0.0` (fixed log4j) are reused.

## Why no `repositories { ... }` block in `build.gradle`

Backline's runner injects `$HOME/.gradle/init.gradle` at job-start time:

```groovy
allprojects {
    repositories {
        maven {
            url 'https://backlineai.jfrog.io/artifactory/maven-test/'
            credentials(HttpHeaderCredentials) {
                name = 'Authorization'
                value = 'Bearer <jfrog-access-token>'
            }
            authentication { header(HttpHeaderAuthentication) }
        }
    }
}
```

That's done by `JFrogRegistryMavenPackageRegistryHandler.GetPackageRegistryInitCommands` in the runner. The same handler also writes `$HOME/.m2/settings.xml` (Maven), which is what registry-aware version-discovery commands like `mvn dependency:get` use during analysis.

## Local development

If you want to build this repo locally (outside Backline), drop a similar `init.gradle` into your own `~/.gradle/` with your JFrog credentials, or pass it via `--init-script`:

```bash
./gradlew --init-script ~/.gradle/jfrog-init.gradle dependencies
```

## Why this exercises the new handler

1. Trivy scan of `build.gradle` flags `log4j-core:2.14.1` (transitive of `demo-lib`).
2. Backline SCA Planning resolves the upgrade. To inspect `demo-lib`'s POM and check for fixed versions it has to authenticate against JFrog.
3. The runner's Gradle init script + Maven settings.xml provide that auth.
4. The analysis prompt (BKLN-2092 update) queries the configured registry (`mvn dependency:get` honors settings.xml) to find available versions — discovers `demo-lib:2.0.0` brings safe `log4j-core:2.17.1`.
5. Either `demo-lib` is bumped or `log4j-core` is pinned directly, depending on the LLM's remediation strategy.

## Files

- `build.gradle` — Groovy DSL, single dep on `com.backline.test:demo-lib:1.0.0`
- `settings.gradle` — root project name
- `src/main/java/.../App.java` — calls `Demo.greet()` from the private artifact
- `trivy-report.json` — pre-staged Log4Shell finding
