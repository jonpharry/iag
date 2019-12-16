# IBM Application Gateway Deployment Assets
These are my work-in-progress assets for deploying IBM Application Gateway (IAG) Tech Preview.

## Cloud Identity configuration
The IBM Application Gateway requires an IBM Cloud Identity Tenant for authentication.
For information on setting up a free trial tenant, check out https://ibm.biz/cloudidcookbook (Exercises 1 and 2).

You will need to configure a Custom Application with these settings:
  - Sign-in Method: Open ID Connect 1.0
  - Grant Types: Authorization Code
  - Redirect URL: \<Route\>/pkmsoidc (e.g. https://iag.127.0.0.1.nip.io/pkmsoidc)

Once this application is created, you can get the Client ID and Client Secret which you will need to configure IAG.

## Generate a certificate and key
With the exception of the "Hello World" OpenShift template, all other deployments in this repository expect to use a pre-created certificate+key for the IAG HTTPS connection.  In the common directory is a script which will create a certificate+private key for the front end of the application gateway.  This script will create the certificate with extended key usage and lifetime set so that it is accepted by latest Firefox and Chrome browsers.

```
common/create-cert.sh
```

A certkey.pem file is created which can be mounted to the IAG container (either via a host mount or via a secret).
The script also outputs the base64 encoding of this file which can be used in-line in a configuration file.

## Docker
Native Docker assets are in the docker folder.
If you want to build a Docker test system you can use this asset:
  - https://ibm.biz/isamdockerbuild

### Create docker.properties file
The iag-run.sh script reads OIDC information from docker/docker.properties file.  Copy the sample file:
```
cp docker/docker.properties.sample docker/docker.properties
```
and then edit `docker\docker.properties` and fill in your information.

### Run IAG
A script is provided for starting the IAG in a native Docker container.  This script has the following usage:
```
docker/iag-run.sh <config-directory> <publish host>:<port>
```

For example:
```
docker/iag-run.sh hello-world 127.0.0.1:443
```

### Clean up Docker
Stop and remove the IAG docker container:
```
docker rm -f iag-<config name>
```

Clean up the bind mount directories:
```
rm -rf docker/*.mount
```

## OpenShift
OpenShift assets are in the openshift folder.
If you want to build an OpenShift test system you can use these assets:
  - OKD on Centos 7 (https://ibm.biz/isamopenshiftbuild)
  - OKD on MacOS (https://ibm.biz/openshiftmac)

### Hello World OpenShift Template
A "Hello World" OpenShift template is provided which can be deployed to test the IAG.

```
oc create -f openshift/iag-hello-world.yaml
```

Once you have created this template, use the OpenShift console to deploy it.  you will need to complete the CI Tenant Hostname, OIDC Client ID,
and OIDC Client Secret in the template deployment wizard.

Once deployed, use the Route to connect to the IAG.  You will be redirected to your Cloud Identity tenant to authenticate.
Once authenticated, you can request the `/cred-viewer` url to see the attributes provided by Cloud Identity.
e.g. https://iag.127.0.0.1.nip.io/cred-viewer

### Common: Create a crypto secret
A script is provided that will create a Secret (iag-crypto) from the certkey.pem file in the common directory.  If the certkey.pem file doesn't exist, the create-cert.sh script will be called to generate it.

```
openshift/create-crypto-secret.sh
```

### Option 1: Deploy using local ConfigMap
In this case you load your configuration into an entry of a ConfigMap object.  This ConfigMap is then mounted to the /var/iag/config directory of the IAG container.

#### Create Config Map
Create a config map containing your configuration:
```
oc create configmap iag-config --from-file configs/hello-world/src/hello-world.yaml
```

#### Install Template
Install the template:
```
oc create -f openshift/iag-configmap-template.yaml
```

#### Deploy Template
Now open the OpenShift UI (e.g. https://localhost:8443), select project, and go to catalog. You will see icon for the application.  Click to deploy.  You can change parameters before deploy.  Parameters include CI Tenant Hostname and OIDC Client ID and Client Secret.

### Option 2: Deploy with Build from Source Repository
In this case your configuration files are downloaded from a source repository and baked into a new Docker image by a BuildConfig.  The new image is loaded to an ImageStream which a DeploymentConfig then uses to create the IAG containers.

A sample "Hello World" repository is pre-configured in the template parameters.

#### Install Template
Install the template:
```
oc create -f openshift/iag-build-template.yaml
```

#### Deploy Template
Now open the OpenShift UI (e.g. https://localhost:8443), select project, and go to catalog. You will see icon for the application.  Click to deploy.  You can change parameters before deploy.  Parameters include CI Tenant Hostname and OIDC Client ID and Client Secret.


### Clean up IAG Assets
You can uninstall all assets associated with the IAG (iag is default app name) using command:
```
oc delete all -l app=iag
oc delete secret -l app=iag
oc delete secret iag-crypto
oc delete configmap iag-config
```

# License

The contents of this repository are open-source under the Apache 2.0 licence.

Copyright 2019 International Business Machines

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
