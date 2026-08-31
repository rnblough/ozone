# Ozone CI with GitHub Actions

The Ozone project uses GitHub Actions (GA) for its CI system.  GA is implemented with *workflows*, which are groups of *jobs* combined to accomplish a CI task, all defined in a single YAML file.  The Ozone workflow YAML files live in [.github/workflows](./workflows).

Every factual claim below is annotated with the file (and line, where useful) that backs it, so drift between this document and the actual CI is easier to spot.

## Overview

- The entry point for both push and pull-request events is [`post-commit.yml`](./workflows/post-commit.yml), whose workflow name is `build-branch`.  Its single job invokes [`ci.yml`](./workflows/ci.yml) (workflow name `full-ci`) via `workflow_call`.
- Cross-job cancellation for a given PR (or non-`apache/ozone` push) is handled by the `concurrency:` block in [`post-commit.yml`](./workflows/post-commit.yml) lines 20-22.  There is no separate "Cancelling" workflow.
- The default JDK for build and test steps is set by `TEST_JAVA_VERSION: 21` in [`ci.yml`](./workflows/ci.yml) line 43, and duplicated in [`populate-cache.yml`](./workflows/populate-cache.yml) line 39.  It matches the JDK bundled in the [`ghcr.io/apache/ozone-runner`](./workflows/check.yml) image.
- All matrix jobs use `fail-fast: false` ([`ci.yml`](./workflows/ci.yml) lines 181, 215, 306, 352).  A failure in one matrix leg does not cancel its siblings.
- Nearly every `ci.yml` job delegates its work to the reusable [`check.yml`](./workflows/check.yml) harness, which handles checkout, cache restore, Java setup, script execution, and artifact upload.  Only `coverage`, `generate-config-doc`, and `update-ozone-site-config-doc` don't use it.

## Parallelism

The CI pipeline layers several parallelism mechanisms, some effective and some dormant.  The status of each is called out below so that a future reader can tell working knobs from ones that have been added, disabled, or never activated.

### Job-level fanout (working)

- The `concurrency:` group in [`post-commit.yml`](./workflows/post-commit.yml) lines 20-22 cancels superseded runs for PRs and non-`apache/ozone` pushes.
- Four matrices in [`ci.yml`](./workflows/ci.yml) run their legs concurrently, all with `fail-fast: false`:
  - `compile` (lines 174-181): Java **8**, **11**, **17** on `ubuntu-24.04` plus Java **21** on `macos-15`.
  - `basic` (lines 212-215): one leg per basic check selected by `build-info`.
  - `acceptance` (lines 303-306): one leg per suite from [`acceptance_suites.sh`](../dev-support/ci/acceptance_suites.sh).
  - `integration` (lines 349-352): one leg per profile from [`integration_suites.sh`](../dev-support/ci/integration_suites.sh) (`client`, `container`, `filesystem`, `flaky`, `hdds`, `om`, `ozone`, `recon`, `snapshot`).
- Every matrix leg is dispatched through [`check.yml`](./workflows/check.yml).  The `split` input (lines 114-118) is used **only** to disambiguate the job display name (line 150) and the uploaded artifact name (line 272) — it is not forwarded to the check script as an env var.  Work division for the four `ci.yml` matrices happens through `script-args` (e.g. `-Ptest-${{ matrix.profile }}` at line 344), not through `split`.
- [`intermittent-test-check.yml`](./workflows/intermittent-test-check.yml) (lines 88-94, 160-162) and [`repeat-acceptance.yml`](./workflows/repeat-acceptance.yml) (lines 72-78, 133-135) construct a `[1..N]` split matrix, but each leg runs **the same test class or suite** — splits are concurrent replicas for flakiness detection, not a work-division scheme.  In `intermittent-test-check.yml`, the total number of executions is `splits × iterations`, where `iterations` drives the sequential `ITERATIONS` loop inside [`junit.sh`](../hadoop-ozone/dev-support/checks/junit.sh) (lines 28, 68).

### Maven reactor parallelism (`-T 1C`)

`-T 1C` runs one Maven builder thread per available core across the *reactor* — it parallelizes module builds, not Surefire forks within a single module.  It was added recently to nearly every check script.  For per-module goals it works; for aggregator goals that execute once at the reactor root it is a no-op.

