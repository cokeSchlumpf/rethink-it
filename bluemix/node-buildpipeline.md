# How to use a current Node version on IBM DevOps Delivery Pipeline

The [Delivery Pipeline](https://console.ng.bluemix.net/catalog/delivery-pipeline/) on IBM DevOps services let's you automate builds and deployments to IBM Bluemix in a very easy way. Unfortunately its pre-installed node versions are not up to date and you may need a newer version to build your project.

---

To configure your Build Job select `npm` as Builder Type and use the following shell snippet to easily set up your desired Node version with the help of NVM.

```
#!/bin/bash
export NVM_DIR=/home/pipeline/nvm
export NODE_VERSION=5.10.1
export NVM_VERSION=0.29.0

npm config delete prefix \
  && curl https://raw.githubusercontent.com/creationix/nvm/v${NVM_VERSION}/install.sh | sh \
  && . $NVM_DIR/nvm.sh \
  && nvm install $NODE_VERSION \
  && nvm alias default $NODE_VERSION \
  && nvm use default \
  && node -v \
  && npm -v

npm install
# Further steps ...
```

Just adjust the versions in the exported environment variables in the beginning and you're done.

![Build Job configuration](https://raw.githubusercontent.com/cokeSchlumpf/rethink-it/images/2016-06-02_DeliveryPipeline.png)
