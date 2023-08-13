+++
title = "Monorepo NuGet Package Updates with GitHub Actions"
description = "Learn how to automate NuGet package updates in a monorepo by using GitHub Actions to run a script that updates all packages and opens a single pull request."
date = 2023-06-13

[taxonomies]
categories = ["Tech"]
tags = ["dotnet", "csharp", "bash", "github actions"]

[extra]
toc = true
cc_license = true
+++

{% important(header="Important") %}
The following is out-of-date.

A much simpler solution is to use the new dependabot beta feature called
[groups](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#groups)
and nuget [central package
management](https://learn.microsoft.com/en-us/nuget/consume-packages/Central-Package-Management).

{% end %}

# The problem with dependabot

I experimented with Dependabot in a smaller Github repository and
wanted to add it to a larger monorepo. The problem is dependabot creates a PR
for each dependency upgrade. I even used the `open-pull-requests-limit: 10`
setting, but with so many projects it opened 100 PRs before I could stop it.

Cleaning up all the PRs wasn't so hard with the help of ChatGPT and the [Github
CLI](https://cli.github.com/):

```bash
label="dependencies"
repo="your-org/your-repo"
for pr in $(gh pr list -R $repo --label "$label" --json number -q '.[].number')
do
  gh pr close $pr -R $repo
done
```

The above is a bash script. You can make it a one-liner by replacing the
newlines with `;`. No doubt popular LLMs these days, such as ChatGPT, can
convert it to PowerShell if you like.

Cleanup all the created branches:

```sh
$ git fetch -all
$ git branch -r | grep 'origin/dependabot' \
  | sed 's/origin\///' \
  | xargs -I {} git push origin --delete {}
```

# The solution

The solution is to use GitHub actions to run a script that updates all the nuget
packages in the monorepo and opens a single PR. The script uses
[dotnet-outdated](https://github.com/dotnet-outdated/dotnet-outdated) tool to
upgrade all packages in each project.

`your-repo/upgrade-packages.sh`:

```bash
#!/usr/bin/env bash
set -e

if ! command -v dotnet-outdated &>/dev/null; then
    echo "Error: dotnet-outdated not found" >&2
    echo "Run:" >&2
    echo "dotnet tool install --global dotnet-outdated-tool" >&2
    exit 1
fi

cleanup() {
    echo "Cleaning up before exiting the script"
    # Remove the added "NoWarn" property
    find . -type f -iname '*.csproj' | xargs -I{} sed -i '/<NoWarn>NU1605<\/NoWarn>/d' {}
}

# Trap the EXIT signal and call the cleanup function.
# Sort of like a try/finally block in C#
trap cleanup EXIT

# Disable the "Detected package downgrade" warning as error because we want to
# update projects without having to do it in depth first traversal dependency
# order.
find . -type f -iname '*.csproj' | xargs -I{} sed -i '/<PropertyGroup>/a<NoWarn>NU1605<\/NoWarn>' {}

find ./Libraries ./Services -type f -iname '*.csproj' \
    -not -path './Libraries/SomeSqlProject/*' \
    -not -path './Services/SomeLegacyService/*' \
    -not -path './Services/SomeLegacyService2/*' |
    xargs -I{} sh -c 'echo "Updating {}"; \
    dotnet outdated {} --upgrade'
```

The `NU1605` "Detected package downgrade" warning is treated as an error by
default in newer dotnet projects. This is a problem because I rather not have to
do a depth first traversal in dependency order for project updates. Fortunately,
it can be overridden by adding a `NoWarn` property to the project file.

Some of the following takes ideas from this [blog post](https://www.oddbird.net/2022/06/01/dependabot-single-pull-request/).

`your-repo/.github/workflows/upgrade-dependencies.yaml`:

```yaml
---
name: Upgrade nuget dependencies

on:
  workflow_dispatch: # Allow running on-demand
  schedule:
    # Runs every Monday at 8:00 UTC (4:00 Eastern)
    - cron: "0 8 * * 1"

jobs:
  upgrade-nuget-dependencies:
    name: Upgrade & Open Pull Request
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "7.0.x"

      - name: Install dotnet-outdated
        run: dotnet tool install --global dotnet-outdated-tool

      - name: Run upgrade-packages.sh script
        run: |
          chmod +x ./upgrade-packages.sh
          export PATH="$HOME/.dotnet/tools:$PATH"
          ./upgrade-packages.sh

      - name: Create Pull Request
        id: create-pull-request
        uses: peter-evans/create-pull-request@v5
        with:
          commit-message: Automated dependency upgrade
          committer: GitHub <noreply@github.com>
          author: ${{ github.actor }} <${{ github.actor }}@users.noreply.github.com>
          branch: feature/auto-dependency-upgrades
          delete-branch: true
          title: "Update Nuget dependencies"
          body: |
            Update report log:
            https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}
          labels: |
            dependencies
```

>Note: If you want pull requests created by this action to trigger an on: push
>or on: pull_request workflow then you cannot use the default GITHUB_TOKEN. See
>the
>[documentation](https://github.com/peter-evans/create-pull-request/blob/main/docs/concepts-guidelines.md#triggering-further-workflow-runs)
>here for workarounds. — <cite>[^1]</cite>

# Run GitHub action locally

Install:

- [act](https://github.com/nektos/act) to run GitHub actions locally.
- GitHub CLI: https://cli.github.com/

Now you can run the `upgrade-nuget-dependencies` GitHub action locally:

```sh
act -j upgrade-nuget-dependencies -s GITHUB_TOKEN="$(gh auth token)"
```

# Run upgrade-packages.sh locally

```sh
chmod u+x ./upgrade-packages.sh
dotnet tool install -g dotnet-outdated`
./upgrade-packages.sh
```

---

[^1]: <https://github.com/peter-evans/create-pull-request#action-inputs>