| Script | Line | Goal | Effective? |
|---|---|---|---|
| [`_build.sh`](../hadoop-ozone/dev-support/checks/_build.sh) | 30 | full build lifecycle (used by `build.sh`, `compile.sh`, `repro.sh`) | Yes |
| [`checkstyle.sh`](../hadoop-ozone/dev-support/checks/checkstyle.sh) | 27 | `checkstyle:check` | Yes |
| [`pmd.sh`](../hadoop-ozone/dev-support/checks/pmd.sh) | 29 | `pmd:check` | Yes |
| [`findbugs.sh`](../hadoop-ozone/dev-support/checks/findbugs.sh) | 33 | `spotbugs:check` | Yes |
| [`rat.sh`](../hadoop-ozone/dev-support/checks/rat.sh) | 27 | `apache-rat-plugin:check` | Yes |
| [`javadoc.sh`](../hadoop-ozone/dev-support/checks/javadoc.sh) | 26 | `javadoc:aggregate` | **No** — aggregator goal runs once at the reactor root; `-T` cannot split it |
| [`license.sh`](../hadoop-ozone/dev-support/checks/license.sh) | 45 | `license:aggregate-add-third-party` | **No** — same reason as `javadoc:aggregate` |
| [`junit.sh`](../hadoop-ozone/dev-support/checks/junit.sh) | 38 | `verify` (used by the `integration` job via [`integration.sh`](../hadoop-ozone/dev-support/checks/integration.sh)) | Runs, but with a caveat — see below |
| [`populate-cache.yml`](./workflows/populate-cache.yml) | 38, 100 | `-Pgo-offline clean verify` and Java 8 `test-compile` | Yes |

The `-T 1C` on `junit.sh` runs multiple modules' Surefire executions concurrently, one per builder thread.  Because the CI `integration` job does not restrict the reactor (`ci.yml` line 344 passes only `-Ptest-<profile> -Drocks_tools_native`), several modules can execute Surefire in parallel.  Each individual Surefire still uses the default `forkCount=1` because the [`parallel-tests` profile](#dormant-knobs) that would apply per-fork isolation is not activated.  `MiniOzoneCluster` (which binds fixed default ports and shares `test.build.data` when isolation is off) can therefore run in more than one JVM at once with no coordination.  The flag is not broken — Maven does start parallel builders — but the isolation contract that would make cluster-based tests safe under it is not in effect.

### Surefire and JVM knobs (working)

These apply to every test run, whether or not any `-T` flag is present.

| Knob | Location | Effect |
|---|---|---|
| `reuseForks=false` | [`pom.xml`](../pom.xml) line 2373 | Each test class runs in a fresh JVM. |
| `forkedProcessTimeoutInSeconds=${surefire.fork.timeout}` (default `1200`s) | [`pom.xml`](../pom.xml) lines 2374, 213 | Kills stuck forks. |
| `-Xmx8192m -XX:+HeapDumpOnOutOfMemoryError` | [`pom.xml`](../pom.xml) line 150 | Per-fork heap ceiling, injected into `argLine`. |
| `MALLOC_ARENA_MAX=4` | [`pom.xml`](../pom.xml) line 2379 | Caps glibc arenas to reduce RSS across parallel forks. |
| `surefire.rerunFailingTestsCount=5` + `surefire.fork.timeout=3600` | [`integration.sh`](../hadoop-ozone/dev-support/checks/integration.sh) lines 22-23 | Auto-injected only when `-Ptest-flaky` is in the args. |

The [`test-flaky` profile](../pom.xml) (line 2888) is one of the profiles emitted by `integration_suites.sh`, so the `flaky` matrix leg is what enables both reruns and the 60-minute per-fork timeout.

### Dormant knobs

- **`parallel-tests` Maven profile** — defined in [`hadoop-hdds/pom.xml`](../hadoop-hdds/pom.xml) lines 94-138 and [`hadoop-ozone/pom.xml`](../hadoop-ozone/pom.xml) lines 140-185.  Would set `forkCount=${testsThreadCount}`, `reuseForks=false`, per-fork `test.build.data` / `hadoop.tmp.dir` / `test.unique.fork.id` system properties, and wire in the `parallel-tests-createdir` goal.  Nothing under [`.github/workflows`](./workflows) or [`hadoop-ozone/dev-support/checks/`](../hadoop-ozone/dev-support/checks) activates it.  The `testsThreadCount` property ([`pom.xml`](../pom.xml) line 218, default 4), the per-fork sysprops, and the `-DminiClusterDedicatedDirs=true` flag inside the profile are therefore unused as CI is configured.
- **JUnit 5 parallel execution** — [`pom.xml`](../pom.xml) lines 2400-2404 set `junit.jupiter.execution.parallel.enabled = true`, but the default execution mode is `same_thread`, so tests only run concurrently if they opt in via `@Execution(CONCURRENT)`.  A repo-wide grep for `@Execution(` and `@Isolated` finds two annotated files (`TestDiskCheckUtil`, `TestOmSnapshotManagerConfig`), so in practice this engine is doing no meaningful concurrency.
- **`check.yml`'s `split` input as a work-division parameter** — the input exists (lines 114-118) but is never passed into the check script.  Work division for `acceptance` and `integration` happens through `script-args`, not through `split`.

