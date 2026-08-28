Some yml templates I use on my azure devops server. Needed by some published projects. Keep in mind that currently you cannot reference github from azure devops <ins>server</ins>. Templates need to be in the same organisation when used by pipelines on devops server.

<h6>GenerateAndPublishCodeCoverage.yml</h6> 

- requires the reportgenerator package 'dotnet tool update -g dotnet-reportgenerator-globaltool'. Which should be run at some point in time before the unit tests are executed.
- tests should be run like 'dotnet test -c $(BuildConfiguration) --no-build --collect:"XPlat Code Coverage"'.
- coverage is picked up from `$(Agent.TempDirectory)/**/coverage.*`, i.e. wherever the VSTest coverlet data collector writes it — not from a project's own build output directory.
- runs with `condition: succeeded()` and `continueOnError: false`, so it's skipped (and the job fails) if an earlier step already failed.
- uses `PublishCodeCoverageResults@2`. Tried on 2026-07-04, failed at the time because `@2` spins up
  a separate `CoveragePublisher.Console.exe` that connects back to the Azure DevOps server itself
  over Basic/PAT auth, and that tool hard-refuses to do so over plain HTTP
  (`System.InvalidOperationException: Basic authentication requires a secure connection to the server`).
  Reverted to `@1` at the time. Since then the build agent (`KONFIDENCE8`) was given an HTTPS
  connection to the server (see `TfsHttpsMigration.md` in the `DayTradingServices` repo) — `@2` was
  re-adopted the same day and confirmed working.

<h6>PublishKonfidenceToGithub.yml</h6>

- pushes every branch (except 'master'/'main') to `https://github.com/a3helmich/<githubPrefix>.$(Build.Repository.Name).git`.
- `githubPrefix` is a template parameter, default `Konfidence` (matches existing consumers without
  any change needed). Added 2026-07-05 since the prefix used to be hardcoded, which breaks for
  repos whose Azure DevOps project doesn't match their intended GitHub naming — e.g.
  `Producten/ProjectReferences` still targets `Konfidence.ProjectReferences` on GitHub (its Azure
  DevOps project is `Producten`, but that's a project-placement quirk, not its actual naming
  convention — verify the *intended* GitHub target per repo rather than assuming it matches the
  Azure DevOps project name).
- requires a `GitHubToken` variable (Variable Group `GitHubTokens`) authorized for the calling
  pipeline — note each project has its **own** `GitHubTokens` group (Variable Groups aren't shared
  cross-project here, same pattern as `nuget.org-apikeys`), so a new consumer project needs its own
  copy created with the same token value, not a reference to Konfidence's.
