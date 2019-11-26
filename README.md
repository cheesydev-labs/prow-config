# my-prow-config

### Install Prow

To install Prow in a Kubernetes cluster, run `tackle`. You will be offered the
option to choose the `starter.yaml` from test-infra's upstream in github.

```
# cd test-infra
bazel run //prow/cmd/tackle # select cluster,
                            # choose starter.yaml,
                            # github api token from bot account,
                            # add webhook in your org/proj
```

### Update prow's plugins configurations:

Optionally, first check if configs are valid:
```
# from test-infra root dir
bazel run //prow/cmd/checkconfig -- \
  --plugin-config=/path/to/plugins.yaml \
  --config-path=/path/to/config.yaml
#
# it should output:
# { ... "func":"main.main","level":"info","msg":"checkconfig passes without any error!" ... }
```

Makefile needs to find your GKE cluster:
```
export CLUSTER=prow               #
export PROJECT=tekton-257312      # replace with your values
export ZONE=europe-west1-b        #
```

Apply configs from plugins.yaml and config.yaml:
```
# apply changes from the local plugins.yaml and config.yaml
make update-plugins
```

Now, PRs to the target projects will be labeled with a size tag, e.g.
`size/XS`.

### Configure plank

From [configure cloud storage](https://github.com/kubernetes/test-infra/blob/master/prow/getting_started_deploy.md#configure-cloud-storage)
to enable Pod Utilities for Prow jobs.

Cloud Storage (Google's s3) bucket names should be globally unique, so I'm
appending the GCP project id at the end of bucket name.

```
gcloud iam service-accounts create prow-gcs-publisher # step 1
identifier="$(  gcloud iam service-accounts list --filter 'name:prow-gcs-publisher' --format 'value(email)' )"

bucket_name=prow-artifacts-$PROJECT
gs=gs://$bucket_name

gsutil mb $gs # step 2
gsutil iam ch allUsers:objectViewer $gs # step 3
gsutil iam ch "serviceAccount:${identifier}:objectAdmin" $gs # step 4
gcloud iam service-accounts keys create --iam-account "${identifier}" service-account.json # step 5
kubectl -n test-pods create secret generic gcs-credentials --from-file=service-account.json # step 6
```

With the service account, the Cloud Storage bucket, and the secret credentials
for it, we can finally config plank in `config.yaml`. The image tags below
should be the same deployed initially by our `starter.yaml` file.

```
plank:
  default_decoration_config:
    utility_images: # using the tag we identified above
      clonerefs: "gcr.io/k8s-prow/clonerefs:v20191121-1ff2207db"
      initupload: "gcr.io/k8s-prow/initupload:v20191121-1ff2207db"
      entrypoint: "gcr.io/k8s-prow/entrypoint:v20191121-1ff2207db"
      sidecar: "gcr.io/k8s-prow/sidecar:v20191121-1ff2207db"
    gcs_configuration:
      bucket: prow-artifacts # the bucket we just made
      path_strategy: explicit
    gcs_credentials_secret: gcs-credentials # the secret we just made

postsubmits:
  ...

presubmits:
  ...
```

Running the optional check config was helpful at this time. It pointed me that
the repo with the postsubmits and presubmit jobs declared above, in
config.yaml, does not have the trigger plugin enabled.
```
# from test-infra root dir
bazel run //prow/cmd/checkconfig -- \
  --plugin-config=/path/to/my/plugins.yaml \
  --config-path=/path/to/my/config.yaml
#
# output:
# { ... "level":"warning","msg":"The `plank.allow_cancellations` setting is deprecated. It will be removed and set to always true in March 2020"}
# { ... "level":"warning","msg":"default_decoration_config will be deprecated on April 2020, and it will be replaced with default_decoration_configs['*']."}
# { ... "level":"warning","msg":"the following repos have jobs configured but do not have the trigger plugin enabled: repo: cheesy-labs/java-app"}
```

After adding `trigger` to plugins.yaml and running bazel checkconfig again,
I have an ok in the final outputline:
```
{"level":"info","msg":"checkconfig passes without any error!"}
```

Apply the latest changes from plugins.yaml and config.yaml to the cluster:
```
make update-plugins
make update-config
```

Now, the presubmit job will run when a PR is open, and a status message will be
posted to the PR. The postsubmit will run after a change to the repo, a push or
a merge.

