# Argo
- Implemented as a K8s Custom Resource Defintion (CRD)
    - K8s spec called `Workflow`

## Components
- Workflow controller
  - For running workflows
- Argo server
  - For providing a UI and API

## Usage
```sh
# Create a new Argo workflow
argo submit workflow-spec.yaml

# List the current workflows
argo list 

# Get info about a specific workflow
argo get workflow-spec-xx

# Get the logs from a workflow
argo logs workflow-spec-xx
```
- Can also just use `kubectl` to run Argo workflows, but using Argo CLI (`argo`) provides syntax checking and customized output

## Workflow
- A workflow is defined as a K8s resource
  - Each workflow consists of >=1 template with each being an entrypoint
- Each template can be one of many types: [Workflow templates](https://argo-workflows.readthedocs.io/en/latest/workflow-templates/)

### Workflow Template 
- Workflow specs are composed of a set of Argo templates
- `container` section of the `Workflow` spec will accept the same options as the `container` section of a `pod` spec (e.g.: environment variables, secrets, volume mounts)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
    generateName: workflow-spec # name of the workflow spec
spec:
    entrypoint: template-1 # invoke the template-1 template
    arguments: # invoke the template with parameters
        parameters:
        - name: message
          value: hello world
        - name: log-level
          value: INFO
    templates:
        - name: template-1
          container:
            image: busybox:latest
            command:
                - echo
                - "{{workflow.parameters.message}}"
        - name: template-2
          container:
            image: container-2
            env:
            - name: LOG_LEVEL
              value: "{{workflow.parameters.log-level}}"
        - name: hello-hello-hello
          steps:
          - - name: hello-1 # hello-1 will be ran before the following steps
              template: whalesay
              arguments:
                parameters:
                - name: message
                  value: "hello1"
          - - name: hello-2 # hello-2 will run after hello-1 as indicated by double dash
              template: whalesay
              arguments:
                parameters:
                - name: message
                  value: "hello2"
            - name: hello-2b # hello-2b will run in parallel with hello-2 as per single dash
              template: whalesay
              arguments:
                parameters:
                - name: message
                  value: "hello2b"
        - name: whalesay
          inputs:
            parameters:
            - name: message
          container:
            image: docker/whalesay
            command: [cowsay]
            args: ["{{inputs.parameters.message"}}]
    
```
**Step templates**
- Step templates use `steps` prefix to refer to another step
    - e.g.: `{{steps.influx.ip}}`

**Parameters**
- Values in `spec.arguments.parameters` are globally scoped and can be accessed via `{{workflow.parameters.parameter_name}}`
- Useful to pass info to multiple steps in a workflow

**Directed-Acyclic Graph (DAG)**
- Instead of specifying sequences of steps, can also define a workflow as a directed-acyclic graph (DAG) by specifying the dependencies of each task
- DAGs can be simpler to maintain the complex workflows and allow for maximum parallelism when running tasks

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
  - name: diamond
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```
- Following will result in:
    1. Step A
    -> Step A Completed
    2. Step B and C in parallel
    -> Steps B and C Completed
    3. Step D
- By default, DAGs fail fast
    - 1 task fails, no new tasks will be scheduled
    - Once all running tasks are completed, the DAG will be marked as failed
    - If `failFast` sets to false, all branches will run to completion even with failures in other branches
- DAG templates use the `tasks` prefix to refer to another task
    - e.g.: `{{tasks.generate-artifact.outputs.artifacts.hello-art}}`

