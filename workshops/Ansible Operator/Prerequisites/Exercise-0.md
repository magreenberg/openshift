# Exercise 0 - Workshop Configuration
## Logging in to OpenShift
First let’s log in to the cluster (login credentials placed in the sheets file under ocp user and ocp password):
```bash
$ export OCP_CLUSTER="" # Ask the Instructor 
$ export OCP_DOMAIN="" # Ask the Instructor 
$ oc login -u ${USER} api.$OCP_CLUSTER.$OCP_DOMAIN:6443
```

## Logging in to the Workshop Registry
Different registries are used in this workshop. Your instructor will direct you which of the steps in this section to follow to log in to the registry for this workshop.

### Logging in to the Internal OpenShift Registry
Log in to the internal OpenShift registry by running:
```bash
$ REGISTRY="efault-route-openshift-image-registry.apps.$OCP_CLUSTER.$OCP_DOMAIN"
$ podman login -u $(oc whoami) -p $(oc whoami -t) ${REGISTRY}
```
The output should be:
```
Login Succeeded!
```

### Logging in to an External Registry
Your instructor will provide you with the name of the external registry. Log in to the registry as follows. Enter the password provided by the instructor:
```bash
$ REGISTRY=<name of registry provided by instructor>
$ podman login -u ${USER} ${REGISTRY}
```

## Configuring an Environment Variable
Create an environment variable for the registry that will be used in this workshop:
```bash
$ echo "REGISTRY=${REGISTRY}" >> ~/.bashrc
$ source ~/.bashrc
```