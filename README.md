# Jenkins slave operator charm
This charm sets up a jenkins-slave in kubernetes.

To prepare this charm for deployment, run the following to install the
framework in to the `lib/` directory:

```
git submodule add https://github.com/canonical/operator mod/operator
```

Link the framework:
```
ln -s ../mod/operator/ops lib/ops
```

Update the operator submodule:
```
git submodule update --init
```

## Testing the docker image

```
juju deploy jenkins
```

Then go on the jenkins interface and create a permanent node called "jenkins-slave-k8s-test" manually for now.

```
export JENKINS_API_TOKEN=$(juju ssh 0 -- sudo cat /var/lib/jenkins/.admin_token)
export JENKINS_IMAGE="jenkins-slave-k8s:devel"
export JENKINS_IP=$(juju status --format json jenkins | jq -r '.machines."0"."ip-addresses"[0]')
make build-image
docker run --rm -ti --name jenkins-slave-test \
 -e JENKINS_API_USER=admin \
 -e JENKINS_API_TOKEN="${JENKINS_API_TOKEN}" \
 -e JENKINS_URL="http://${JENKINS_IP}:8080" \
 -e JENKINS_HOSTNAME="jenkins-slave-test" "${JENKINS_IMAGE}"
```

## Testing with microk8s and LXD

### Install prerequisites
```
sudo snap install --channel 2.8/beta juju --classic
sudo snap install lxd --classic
sudo snap install microk8s --classic
sudo ufw allow in on cni0 && sudo ufw allow out on cni0
sudo ufw default allow routed
sudo snap install docker

# Start microk8s and enable needed modules
microk8s.start
microk8s.enable registry dns storage
juju bootstrap microk8s micro
```

### Build and run the jenkins-slave-k8s image

We consider that you have 2 controllers:

* lxd-local, using a LXD cloud

* micro, using a microk8s cloud

Every following command should be run from the current directory

#### Create the models
```
export MODEL_K8S=jenkins-slave-k8s
export MODEL_IAAS=jenkins

juju add-model -c lxd-local --no-switch "${MODEL_IAAS}"
juju add-model -c micro "${MODEL_K8S}"
juju model-config logging-config="<root>=DEBUG"
```

#### Deploy jenkins master

```
JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}" juju deploy jenkins
```
This will take some time, check the status with
```
JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}" juju status
```
Once ready,
```
export JENKINS_API_TOKEN=$(JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}" juju ssh 0 -- sudo cat /var/lib/jenkins/.admin_token)
export JENKINS_IMAGE="jenkins-slave-k8s:devel"
export JENKINS_IP=$(JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}" juju status --format json jenkins | jq -r '.machines."0"."ip-addresses"[0]')
```

```
export JENKINS_IMAGE="localhost:32000/jenkins-slave-k8s:devel"
make build-image
docker save "${JENKINS_IMAGE?}" > /var/tmp/"${JENKINS_IMAGE##*/}".tar
microk8s.ctr image import /var/tmp/"${JENKINS_IMAGE##*/}".tar
juju deploy . \
  --config "jenkins_agent_name=jenkins-slave-k8s-test" \
  --config "jenkins_api_token=${JENKINS_API_TOKEN:?}" \
  --config "jenkins_master_url=http://${JENKINS_IP:?}:8080" \
  --config "image=${JENKINS_IMAGE}"
```

Once everything is deployed, you can check the logs with
```
microk8s.kubectl -n "${MODEL_K8S}" logs -f --all-containers=true deployment/jenkins-slave
```

#### Upgrade your charm

```
juju upgrade-charm --path .
```


### Create a relation with the jenkins master

* Offer the relation on the jenkins slave
```
juju offer jenkins-slave:slave
```

* On the IAAS jenkins charm
```
JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}"  juju find-offers micro:admin/${MODEL_K8S}
JUJU_MODEL="lxd-local:admin/${MODEL_IAAS}"  juju add-relation jenkins micro:admin/${MODEL_K8S}.jenkins-slave
```
