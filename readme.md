- [Nexus Repository Manager on OpenShift](#nexus-repository-manager-on-openshift)
  - [Deploying the operator](#deploying-the-operator)
  - [Configuring NexusRepo](#configuring-nexusrepo)
  - [Exposing UI and Docker registry](#exposing-ui-and-docker-registry)
  - [Login to UI](#login-to-ui)
  - [Creating Docker repository](#creating-docker-repository)
  - [Login to Nexus Docker repo](#login-to-nexus-docker-repo)

# Nexus Repository Manager on OpenShift

## Deploying the operator

Install Nexus Operator via OperatorHub to dedicated namespace (`nexus` in my example) or via CLI.

```shell
oc new-project nexus
oc create -f manifests/operator.yaml
```

## Configuring NexusRepo

Differences from vanilla configuration:

- Service type changed to `ClusterIP` instead of `NodePort`.
- Enabled persistence mode to create PV/PVC.
- Added `Djava.util.prefs.userRoot=/nexus-data/javaprefs` parameter to workaround issue `Could not lock User prefs`. [Knowledge base](https://support.sonatype.com/hc/en-us/articles/213464868-Nexus-startup-fails-with-Could-not-lock-User-prefs-Couldn-t-flush-user-prefs-Couldn-t-get-file-lock-)

Create NexusRepo instance

```shell
oc create -f manifests/nexus-repo.yaml
```

## Exposing UI and Docker registry

```shell
oc create -f manifests/service-docker.yaml
oc create -f manifests/route-ui.yaml
oc create -f manifests/route-docker.yaml
```

## Login to UI

[Default credentials](https://help.sonatype.com/repomanager2/installing-and-running/post-install-checklist#PostInstallChecklist-Step1:ChangetheAdministrativePasswordandEmailAddress)

```shell
NEXUS_URL=$(oc get route nexus-ui -n nexus --template='{{ .spec.host }}')
xdg-open "https://${NEXUS_URL}"
```

## Creating Docker repository

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
