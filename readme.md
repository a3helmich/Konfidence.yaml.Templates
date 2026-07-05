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

<h6>azure-pipelines-github.yml</h6>

- this repo's own pipeline for publishing itself to GitHub. Calls `PublishKonfidenceToGithub.yml` locally (same repo, no `resources: repositories` needed).


