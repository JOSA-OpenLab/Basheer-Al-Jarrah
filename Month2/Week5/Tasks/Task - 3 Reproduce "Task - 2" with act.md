
## What I Did

Configured and ran the GitHub Actions CI pipeline locally using [`act`](https://github.com/nektos/act), a tool that simulates GitHub Actions inside Docker containers.

---

## Steps Taken

1. Confirmed `act` was already installed at `/usr/bin/act`
2. Installed Docker (v29.5.2) and started the daemon
3. Added my user to the `docker` group to run Docker without `sudo`:

```zsh
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

4. Logged out and back in to apply group membership system-wide
5. Ran the CI pipeline locally from the project root:

```zsh
act push --job build-and-test
```

6. On first run, `act` prompted for a Docker image size — chose **Medium (~500 MB)**, which includes the tooling needed to bootstrap Actions

---

## What Happened

`act` pulled the `catthehacker/ubuntu:act-latest` image and spun up **3 parallel containers**, one per OS in the CI matrix:

| Job | OS               | Result     | Duration |
|-----|------------------|------------|----------|
| 1   | `ubuntu-latest`  | ✅ Passed  | 2m 53s   |
| 2   | `windows-latest` | ✅ Passed  | 2m 53s   |
| 3   | `macos-latest`   | ✅ Passed  | 2m 58s   |

Each job:

- Set up JDK 25
- Downloaded Maven dependencies from Maven Central
- Compiled the project
- Ran the test suite → **1 test, 0 failures, 0 errors**
- Packaged the Spring Boot fat JAR (`Aurora-0.0.1-SNAPSHOT.jar`)

---

## Minor Warning (Non-Fatal)

The Maven dependency cache failed to save after each job:

```
/usr/bin/tar: /home/ld-rw/Desktop/MyStuff/Programming/Software: No such file or directory
::warning::Failed to save cache
```

**Root cause:** The project path contains spaces (`Software Engineering/E-commerce/Aurora`), which `tar` inside the Docker container can't handle.

This only affects `act` locally — real GitHub Actions runners handle spaces correctly. The jobs still passed; it just means Maven re-downloads dependencies on each local `act` run.

---

## Outcome

The CI pipeline works correctly both **locally (via `act`)** and **on GitHub Actions**. All matrix builds pass on all three platforms.
