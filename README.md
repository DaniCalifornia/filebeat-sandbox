# Using a custom agent build for community integrations

This sandbox takes you through how to build a custom agent image for installing community integrations.

## Relevant docs:

- How to test Integration code change in Containerized agent (internal): https://datadoghq.atlassian.net/wiki/spaces/TS/pages/2378662760/How+to+test+Integration+code+change+in+Containerized+agent 
- Use community integrations (public): https://docs.datadoghq.com/agent/guide/use-community-integrations/?tab=docker#overview 


## Community integration: Filebeat

- Filebeat doc: https://docs.datadoghq.com/integrations/filebeat/#pagetitle


### Let's get started
First, create your dockerfile
```
FROM gcr.io/datadoghq/agent:latest
RUN agent integration install -r -t datadog-filebeat==1.3.0
```

latest version can be checked in the CHANGELOG.md in the integration repo: https://github.com/DataDog/integrations-extras/blob/master/filebeat/CHANGELOG.md 

build the docker image
`docker build -t <image_name> .`
docker doc: https://docs.docker.com/engine/reference/commandline/build/ 
```
> docker build -t fb-agent .
[+] Building 7.8s (6/6) FINISHED
 => [internal] load build definition from Dockerfile                                                0.0s
 => => transferring dockerfile: 136B                                                                0.0s
 => [internal] load .dockerignore                                                                   0.0s
 => => transferring context: 2B                                                                     0.0s
 => [internal] load metadata for gcr.io/datadoghq/agent:latest                                      0.0s
 => CACHED [1/2] FROM gcr.io/datadoghq/agent:latest                                                 0.0s
 => [2/2] RUN agent integration install -r -t datadog-filebeat==1.3.0                               7.4s
 => exporting to image                                                                              0.3s
 => => exporting layers                                                                             0.3s
 => => writing image sha256:1f5de89d77374b3b302a258a125b5689a9ea0af16e92378798ed1d853184d677        0.0s
 => => naming to docker.io/library/fb-agent                                                         0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
```

run `docker images` to see the image we just built
```
REPOSITORY                          TAG          IMAGE ID       CREATED         SIZE
fb-agent                            latest       1f5de89d7737   6 minutes ago   1.01GB
```

before you push the image to dockerhub, you might have to run `docker login`:
```
docker login -u “myusername” -p “mypassword”
```

we also have to tag the image:
```
docker tag fb-agent throatslasher/fb-agent:v1
```

and then push the docker image: `docker push {{username}}/{{imagename}}:{{version}}`
```
> docker push throatslasher/fb-agent:v1

The push refers to repository [docker.io/throatslasher/fb-agent]
ae2d8ca65199: Pushed
0c455dabfd19: Mounted from throatslasher/traefik_agent
v1: digest: sha256:e2bdedde7309378eba85801325527276e2cedd67fee834ce9b1f3785e4a45962 size: 741
```

you should be able to see your image pushed to your docker repo. here's mine: https://hub.docker.com/r/throatslasher/fb-agent 

now we can configure our helm chart to use our custom agent
you can find our default helm chart here: https://github.com/DataDog/helm-charts/blob/main/charts/datadog/values.yaml

you can find a minikube-safe `values.yaml` in this repo

create a file called `fb-values.yaml` with your helm config:
```
targetSystem: "linux"
datadog:
  apiKeyExistingSecret: datadog-agent
  logLevel: DEBUG
  kubelet:
    tlsVerify: false
  kubeStateMetricsEnabled: false
```

now to set the helm chart to use our custom agent image, we have to set the following:
```
agents:
  image: 
    name: traefik_agent  # image_name
    tag: version1  # image_tag
    repository: docker.io/throatslasher/traefik_agent  # registry/repository/image_name
    doNotCheckTag: true  # set to true to bypass helm chart's version requirements
```

when you deploy the agent with this config, the agent package will now include the filebeat check code to run:
- deploy with helm: https://docs.datadoghq.com/containers/kubernetes/installation/?tab=helm#minimum-agent-and-cluster-agent-versions
```
helm install datadog -f fb-values.yaml datadog/datadog
```

you should now have a datadog agent pod and cluster agent pod running

now we need a filebeat deployment. i copied this one from the elastic filebeat k8s repo: https://github.com/elastic/beats/blob/master/deploy/kubernetes/filebeat-kubernetes.yaml#L45-L67
and added the check config via annotations
you can try making it yourself or find it in this repo under `answers/filebeat.yaml`

create your filebeat deployment:
```
kubectl apply -f filebeat.yaml
```

and you should see a filebeat pod running as well. you can describe the pod to take a look at the annotations:
```
kubectl describe pod <filebeat_pod_name>
```
and get the agent status to see the filebeat check running:
```
kubectl exec -it <agent_pod_name> agent status
```

(there will be some errors cuz i can't figure out how to set the registry file path, sorry)