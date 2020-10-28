# Step-by-step guide for github pages based helm repository

This helm charts repository is served at: https://noygal.github.io/helm-github-pages-example

## Perquisites

[Helm](https://helm.sh/) should be install for the helm operation, if you also want to deploy the charts you will need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and a cluster available.

## Step 1 - open git repository

Open a git repo and add `.gitignore` file ([gitignore.io](https://www.toptal.com/developers/gitignore) recommended), make sure to add the next line it would ignore chart dependencies:

```bash
**/charts/*.tgz
```

## Step 2 - create an helm chart

We will use `helm` to create a basic chart, it would be sufficient for the propose of this guide.

```bash
mkdir charts
helm create charts/first-chart
```

## Step 3 - create repository index file

Helm docs:
> A chart repository is an HTTP server that houses an index.yaml file and optionally some packaged charts. When you're ready to share your charts, the preferred way to do so is by uploading them to a chart repository.

The packages and `index.yaml` are very easy to generate.

```bash
helm package charts/first-chart -d docs
# Replace --url parameter with your repository path
helm repo index --url https://noygal.github.io/helm-github-pages-example/ --merge docs/index.yaml docs
```

## Step 4 - serving repository via GitHub Pages

Helm docs:
> Because a chart repository can be any HTTP server that can serve YAML and tar files and can answer GET requests, you have a plethora of options when it comes down to hosting your own chart repository. For example, you can use a Google Cloud Storage (GCS) bucket, Amazon S3 bucket, GitHub Pages, or even create your own web server.

Turning on the __GitHub Pages__ is very easy, from the repository page, go to your github repository _Setting_ tab and scroll down to the _GitHub Pages_ section, on _Source_ select branch `main` and folder `/docs`, lastly click on th _save_ button.

Commit the files and folders we've created and push it to GitHub.

## Step 5 - add your repository to helm

```bash
# Replace the url with your repository folder
helm repo add helm-github-pages-example https://noygal.github.io/helm-github-pages-example
helm repo list
helm search repo helm-github-pages-example
```

You can also install the chart to your cluster:

```bash
helm install first-chart helm-github-pages-example/first-chart
```

## Step 6 - chart versions

If we want to publish a new chart we just need to change the chart version on `charts/first-chart/Chart.yaml`, build the index again (step 3). __Notice__: packaging the chart without version change would OVERWRITE the existing package, always increase version number upon chart changes.

```bash
helm package charts/first-chart -d docs
# Replace --url parameter with your repository path
helm repo index --url https://noygal.github.io/helm-github-pages-example/ --merge docs/index.yaml docs
```

Commit the new/changed files and push them to github.

You'll need to update the helm repo to view the new chart version.

```bash
helm repo update
helm search repo helm-github-pages-example --versions
```

If you find that the refresh doesn't work, use this [script](https://gist.github.com/naotookuda/db74069729ba0b7740b658b689cba65b) to manually delete the helm cache folders.

## Step 7 - chart dependencies

I think that most of the people who would serve there helm repository would gain from using the dependency features of helm, let's start by creating a second chart.

```bash
helm create charts/second-chart
```

Add this section to the end of the file: `charts/second-chart/Chart.yaml`, it would state that the `second-chart` is dependent on the `first-chart`.

```yaml
dependencies:
  - name: first-chart
    version: 0.1.1
    repository: https://noygal.github.io/helm-github-pages-example
```

We will need to build the dependencies for the chart, notice that `Chart.lock` been add to the chart main folder, and the packaged chart to the charts folder (should be excluded by the git ignore file). If you want to update the dependencies, you'll need to use the `update` command, `build` will install the version on the lockfile.

```bash
helm dependency build charts/second-chart
```

Package the `second-cart`, and update the index as we did in the previouse steps.

```bash
helm package charts/second-chart -d docs
helm repo index --url https://noygal.github.io/helm-github-pages-example --merge docs/index.yaml docs
```

Commit and push, and then update the repo.

```bash
helm repo update
helm search repo helm-github-pages-example
```

If you install the `second-chart` chart you'll notice two pods deploying, one for each chart.

For those who want to tinker it you can change the values of the `first-chart` by adding something like this to the `second-chart` `values.yaml` file.

```yaml
first-chart:
  image:
    repository: traefik
    pullPolicy: IfNotPresent
    tag: latest
```

## Step 8 - github actions - testing

Helm supports a few _Github Action_ operators, for adding a basic linting and testing you can the [helm-chart-testing](https://github.com/marketplace/actions/helm-chart-testing) action.

Add the `.github/workflows/main.yml` file to the repository and push to github.

```yaml
name: Lint and Test Charts

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  # Reference https://github.com/marketplace/actions/helm-chart-testing
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Fetch history
        run: git fetch --prune --unshallow

      - name: Run chart-testing (lint)
        id: lint
        uses: helm/chart-testing-action@v1.1.0
        with:
          command: lint
          config: .github/config/cf.yaml

      - name: Create kind cluster
        uses: helm/kind-action@v1.0.0
        # Only build a kind cluster if there are chart changes to test.
        if: steps.lint.outputs.changed == 'true'

      - name: Run chart-testing (install)
        uses: helm/chart-testing-action@v1.1.0
        with:
          command: install
          config: .github/config/cf.yaml
```

Also due to the changing of `master` branch to `main` we will need to another configuration file:

`.github/config/cf.yaml`

```yaml
target-branch: main
```

## Step 9 - github actions - release

Another useful _Github Action_ operator is the [helm-chart-releaser](https://github.com/marketplace/actions/helm-chart-releaser), it's an official `helm` release tool that utilize github repository _Releases_ feature and _Github Pages_ manage git based helm chart repository.

The action is configured to use _Github Pages_ branch based serving on the default `gh-pages`, it would also merge the online version of the current repository file, so the first thing we will need to change the serving method we set on `Step 4`, go to the _Github Pages_ and change it to branch `gh-pages` and the folder `/(root)`.

__NOTICE:__ If you follow the `Step 7 - chart dependencies` the build would fail as the packaging of the `second-chart` would require the `first-chart` artifact from the registry, the simplest way would be to remove the dependency, commit and add it again.

Add the action file `.github/workflows/release.yaml`

```yaml
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.0.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

Notice that after running the action the first time you we have `index.yaml` on the root of the `gh-pages` folder, and new release artifacts.

## Step 10 - html file

As a cherry on top we'll add a simple UI to our repository, _Github Pages_ had a build in support for markdown when you enable "themes" section.

The idea is simple, get a template file and use the `index.yaml` as the value file. Check out the `.github/workflows/update-ui.yaml` file for the workflow implementation and `.github/templates/index.hbs` implementation, the solution is NodeJS oriented, but this can be also be achieved by `bash`, `go`, `python` or any other platform.
