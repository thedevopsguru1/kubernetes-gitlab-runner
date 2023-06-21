# kubernetes-gitlab-runner
Prerequisites:
gitlab-runner helm chart
access to the cluster and namespace


Download the gitlab-runner helm chart. Use the values.yaml file provided from there for this document.


The cluster doesn't allow creating ClusterRoles so we have to rely on the built-in admin Role. The Gitlab runner chart's rbac support is all or nothing so we can't specify the Role to use with a generated Service Account. That means we have to:

Disable the gitlab runner chart's Service Account creation and specify one of our own
Create said Service Account in the ${PROJECT}-tooling Namespace
Bind the ServiceAccount to the admin ClusterRole in each namespace the runner will deploy to
To separate resource quotas, we typically create 3 Namespaces per project:
${PROJECT}-tooling which holds all the gitlab runner containers
${PROJECT}-review which holds all the Review Apps deployments
${PROJECT} which holds the "production" or stable release from master branch. This namespace is usually on the production cluster with it's own runner in the ${PROJECT}-tooling namespace
Using Foo as an example project name, the commands would look like this:
```
kubectl create serviceaccount gitlab-runner --namespace ${PROJECT}-tooling
```
```
kubectl create rolebinding gitlab-runner:admin --clusterrole=admin --serviceaccount=${PROJECT}-tooling:gitlab-runner --namespace ${PROJECT}-tooling
```
```
kubectl create rolebinding gitlab-runner:admin --clusterrole=admin --serviceaccount=${PROJECT}-tooling:gitlab-runner --namespace ${PROJECT}-review
```

Additionally, the gitlab runner chart doesn't support caching to PVCs out of the box so we'll need to manually create a PVC to be used for caching. The important thing here is that the accessModes include ReadWriteMany otherwise concurrent job pods will get stuck waiting for the volume to free up and timeout.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-gitlab-runner-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```
To find the gitlab-runner registration token:

Go to the project’s Settings > CI/CD and expand the Runners section.
Note the URL and token.


Finally, configure the runner to use the ServiceAccount and cache PVC via values.yaml:
```
checkInterval: 5
concurrent: 10
gitlabUrl: https://gitlaburlhere/
rbac:
  create: false
  podSecurityPolicy:
    enabled: false
  serviceAccountName: gitlab-runner
runnerRegistrationToken: "XXXXXX"
runners:
  config: |
    [[runners]]
      cache_dir = "/tmp/gitlab/cache"
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:20.04"
        cpu_limit = "128m"
        cpu_limit_overwrite_max_allowed = "4"
        cpu_request = "128m"
        cpu_request_overwrite_max_allowed = "4"
        helper_cpu_limit = "128m"
        helper_cpu_limit_overwrite_max_allowed = "4"
        helper_cpu_request = "128m"
        helper_cpu_request_overwrite_max_allowed= "4"
        memory_limit = "128Mi"
        memory_limit_overwrite_max_allowed = "4G"
        memory_request = '128Mi'
        memory_request_overwrite_max_allowed = "4G"
        helper_memory_limit = "128Mi"
        helper_memory_limit_overwrite_max_allowed = "4G"
        helper_memory_request = "128Mi"
        helper_memory_request_overwrite_max_allowed = "4G"
      [runners.kubernetes.volumes]
        [[runners.kubernetes.volumes.pvc]]
          name = '{{ include "gitlab-runner.fullname" . }}-cache'
          mount_path = "/tmp/gitlab/cache"
  serviceAccountName: gitlab-runner
  tags: mariner
unregisterRunners: true
```
After you have edited the values.yaml file to replicate what is above, deploy the chart to the cluster:

Deploy the gitlab-runner to the Cluster Expand source
## Troubleshooting:

### What to do when gitlab runners are not using the correct service account
 
If you see an error such as: `Error: UPGRADE FAILED: query: failed to query with labels: secrets is forbidden: User "system:serviceaccount:darknet-tooling:default" cannot list resource "secrets" in API group "" in the namespace "darknet-prod"`, add the following to you values.yaml file:

Update values.yaml to use the proper service account Expand source
This adds the KUBERNETES_SERVICE_ACCOUNT environment variable to each gitlab-runner to ensure the correct service account is used for each runner.

Now you can upgrade your helm chart by issuing the following:

Upgrade the helm chart