- calling pipeline's checkout step needs `fetchDepth: 0` (full history), since every branch is pushed.
- runs with `condition: succeededOrFailed()` and `continueOnError: true`, so it still runs (and won't fail the job) even if an earlier step in the pipeline failed.
- checks out each branch with `git checkout -B <branch> <origin/branch>` (force-reset), not plain `git checkout <branch>` — required because self-hosted agents reuse their working directory across runs, so a plain checkout of an already-existing local branch would freeze it at whatever commit it had from a *previous* run instead of advancing it to match origin's current state. Found 2026-07-03 after this silently stopped updating GitHub's `develop` branch for several runs.

<h6>ComputeVersion.yml</h6>

- **variables template** (not a steps template — included via `variables: - template: ComputeVersion.yml@templates`, not `steps:`), added 2026-07-05. Needed because `counter()` runtime expressions can only be used inside a `variables:` block, not from within a steps template.
- computes a `year.majorVersion.minorVersion` package version (CalVer-style), e.g. `2026.1.42`.
- `majorVersion` is a required parameter, set by the calling pipeline — a human-maintained value: `1` at the start of each year, bumped manually mid-year only for a deliberate major-version change.
- `year` is the current calendar year, computed automatically from `pipeline.startTime`.
- `minorVersion` uses Azure Pipelines' `counter()` expression keyed on `year.majorVersion` (e.g. `"2026.1"`) — this means the counter automatically starts fresh at 1 whenever the key changes, which happens for free both when `majorVersion` is bumped **and** when the year rolls over, with no extra logic needed.
- the counter increments on **every** run that evaluates it, regardless of trigger (PR validation, scheduled nightly, manual, etc.) — evaluating the `variables:` block always ticks it, there's no way to make `counter()` itself conditional. This means the published version sequence can have gaps, but that's fine since not every run that ticks the counter actually packs/publishes.
- usage in the calling pipeline: pass `/p:Version=$(year).$(majorVersion).$(minorVersion)` on the `dotnet build` step's `arguments` (so it's baked into the compiled assembly's version metadata), and `buildProperties: 'Version=$(year).$(majorVersion).$(minorVersion)'` on the `dotnet pack` step (since `--nobuild` packs don't re-evaluate the build step's properties) — using the same `Version` property (not `PackageVersion`) on both keeps the DLL's assembly version and the NuGet package version identical. A static `<Version>` in `Directory.Build.props` (if present) is overridden by this at build/pack time; it only matters for local, non-CI builds.
- **gate the actual `dotnet pack`/publish steps** on `condition: eq(variables['Build.Reason'], 'PullRequest')` (or whatever trigger should be the real publish event for that pipeline) so packages are only published for the intended trigger, not every run that happens to evaluate the version counter. `ProjectReferences` originally had no such condition, so its nightly schedule *and* every PR validation run each published a real version to nuget.org — found and fixed 2026-07-07, after `Konfidence.Project-References` had already accumulated versions `2026.1.0`–`2026.1.5` from this. Since this pipeline has no separate merge-triggered (`IndividualCI`) run configured at all (`trigger: none`, only a nightly `schedules:` block), the actual "did someone merge good code" signal here is a *successful PR validation build*, not a post-merge push — so `PullRequest` is the correct condition value for this repo, not `IndividualCI`. Verify which trigger is the real "just published" signal for each specific pipeline rather than assuming.

<h6>VerifyVersionMatchesProps.yml</h6>

- **steps template**, added 2026-08-23. Fails the build when the pipeline's `majorVersion` and the `<Version>` in the repository's `Directory.Build.props` have drifted apart.
- takes one optional `propsPath` parameter (default `Directory.Build.props`), for a consumer whose props file lives elsewhere.
- reads the pipeline's `$(year)`/`$(majorVersion)` variables directly at runtime rather than taking them as parameters, since callers already set them via `ComputeVersion.yml@templates` earlier in the same `variables:` block.
- **year and major are compared, the minor is not.** `minorVersion` is a `counter()`, so it would match the props file exactly once and be wrong from the second build onward; the props file's minor is a separate, hand-maintained number.
- the **year check fails by design each 1 January**: `year` rolls over on its own while the props file cannot, and NuGet reads that leading segment as the semver major, so the repository must not keep claiming last year's version once a new one is publishing. A new year also restarts the major at 1 by convention, and nothing automates that — the failure message spells out both edits.
- why the duplication exists at all: `majorVersion` is a *template parameter*, resolved before any step runs, so no runtime value can feed it; and the props file only stamps local builds, since CI passes `/p:Version=...` which overrides it. That is exactly why drift here goes unnoticed — hence a guard rather than a single source.
- place it right after `checkout: self`, so it fails before anything expensive runs. Consumers: `Konfidence.BaseClasses`, `Producten/ProjectReferences`.

<h6>VerifyNoRedundantReferences.yml</h6>

- **steps template**, added 2026-08-21. Runs the `Konfidence.Project-References` tool over a solution and fails the build when it reports project or package references a project already gets another way — through a project it references, or through another package.
- installs/updates the tool itself (`dotnet tool update -g Konfidence.Project-References`, which installs when absent), so callers need no setup step of their own and every run uses the current version.
- parameters: `solution` (default empty) and `condition` (default `eq(variables['Build.Reason'], 'PullRequest')`).
- **pass `solution`** unless the repository's solution happens to be named after its folder. With no `--Solution` the tool looks for a solution named after the working folder, which on an agent is the checkout folder (typically `s`) — the run then reports `not found : solution file` and the gate would pass silently forever.
- **the tool always exits 0**: its entry point is an `async Task Main` with no return value, so it cannot fail a build by itself. It writes `redundant.txt` when it finds something and *deletes* a stale one when the solution is clean, so the file's existence is the signal — that is what the template turns into a failure, after printing the file's contents into the log.
- the report is written to the **current working directory**, not to `--BasePath`, hence the explicit `workingDirectory` in the template.
- package dependencies that arrive *through another package* are read from restore output, so place this after `dotnet restore`/`dotnet build` or the check is only partial (the tool reports how many projects it skipped).
- place it after the tests, so test results and coverage still publish when the gate fails, and before any pack/publish steps, so a finding stops artifacts being published. The default condition limits it to PR validation builds — the merge signal for repositories whose pipeline has `trigger: none`.

<h6>azure-pipelines-github.yml</h6>

- this repo's own pipeline for publishing itself to GitHub. Calls `PublishKonfidenceToGithub.yml` locally (same repo, no `resources: repositories` needed).


