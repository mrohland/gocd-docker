# go.cd based CI installation

Docker go.cd images to execute other docker container for testing purposes on the agent.

## Requirements
* plugins must be upgradeable independently from server or data
* data must be kept between server upgrades
* agents must have native docker support
* images are pulled from local registry

## HowTo

#### Create local registry to pull & push images for testing via:
* create self signed cert
```
openssl req \
    -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
    -x509 -days 365 -out certs/domain.crt
```
* in case there is no DNS add alternative IP in openssl.cnf(under /System/Library/OpenSSL/openssl.cnf on MacOS) in v3_ca section:
```
[ v3_ca ]
subjectAltName = IP:<yourIP>
```
* start docker registry
```
 docker run -d -p 5000:5000 \
    -v `pwd`/certs:/certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry \
    -e SETTINGS_FLAVOR=dev \
    -v /tmp/docker/registry:/tmp/registry-dev \
    --restart=always --name registry \
    registry:2
```

#### Build a plugins image to contain plugins
Idea: PlugIns should be able to get added & updated independently from data and server
```
docker build -t mrohland/gocd-plugins:v1 -f Dockerfile.gocd-plugins .
docker create --name gocd-plugins mrohland/gocd-plugins:v1
```

#### Build a data image to contain gocd
Idea: This is the working volume gocd works on. It won't get versioned as the data(pipelines + other conf) will get backed up by gocd backup pipeline into github)
```
docker build -t mrohland/gocd-data -f Dockerfile.gocd-data .
docker create --name gocd-data mrohland/gocd-data
```

#### Build an agent image to add docker to standard gocd agent image
Copy ```certs/domain.crt``` to ```gocd-agent/registry.crt``` prior to building the image to use private registry.
```
docker build -t mrohland/gocd-agent -f Dockerfile.gocd-agent .
```

#### Run go.cd server and link plugin volume against data volume and that against gocd-server
```
docker start gocd-plugins
docker start gocd-data
docker run -ti -p 8153:8153 --volumes-from=gocd-data --volumes-from=gocd-plugins --name gocd-server gocd/gocd-server
# run at least one go.cd build agent via
for each in 1 ; do docker run -d --privileged --link gocd-server:go-server --name gocd-agent-$each mrohland/gocd-agent; done
```

#### open gocd server via:
```
open http://$(boot2docker ip):$(docker inspect --format='{{(index (index .NetworkSettings.Ports "8153/tcp") 0).HostPort}}' gocd-server)
```
