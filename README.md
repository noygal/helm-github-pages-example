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

```bash
helm package charts/* -d docs
# Replace --url parameter with your repository path
helm repo index --url https://noygal.github.io/helm-github-pages-example/ --merge docs/index.yaml docs
```

## Step 4 - serving repository via GitHub Pages

Helm docs:
> Because a chart repository can be any HTTP server that can serve YAML and tar files and can answer GET requests, you have a plethora of options when it comes down to hosting your own chart repository. For example, you can use a Google Cloud Storage (GCS) bucket, Amazon S3 bucket, GitHub Pages, or even create your own web server.

Go to your github repository _Setting_ tab and scroll down to the _GitHub Pages_ section, on _Source_ select branch `main` and folder `/docs`, lastly click on th _save_ button.

Commit the files and folders we've created and push it to GitHub.