## Workflows

### build-branch ([post-commit.yml](./workflows/post-commit.yml))

- Triggers: `pull_request` types `[opened, ready_for_review, synchronize]` and any `push` (lines 17-19).
- `concurrency:` group cancels in-progress runs for PRs and non-`apache/ozone` pushes (lines 20-22).
- The single `CI` job invokes [`ci.yml`](./workflows/ci.yml) via `workflow_call` (line 30) and forwards the `DEVELOCITY_ACCESS_KEY`, `OZONE_WEBSITE_BUILD`, and `SONARCLOUD_TOKEN` secrets.

### full-ci ([ci.yml](./workflows/ci.yml))

The main CI workflow.  Runs on `workflow_call` (from `post-commit.yml` and `ci-with-ratis.yml`) and `workflow_dispatch`.

For jobs that use [`check.yml`](./workflows/check.yml), the default runner is `ubuntu-24.04` and only non-default settings are called out below.

#### build-info

Runs [`dev-support/ci/selective_ci_checks.sh`](../dev-support/ci/selective_ci_checks.sh) on `ubuntu-slim` to determine which of the other jobs are needed based on which files the PR changed.  If triggered by anything other than a PR — or if the PR has a label containing `full tests needed` — every downstream job runs.

For each type of check, the script matches the changed files against a regex list to set an output flag.  For example, the kubernetes check uses ([selective_ci_checks.sh lines 279-296](../dev-support/ci/selective_ci_checks.sh)):

```
    local pattern_array=(
        "^hadoop-ozone/dev-support/checks/kubernetes.sh"
        "^hadoop-ozone/dev-support/checks/install/flekszible.sh"
        "^hadoop-ozone/dev-support/checks/install/k3s.sh"
        "^hadoop-ozone/dist"
    )
    local ignore_array=(
        "^hadoop-ozone/dist/src/main/compose"
        "^hadoop-ozone/dist/src/main/license"
        "\.md$"
    )
```

`build-info` also invokes [`acceptance_suites.sh`](../dev-support/ci/acceptance_suites.sh) and [`integration_suites.sh`](../dev-support/ci/integration_suites.sh) to produce the matrix lists consumed by the `acceptance` and `integration` jobs, and [`categorize_basic_checks.sh`](../dev-support/ci/categorize_basic_checks.sh) for `basic`.

#### build

Runs [`build.sh`](../hadoop-ozone/dev-support/checks/build.sh) at Java 21.  This is the artifact-producing job: on success it uploads three artifacts consumed by downstream jobs — `ozone-bin`, `ozone-src`, and `ozone-repo` (see [`check.yml`](./workflows/check.yml) lines 279-305).  Timeout 60 minutes.

#### generate-config-doc / update-ozone-site-config-doc

- `generate-config-doc` (via [`generate-config-doc.yml`](./workflows/generate-config-doc.yml)) extracts the Ozone binary tarball and runs `dev-support/ci/xml_to_md.py` to produce `Configurations.md`.  Runs on PRs and on `master` pushes.
- `update-ozone-site-config-doc` (via [`update-ozone-site-config-doc.yml`](./workflows/update-ozone-site-config-doc.yml)) opens a PR against `apache/ozone-site` when the generated docs change.  Runs only on `master` pushes in the `apache/ozone` repo.

#### compile

Runs [`compile.sh`](../hadoop-ozone/dev-support/checks/compile.sh) across a matrix of Java versions to verify multi-JDK compatibility.  The matrix ([`ci.yml`](./workflows/ci.yml) lines 174-181):

- Java **8, 11, 17** on `ubuntu-24.04`
- Java **21** on `macos-15`

`fail-fast: false`.  The `-Dmaven.compiler.release=${{ matrix.java }}` flag ensures each leg targets its own JDK.  Timeout 45 minutes.

#### basic

