- [Nexus Repository Manager on OpenShift](#nexus-repository-manager-on-openshift)
  - [Operator installation](#operator-installation)
  - [NexusRepo creation](#nexusrepo-creation)
  - [Expose UI and Docker registry](#expose-ui-and-docker-registry)
  - [Login to UI](#login-to-ui)
  - [Create new Docker repository](#create-new-docker-repository)
  - [Login to Nexus Docker repo](#login-to-nexus-docker-repo)

# Nexus Repository Manager on OpenShift

## Operator installation

Install Nexus Operator via OperatorHub to dedicated namespace (`nexus` in my example)

```shell
oc new-project nexus
oc create -f manifests/operator.yaml
```

## NexusRepo creation

Differences from vanilla configuration:

- Service type changed to `ClusterIP` instead of `NodePort`.
- Enabled persistence mode to create PV/PVC.
- Added `Djava.util.prefs.userRoot=/nexus-data/javaprefs` parameter to workaround issue `Could not lock User prefs`. [Knowledge base](https://support.sonatype.com/hc/en-us/articles/213464868-Nexus-startup-fails-with-Could-not-lock-User-prefs-Couldn-t-flush-user-prefs-Couldn-t-get-file-lock-)

Create NexusRepo instance

```shell
oc create -f manifests/nexus-repo.yaml
```

## Expose UI and Docker registry

```shell
oc create -f manifests/service-docker.yaml
oc create -f manifests/route-ui.yaml
oc create -f manifests/route-docker.yaml
```

## Login to UI

[Default permissions](https://help.sonatype.com/repomanager2/installing-and-running/post-install-checklist#PostInstallChecklist-Step1:ChangetheAdministrativePasswordandEmailAddress)

```shell
NEXUS_URL=$(oc get route nexus-ui -n nexus --template='{{ .spec.host }}')
xdg-open "https://${NEXUS_URL}"
```

## Create new Docker repository

The following command create hosted docker repository with name `docker`.

```shell
NEXUS_URL=$(oc get route nexus-ui -n nexus --template='{{ .spec.host }}')
curl -X 'POST' \
  "https://${NEXUS_URL}/service/rest/v1/repositories/docker/hosted" \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -u 'admin:admin123' \
  -d '{
  "name": "docker",
  "online": true,
  "storage": {
    "blobStoreName": "default",
    "strictContentTypeValidation": true,
    "writePolicy": "allow_once",
    "latestPolicy": true
  },
  "cleanup": {
    "policyNames": [
      "string"
    ]
  },
  "component": {
    "proprietaryComponents": true
  },
  "docker": {
    "v1Enabled": false,
    "forceBasicAuth": true,
    "httpPort": 9001
  }
}'
```

## Login to Nexus Docker repo

```shell
DOCKER_URL=$(oc get route nexus-docker -n nexus --template='{{ .spec.host }}')
docker login "${DOCKER_URL}"
```
