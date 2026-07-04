Some yml templates I use on my azure devops server. Needed by some published projects. Keep in mind that currently you cannot reference github from azure devops <ins>server</ins>. Templates need to be in the same organisation when used by pipelines on devops server.

<h6>GenerateAndPublishCodeCoverage.yml</h6> 

- requires the reportgenerator package 'dotnet tool update -g dotnet-reportgenerator-globaltool'. Which should be run at some point in time before the unit tests are executed.
- tests should be run like 'dotnet test -c $(BuildConfiguration) --no-build --collect:"XPlat Code Coverage"'.
- coverage is picked up from `$(Agent.TempDirectory)/**/coverage.*`, i.e. wherever the VSTest coverlet data collector writes it — not from a project's own build output directory.
- runs with `condition: succeeded()` and `continueOnError: false`, so it's skipped (and the job fails) if an earlier step already failed.
- uses `PublishCodeCoverageResults@1`, not `@2`. `@1` is deprecated but `@2` **does not work on this server**: v2 spins up a separate `CoveragePublisher.Console.exe` that connects back to the Azure DevOps server itself over Basic/PAT auth, and that tool hard-refuses to do so over plain HTTP (`System.InvalidOperationException: Basic authentication requires a secure connection to the server`). This server is `http://tfs.konfidence.nl:8080` (no TLS), so v2 will always fail here until/unless the server gets HTTPS. Tried and reverted 2026-07-04 — don't re-attempt the `@2` upgrade without HTTPS on the server first.

<h6>PublishKonfidenceToGithub.yml</h6>

- pushes every branch (except 'master'/'main') to `https://github.com/a3helmich/Konfidence.$(Build.Repository.Name).git`.
- requires a `GitHubToken` variable (Variable Group `GitHubTokens`) authorized for the calling pipeline.
- calling pipeline's checkout step needs `fetchDepth: 0` (full history), since every branch is pushed.
- runs with `condition: succeededOrFailed()` and `continueOnError: true`, so it still runs (and won't fail the job) even if an earlier step in the pipeline failed.
- checks out each branch with `git checkout -B <branch> <origin/branch>` (force-reset), not plain `git checkout <branch>` — required because self-hosted agents reuse their working directory across runs, so a plain checkout of an already-existing local branch would freeze it at whatever commit it had from a *previous* run instead of advancing it to match origin's current state. Found 2026-07-03 after this silently stopped updating GitHub's `develop` branch for several runs.

<h6>azure-pipelines-github.yml</h6>

- this repo's own pipeline for publishing itself to GitHub. Calls `PublishKonfidenceToGithub.yml` locally (same repo, no `resources: repositories` needed).


