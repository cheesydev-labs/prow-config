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
export PROJECT=my-project-123456  # replace with your values
export ZONE=europe-west1-b        #
```

Apply configs from plugins.yaml and config.yaml:
```
# apply changes from the local plugins.yaml and config.yaml
make update-plugins
```

Now, PRs to the target projects will be labeled with a size tag, e.g.
`size/XS`.
