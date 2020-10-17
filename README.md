# Step-by-step guide for github pages based helm repository

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

## Step 8 - github actions ?

## Step 9 - html file