## Artifacts
- When running workflows, can include steps that generate or consume artifacts (e.g.: output artifacts of a step used as input artifacts to a subsequent steps)
- See [Argo Artifacts](https://argo-workflows.readthedocs.io/en/latest/walk-through/artifacts/) for more info
- Will need to configure an artifact repository to run Argo workflows that use artifacts
    - Any S3 compatible artifact repository like `MinIO`
    - See [Configure Artifactory](https://argo-workflows.readthedocs.io/en/latest/configure-artifact-repository/) for more info

### Hardwired Artifacts
- Can use any container iamge to generate any kind of artifact
    - Commonly used: `git` and `S3` artifacts
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hardwired-artifact-
spec:
  entrypoint: hardwired-artifact
  templates:
  - name: hardwired-artifact
    inputs:
      artifacts:
      # Check out the main branch of the argo repo and place it at /src
      # revision can be anything that git checkout accepts: branch, commit, tag, etc.
      - name: argo-source
        path: /src
        git:
          repo: https://github.com/argoproj/argo-workflows.git
          revision: "main"
      # Download kubectl 1.8.0 and place it at /bin/kubectl
      - name: kubectl
        path: /bin/kubectl
        mode: 0755
        http:
          url: https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl
      # Copy an s3 compatible artifact repository bucket (such as AWS, GCS and MinIO) and place it at /s3
      - name: objects
        path: /s3
        s3:
          endpoint: storage.googleapis.com
          bucket: my-bucket-name
          key: path/in/bucket
          accessKeySecret:
            name: my-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-s3-credentials
            key: secretKey
    container:
      image: debian
      command: [sh, -c]
      args: ["ls -l /src /bin/kubectl /s3"]
```

## Scripts
- Can also create template that executes script
    - In Argo terms, its called `here-script` (or `here document`) in the `Workflow` spec

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scripts-bash-
spec:
  entrypoint: bash-script-example
  templates:
  - name: bash-script-example
    steps:
    - - name: generate
        template: gen-random-int-bash
    - - name: print
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "{{steps.generate.outputs.result}}"  # The result of the here-script

  - name: gen-random-int-bash
    script:
      image: debian:9.4
      command: [bash]
      source: |                                         # Contents of the here-script
        cat /dev/urandom | od -N2 -An -i | awk -v f=1 -v r=100 '{printf "%i\n", f + r * $1 / 65536}'

  - name: gen-random-int-python
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        i = random.randint(1, 100)
        print(i)

  - name: gen-random-int-javascript
    script:
      image: node:9.1-alpine
      command: [node]
      source: |
        var rand = Math.floor(Math.random() * 100);
        console.log(rand);

  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo result was: {{inputs.parameters.message}}"]
```
- `script` keyword: allows specification of the script body using `source` tag
- `script` also then assigns the standard output of running the script to `result` which is a special output parameter that we can later use in the rest of the `Workflow` spec

## Output parameters
- `outputs` parameter allows us to store the result of a step
- `result` is a special output parameter that we can later use in the rest of the `Workflow` spec
    - Not just from a `script`
- **Works just like `script result` except this value of `output` is set to the contents of a generated file rather than the contents of `stdout`**
- See [`output` parameters](https://argo-workflows.readthedocs.io/en/latest/walk-through/output-parameters/) for more info

## Manage K8s Resources
- Can manage k8s resources through Argo workflows
- `resource` template allows any `kubectl` actions (e.g.: `create, delete, apply, patch`)
- See [Kubernetes Resources](https://argo-workflows.readthedocs.io/en/latest/walk-through/kubernetes-resources/) for more info

## Other features
### Commonly used
- Accessing `Secrets`: https://argo-workflows.readthedocs.io/en/latest/walk-through/secrets/
- Declaring `Volume` and Using: https://argo-workflows.readthedocs.io/en/latest/walk-through/volumes/
- Retry failed / error steps: https://argo-workflows.readthedocs.io/en/latest/walk-through/retrying-failed-or-errored-steps/
- Exit handlers: https://argo-workflows.readthedocs.io/en/latest/walk-through/exit-handlers/
    - Template that always executes, irrespective of success / failure at the end of the workflow
- Timeouts: https://argo-workflows.readthedocs.io/en/latest/walk-through/timeouts/
- Daemon containers: https://argo-workflows.readthedocs.io/en/latest/walk-through/daemon-containers/
    - Argo workflows can start `dameon containers` while the workflow continues execution
    - Daemons will automatically get destroyed when the workflow exits in the template scope where the daemon was invoked
    - Benefit over `sidecars` is how dameons can persist across multiple steps / entire workflow
- Sidecars: https://argo-workflows.readthedocs.io/en/latest/walk-through/sidecars/

### Misc
- Running `Loops` to iterate inputs: https://argo-workflows.readthedocs.io/en/latest/walk-through/loops/
- Conditional execution: https://argo-workflows.readthedocs.io/en/latest/walk-through/conditionals/
- Recursively invoke templates: https://argo-workflows.readthedocs.io/en/latest/walk-through/recursion/

# Resources
- [Argo github](https://github.com/akuity/awesome-argo?tab=readme-ov-file)
- [Argo hands-on course](https://killercoda.com/argoproj/course/argo-workflows/)
