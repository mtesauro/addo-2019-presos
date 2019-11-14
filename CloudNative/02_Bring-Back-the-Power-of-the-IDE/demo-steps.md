# Bring Back the Power of the IDE

This repo is based on the talk for All Day Devops 2019:
https://www.alldaydevops.com/addo-speakers/jeff-knurek

The slides are included here in [pdf](slides.pdf), and below are the details/steps to reproduce the demos

## Steps taken to reproduce demo

### Pre-setup (getting code & creating directories):
* `mkdir -p ~/code`
* `mkdir -p ~/.armador`
* `cd ~/code`
* `git clone https://github.com/travelaudience/armador.git`
* `git clone https://github.com/dockersamples/example-voting-app.git`
* `cp -r armador/docs/example/example-app-charts/vote .`

### Armador (creating the env)

* In order to build the Armador binary, requires Golang 1.12+
    * a welcome PR is a change to the CI steps to add builds to the release page ;)
* I used `minikube` in the demo. Any K8s cluster should work, but if you choose minikube, the install steps are here: https://kubernetes.io/docs/tasks/tools/install-minikube/
* Helm package manager is required for Armador. Install steps are here: https://github.com/helm/helm#install
    * during the demo, only Helm v2 is supported

-----
* installing amrador:
    * `cd armador`
    * `go build -o $GOPATH/bin/armador`
* preparing cluster:
    * `minikube start --kubernetes-version=1.15.2` _(at the time of the demo, the example is not compatible with K8s v1.16+)_
    * `helm init`
* setup the app to be installed:
    * `cp ~/armador/docs/example/global-config.yaml ~/.armador.yaml`
    * `vi ~/.armador.yaml`
        * change `cluster` section to point your cluster, in the demo this was:
```
cluster:
  minikube:
    contextname: minikube
```
* create the env:
    * `cd ~/code/vote`
    * `armador create jeff`
* `kubectl get pods` _(armador changes your context to the namespace you created)_
* `minikube service -n jeff vote-jeff-vote` this exposes the service and loads the page in the browser (even though the context is in the `jeff` namespace, the `minikube` command needs that to be explicit)

### Ksync

This follows closely with the doc here: https://github.com/travelaudience/armador/blob/master/docs/debug-in-k8s/Ksync.md
And the install of ksync should be followed first: https://github.com/vapor-ware/ksync#installation

* You need to run `ksync watch` in a seperate terminal session
* `mkdir -p ~/ksync/` (I find it useful to have a distinct place to keep the synced folders. This removes the "easy" hot-reload ability, but it keeps the sync process cleaner without unneeded file copying)
* `kubectl get pods -n jeff --show-labels` getting the `app=` label of the pod(s) you want to sync
    * NOTE: there's another step that requires some deeper knowledge about the container that you want to sync. In the example I chose the vote app, which has:
    ```
    # Copy our code from the current folder to /app inside the container
    ADD . /app
    ```
    so I know that the files to change are in the `/app` path on the image. This then becomes the path that I want to have synced locally.
* `ksync create --selector=app=vote-jeff-vote --name=vote-jeff-vote -n jeff ~/ksync/vote-jeff-vote /app`
* `vi ~/code/example-voting-app/vote/app.py` _(make a change from "Cats" to "Bunnies")_
* `cp ~/code/example-voting-app/vote/app.py ~/ksync/vote-jeff-vote`
* you should now see the changes being applied in the `ksync watch` termnial, and refreshing the page should show "Bunnies"

### Telepresence

Again, this section follows closely with the docs here: https://github.com/travelaudience/armador/blob/master/docs/debug-in-k8s/Telepresence.md
And installing should follow the official docs: https://www.telepresence.io/reference/install

* I used IntelliJ and imported the code to the `worker` app: `~/code/example-voting-app/worker`
* Add a new "Configuration" for the application, and provide the main class as `worker.Worker`
* `telepresence --namespace jeff --swap-deployment worker-jeff-worker --run-shell` connect cluster with local env
* In IntelliJ
    * set breakpoint in the Worker (maybe at lines 14 & 20)
    * click "Debug"
    * if all goes well, line 14 will hit the breakpoint
    * if you trigger a new vote in the frontend, then line 20 should hit when the worker detects the new vote

### Skaffold

As mentioned, getting `skaffold`/`Cloud Code` to work for the demo wasn't easy. One of the important steps that is missing from their documentation, is that using an app that is `Helm` based doesn't allow for the auto-configure. And maybe that's where things went wrong. What I should say, is that the syncing with `skaffold` works fine, but just not getting breakpoints in the Java app. Here is the `skaffold.yaml` file that was used:

```yaml
apiVersion: skaffold/v1beta15
kind: Config
build:
  tagPolicy:
    sha256: {}
  artifacts:
    - image: local-java-worker
      docker:
        dockerfile: Dockerfile.j # because the defaul image uses .Net, and we want to debug Java
deploy:
  helm:
    releases:
      - name: worker-jeff
        chartPath: helm
        values:
          image: local-java-worker # using this that matches build.artifacts.image allows for syncing the docker image that gets created
        setValues:
          pullPolicy: IfNotPresent # overrides the setting that uses local images in minikube
profiles:
  - build:
      local:
        push: false
        useBuildkit: false
        useDockerCLI: true
    deploy: {}
    name: Demo
```
