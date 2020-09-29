# Tekton inline scanning with Sysdig

This repository contains instructions and examples of how to use Sysdig inline scanning to detect vulnerabilities and misconfiguration in a Tekton CI/CD pipeline, using the **alpha** and **beta** Tekton API.

They have been tested and can be used for **vanilla Kubernetes** as well as on **OpenShift**, as Sysdig inline scanning doesn't require a privileged container.

For more information about Sysdig, visit https://sysdig.com.

## Inline Image Scanning

Sysdig inline image sanning can be used in Tekton, without requiring a docker-in-docker setup, mounting the Docker socket or privileged access.

You have to define a scanning step after building an image in a Tekton task, so it then scans a local folder with the image contents in OCI format, without pushing it to a registry or sending the image contents to Sysdig backend. This is a brief code extract for how to do it (for Tekton v1beta1 API):

```yaml
  - name: scan
    image: sysdiglabs/sysdig-inline-scan:latest
    args:
      - -U
      - $(workspaces.source.path)/$(params.CONTEXT)/image-digest
      - $(outputs.resources.builtImage.url)
    env:
      - name: SYSDIG_API_TOKEN
        valueFrom:
          secretKeyRef:
            name: sysdig-secrets
            key: sysdig-secure-api-key
```

You'll need to add a secret for your Sysdig Secure API token, and reference it in the service account definition that executes the pipeline, as you can see in the [full pipeline example for beta Tekton API](./beta/tekton-inline-scan-beta.yaml).

## Build-scan-push Tekton Task

The example pipeline describen in the [official Tekton tutorial](https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md) uses `kaniko` to build and push the image in a single step. 

To have a task that builds the image, scans it locally, and only pushes it to the registry if it is in compliance with scanning policies, we have to tell `kaniko` in the first step to not push the image, and add a last additional step to push it using `skopeo` (as `kaniko` can't push an image without rebuilding it, which would waste resources).

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  params:
    - name: pathToDockerFile
      type: string
      description: The path to the dockerfile to build
      default: $(resources.inputs.docker-source.path)/Dockerfile
    - name: pathToContext
      type: string
      description: |
        The build context used by Kaniko
        (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
      default: $(resources.inputs.docker-source.path)
  resources:
    inputs:
      - name: docker-source
        type: git
    outputs:
      - name: builtImage
        type: image
  steps:
    - name: build
      image: gcr.io/kaniko-project/executor:v0.16.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.pathToDockerFile)
        - --destination=$(resources.outputs.builtImage.url)
        - --context=$(params.pathToContext)
        - --oci-layout-path=/workspace/oci
        - --no-push

    - name: scan
      image: sysdiglabs/sysdig-inline-scan:latest
      args:
        - -U
        - /workspace/oci
        - $(outputs.resources.builtImage.url)
      env:
        - name: SYSDIG_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: sysdig-secrets
              key: sysdig-secure-api-key

    - name: push
      image: quay.io/skopeo/stable
      command:
        - /usr/bin/skopeo
      args:
        - --insecure-policy      
        - --dest-authfile
        - /tekton/home/.docker/config.json
        - copy
        - oci:/workspace/oci/
        - docker://$(outputs.resources.builtImage.url)
```

## Full pipeline examples with inline scanning for alpha and beta Tekton API

You can find full pipelines examples for both **alpha** and **beta** Tekton API in the following files of this repo:

* [pipeline-example-alpha.yaml](./alpha/tekton-inline-scan-alpha.yaml).
* [pipeline-example-beta.yaml](./beta/tekton-inline-scan-beta.yaml).


