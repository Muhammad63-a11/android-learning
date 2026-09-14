# AI App Doctor — APK Builder (GitHub Actions)

This is a GitHub Actions port of the **AI App Doctor Pro** Colab notebook.
It takes an Android project ZIP, examines it, diagnoses problems,
applies safe automatic repairs, builds it with Gradle, retries with more
repairs on failure, and only ever hands back an APK that has passed
ZIP/manifest/DEX/signature validation.

## What the workflow does, step by step

1. Checks out the repo.
2. Installs Python 3.13 and the Python requirements (`reportlab`, used to
   generate the PDF report).
3. Confirms `input/` contains exactly one Android project ZIP.
4. Runs `scripts/app_doctor_pipeline.py`, which:
   - installs **OpenJDK 17**, `wget`, `unzip`, `zip`;
   - downloads and sets up the **Android command-line tools**, accepts the
     SDK licenses, installs the **SDK platform matching the detected
     `compileSdk`**, and installs **Build-Tools 36.0.0**;
   - creates `local.properties`;
   - safely extracts the ZIP (path-traversal / zip-bomb checked) and
     detects the Android project root;
   - runs the same static examination as the notebook (Gradle wrapper
     health, manifest/settings/build files, `compileSdk`/AGP/Kotlin
     version detection, a secret scanner, source inventory) and produces
     a health score;
   - turns the findings into a diagnosis + treatment plan;
   - runs the real Gradle build, and on failure applies **deterministic,
     conservative repairs** (AndroidX flag, JVM memory args, legacy
     `compileSdkVersion` syntax, Java 1.8 -> 17 targets, missing debug
     keystore, `gradlew` line endings, missing `namespace`, etc.), then
     retries — up to 12 attempts, with a full project backup before every
     attempt;
   - **additionally**, because a GitHub Actions runner starts far emptier
     than Colab, the loop also auto-recovers from a few
     environment-level failure categories: a missing Android SDK
     platform/build-tools version reported by Gradle, a JDK-version
     mismatch reported by Gradle/AGP, transient network/dependency
     resolution failures (retried with `--refresh-dependencies`), and
     `OutOfMemoryError` (heap raised and retried);
   - once Gradle succeeds, locates the exact APK it produced and
     validates it: ZIP integrity, `AndroidManifest.xml` present,
     `classes*.dex` present, manifest/package decoding via `aapt2`,
     signature verification via `apksigner`, and a SHA-256 identity check
     — a Gradle exit code of `0` is **never** treated as sufficient on
     its own;
   - writes a single PDF report (`output/app_doctor_report.pdf`) covering
     the health score, findings, diagnosis, every repair attempted, the
     build result, and the APK validation result — **every run**, success
     or failure;
   - if (and only if) the APK passed every validation gate, copies it to
     `output/<package>-debug-validated.apk`.
5. Uploads whatever ended up in `output/` (PDF always; APK only if one
   was actually built and validated) as a workflow artifact named
   **`ai-app-doctor-validated-output`**, via `if: always()` — so you get
   the report even when the build fails.

## How to use it

1. Put **exactly one** Android project ZIP in `input/`, e.g.
   `input/my-app.zip`.
2. Commit and push (the workflow triggers on changes under `input/`), or
   go to **Actions -> AI App Doctor APK Builder -> Run workflow** to run
   it manually at any time.
3. When the run finishes, open it and download the
   **`ai-app-doctor-validated-output`** artifact from the *Artifacts*
   section at the bottom of the run summary.
4. Inside you'll find `app_doctor_report.pdf` and, if the build
   succeeded and validated, the APK file.

## What this can't honestly promise

No CI pipeline can guarantee that *every* arbitrary Android project will
build without errors — different projects need different JDK/NDK/SDK/AGP
combinations, Maven repositories, or native dependencies. What this
pipeline does promise:

- it will **never** silently rewrite your Kotlin/Java/XML source to force
  a build through — only safe, deterministic, config-level repairs are
  applied automatically;
- it will **never** hand you an APK that hasn't passed every validation
  gate (ZIP integrity, manifest, DEX, signature, SHA-256 identity);
- you will **always** get a PDF report explaining exactly what was
  found, what was tried, and — if it failed — the real Gradle error
  evidence, so you know exactly what to fix by hand if automatic repair
  couldn't do it (this applies to genuine source-code compile errors,
  which are intentionally never auto-patched).

## Adjusting build time

`MAX_REPAIR_ATTEMPTS` (12) and `GRADLE_TIMEOUT_SECONDS` (1200s / 20 min
per attempt) are set at the top of
`scripts/app_doctor_pipeline.py`. The workflow's job-level
`timeout-minutes: 180` gives room for the worst case; lower both if you
want faster failure feedback on a project you already know builds
quickly.