Runs one of the following basic checks per matrix leg, selected by `build-info` output ([`ci.yml`](./workflows/ci.yml) lines 197-215).  Runs at Java 21 except for `findbugs`, which is pinned to Java 8 as an [HDDS-10150](https://issues.apache.org/jira/browse/HDDS-10150) workaround (line 204).  `fail-fast: false`.

- author — [`author.sh`](../hadoop-ozone/dev-support/checks/author.sh): confirms no Java source contains `@author`
- bats — [`bats.sh`](../hadoop-ozone/dev-support/checks/bats.sh): shell tests via [bats-core](https://github.com/bats-core/bats-core)
- checkstyle — [`checkstyle.sh`](../hadoop-ozone/dev-support/checks/checkstyle.sh): Maven Checkstyle plugin
- docs — [`docs.sh`](../hadoop-ozone/dev-support/checks/docs.sh): builds the site with [Hugo](https://gohugo.io/)
- findbugs — [`findbugs.sh`](../hadoop-ozone/dev-support/checks/findbugs.sh): SpotBugs static analysis
- pmd — [`pmd.sh`](../hadoop-ozone/dev-support/checks/pmd.sh): PMD static analysis
- rat — [`rat.sh`](../hadoop-ozone/dev-support/checks/rat.sh): Apache RAT license header check

#### dependency

Runs [`dependency.sh`](../hadoop-ozone/dev-support/checks/dependency.sh) against the built tarball to confirm the shipped jars match [`jar-report.txt`](../hadoop-ozone/dist/src/main/license/jar-report.txt).  If they don't, the output describes how to update the report (or which changes to revert).  Timeout 5 minutes.

#### license

Runs [`license.sh`](../hadoop-ozone/dev-support/checks/license.sh) against the built repo to enforce license constraints.  Timeout 15 minutes.

#### javadoc

Runs [`javadoc.sh`](../hadoop-ozone/dev-support/checks/javadoc.sh) against the built repo when compile is needed.  Timeout 30 minutes.

#### repro

Runs [`repro.sh`](../hadoop-ozone/dev-support/checks/repro.sh) to verify the build is reproducible.  On failure runs [`_diffoscope.sh`](../hadoop-ozone/dev-support/checks/_diffoscope.sh) as `post-failure` ([`ci.yml`](./workflows/ci.yml) line 277).  Timeout 30 minutes.

#### acceptance

Runs [`acceptance.sh`](../hadoop-ozone/dev-support/checks/acceptance.sh) (Robot Framework smoketests against a real docker-compose cluster) once per suite, in parallel.  Pinned to **Java 11** because Hadoop may not work with newer JDKs ([`ci.yml`](./workflows/ci.yml) line 294).  `fail-fast: false`.  Timeout 150 minutes.

The suite list is generated dynamically by [`acceptance_suites.sh`](../dev-support/ci/acceptance_suites.sh), which scans `#suite:` markers under `hadoop-ozone/dist/src/main/compose` and excludes the `failing` suite.  Current suites include `balancer`, `cert-rotation`, `compat-new`, `compat-old`, `diskbalancer`, `EC`, `HA-secure`, `HA-unsecure`, `leadership`, `misc`, `MR`, `s3a`, `secure`, `tools`, `unsecure`, and `upgrade`.

#### kubernetes

Runs [`kubernetes.sh`](../hadoop-ozone/dev-support/checks/kubernetes.sh) against the built tarball when Kubernetes-related files changed.  Timeout 60 minutes.

#### integration

Runs [`integration.sh`](../hadoop-ozone/dev-support/checks/integration.sh) with `-Ptest-<profile>` once per profile, in parallel.  Java 21.  `fail-fast: false`.  Timeout 90 minutes.

The profile list is generated by [`integration_suites.sh`](../dev-support/ci/integration_suites.sh) from the `<id>test-*</id>` entries in `pom.xml`.  Current profiles: `client`, `container`, `filesystem`, `flaky`, `hdds`, `om`, `ozone`, `recon`, `snapshot`.

#### coverage

Runs inline (does not use `check.yml`) on `ubuntu-24.04`.  Only runs on `push` events ([`ci.yml`](./workflows/ci.yml) line 356).  Calls [`coverage.sh`](../hadoop-ozone/dev-support/checks/coverage.sh) to merge JaCoCo data, then [`sonar.sh`](../hadoop-ozone/dev-support/checks/sonar.sh) to publish to SonarCloud.  `needs:` `build-info`, `acceptance`, and `integration`.  Timeout 30 minutes.

### close-stale-prs ([close-stale-prs.yaml](./workflows/close-stale-prs.yaml))

Scheduled at `0 0 * * *` (nightly).  Uses [`actions/stale`](https://github.com/actions/stale) to mark PRs stale after 21 days of inactivity and close them 7 days later.  A comment or update removes the stale label.

### pull-request ([pull-request.yml](./workflows/pull-request.yml))

Runs on `pull_request` events `[reopened, opened, edited, synchronize]` on `ubuntu-slim`.  Validates the PR title via [`pr_title_check.sh`](../dev-support/ci/pr_title_check.sh).

### populate-cache / scheduled-cache-update

- [`populate-cache.yml`](./workflows/populate-cache.yml) — populates the shared Maven cache.  Triggers: `push` to release/master branches when a `pom.xml` or the workflow itself changes, plus `workflow_call` and `workflow_dispatch`.  Warms dependencies for both Java 21 (default) and Java 8 (needed by `findbugs` and `compile` matrix leg).
- [`scheduled-cache-update.yml`](./workflows/scheduled-cache-update.yml) — cron `20 3 * * *`.  Calls `populate-cache.yml` daily.

### zizmor ([zizmor.yml](./workflows/zizmor.yml))

Runs `zizmorcore/zizmor-action` on `ubuntu-latest` for `push` (excluding `dependabot/**`) and `pull_request`.  Static security analysis of the workflow YAML files.

### label-pull-requests / scheduled-label-pull-requests

- [`label-pr.yml`](./workflows/label-pr.yml) — reusable; also supports `workflow_dispatch`.  Adds labels to PRs targeting feature branches based on hard-coded rules.
- [`schedule-label-pr.yml`](./workflows/schedule-label-pr.yml) — cron `*/5 * * * *`.  Calls `label-pr.yml`.

### Developer utility workflows (manual `workflow_dispatch` only)

- **flaky-test-check** ([`intermittent-test-check.yml`](./workflows/intermittent-test-check.yml)) — repeatedly runs a specific test class across parallel splits to surface flakiness.  Defaults to 10 splits × 10 iterations at Java 21.  Can optionally build against a custom Ratis ref via [`build-ratis.yml`](./workflows/build-ratis.yml).
- **repeat-acceptance-test** ([`repeat-acceptance.yml`](./workflows/repeat-acceptance.yml)) — repeats one acceptance suite (or a filter regex) across parallel splits.  Uses Java 8.
- **ci-with-ratis** ([`ci-with-ratis.yml`](./workflows/ci-with-ratis.yml)) — runs the full-ci workflow against a custom Ratis build.  Calls `build-ratis.yml` first, then `ci.yml` with the resulting version overrides in `ratis_args`.

### Reusable internal workflows (`workflow_call` only)

- [`check.yml`](./workflows/check.yml) — the harness that runs a single check script from `hadoop-ozone/dev-support/checks/`.  Handles checkout, Maven and NPM cache restore, artifact download (source tarball, binary tarball, Ozone repo, Ratis jars), Java setup, pre/post scripts, script execution, artifact upload, and failure summary.  Almost every job in `ci.yml` uses this.  Default runner is `ubuntu-24.04`.  Container image env vars live at lines 138-143 (`ghcr.io/apache/hadoop`, `ghcr.io/apache/ozone`, `ghcr.io/apache/ozone-runner`).
- [`build-ratis.yml`](./workflows/build-ratis.yml) — builds a custom Ratis snapshot at Java 8, uploads it as the `ratis-jars` artifact, and emits version-override args for the calling workflow.
- [`generate-config-doc.yml`](./workflows/generate-config-doc.yml) and [`update-ozone-site-config-doc.yml`](./workflows/update-ozone-site-config-doc.yml) — invoked directly as jobs from `ci.yml`, described above.

## Old / Deprecated Workflows

The workflows `main.yml` (Build), `chaos.yml` (former `build-branch`), and `pr.yml` (pr-check) were removed from this repository.  Their files no longer exist under [`.github/workflows/`](./workflows).  They are mentioned here only so readers who arrive from historical GitHub Actions run URLs on the [apache/ozone actions page](https://github.com/apache/ozone/actions) have some context for what those workflows used to be.

## Tips

- When a build of the Ozone master branch fails, its artifacts are stored [here](https://elek.github.io/ozone-build-results/).
- To trigger rerunning the tests, push an empty commit to your PR: `git commit --allow-empty -m 'trigger new CI check'`.
- [This wiki](https://cwiki.apache.org/confluence/display/OZONE/Running+Ozone+Smoke+Tests+and+Unit+Tests) contains tips on running tests locally.
- [This wiki](https://cwiki.apache.org/confluence/display/OZONE/Github+Actions+tips+and+tricks) contains tips on special handling of the CI system, such as executing one test multiple times or SSHing into the CI machine while tests are running.
