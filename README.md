### Using a custom agent build for community integrations

Filebeat doc: https://docs.datadoghq.com/integrations/filebeat/#pagetitle

wiki: https://datadoghq.atlassian.net/wiki/spaces/TS/pages/2378662760/How+to+test+Integration+code+change+in+Containerized+agent 

use community integrations: https://docs.datadoghq.com/agent/guide/use-community-integrations/?tab=docker#overview 

create dockerfile
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