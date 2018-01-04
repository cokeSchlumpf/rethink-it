# Deployment of OpenWhisk projects with multiple actions

In the past few months I was working on some [Apache OpenWhisk](https://openwhisk.apache.org/) projects for my own. In this blog post I'd like to share the experience I made with OpenWhisk when it comes to deployment of larger projects with more than just a few actions.

---

For this post I will focus on Node JS actions and OpenWhisk provided by [IBM Cloud Functions](https://console.bluemix.net/openwhisk/). Other languages and other providers are quite similar.

## Basic deplyoment of OpenWhisk actions

The first things you need to think about when you are going to automate deployment processes are how you can execute build, packaging and deployment steps programmatically. OpenWhisk provides a handy [RESTful API](https://console.ng.bluemix.net/apidocs/98) to execute all tasks you can imagine. This RESTful API can be used indirectly through the the [OpenWhisk CLI](https://github.com/apache/incubator-openwhisk-cli) as well as the [OpenWhisk NPM package](https://www.npmjs.com/package/openwhisk).

When I started I was using the CLI to deploy my actions, in the most simple case when you just have a single javascript file you can deploy it as follows:

```bash
$ wsk action create my-action index.js
ok: created action hello
```

As long as you're action can be coded in a single file and doesn't require an additional dependency which is not provided by OpenWhisk by default, you can use this simple deployment for a single action. Sooner or later actions becoming more complex and you require multiple files or additional dependencies, in that case you can package actions including `node_modules` as ZIP file before creating an action.

```bash
# In your action directory
$ ls -al
total 136
drwxr-xr-x    8 michael  staff    272 29 Dec 14:56 .
drwxr-xr-x   32 michael  staff   1088 29 Dec 14:04 ..
-rw-r--r--    1 michael  staff   9015  2 Dec 20:52 index.js
-rw-r--r--    1 michael  staff  10455 25 Nov 16:44 index.test.js
-rw-r--r--    1 michael  staff    604 19 Nov 19:50 package.json

$ npm install

$ ls -al
total 136
drwxr-xr-x    8 michael  staff    272 29 Dec 14:56 .
drwxr-xr-x   32 michael  staff   1088 29 Dec 14:04 ..
-rw-r--r--    1 michael  staff   9015  2 Dec 20:52 index.js
-rw-r--r--    1 michael  staff  10455 25 Nov 16:44 index.test.js
drwxr-xr-x  131 michael  staff   4454 15 Sep 01:21 node_modules
-rw-r--r--    1 michael  staff  34851 15 Sep 01:21 package-lock.json
-rw-r--r--    1 michael  staff    604 19 Nov 19:50 package.json

$ zip -r action.zip *

$ wsk action create my-action --kind nodejs:6 action.zip
```

Actually, I recommend to always use a `package.json` and the ZIP deployment to ensure that your dependencies are always in the version your code expects. The disadvantage of the ZIP packaging is that the upload time increases during deployment. This comes uncomely when you have many actions in your project.

See also:

* Creating and deploying actions in [OpenWhisk documentation](https://console.bluemix.net/docs/openwhisk/openwhisk_actions.html#openwhisk_actions)
* List of default NodeJS packages in [IBM Cloud Functions](https://console.bluemix.net/docs/openwhisk/openwhisk_reference.html#openwhisk_ref_javascript)

## Deploying multiple actions

When projects are getting more complex, more than just a few actions and maybe multiple packages need to be handled during deployment. A complex project may have a structure as follows:

```
└ my-openwhisk-project
  ├ any-package
  │ ├ my-first-action
  │ │ ├ index.js
  │ │ └ package.json  
  │ ├ my-second-action
  │ │ ├ index.js
  │ │ ├ index.test.js
  │ │ └ package.json
  │ └ ...
  ├ another-package
  │ ├ nice-action
  │ │ ├ index.js
  │ │ ├ foo.js
  │ │ └ package.json  
  │ ├ cute-action
  │ │ ├ index.js
  │ │ └ package.json
  │ └ ...
  └ package.json
```

Starting simple, for deploying the whole project one could create a script as follows:

```bash
#!/bin/bash

# execute in my-openwhisk-package
$ cd any-package
$ wsk package create any-package
$ cd my-first-action
$ zip * action.zip
$ wsk action create my-first-action --kind nodejs:6 action.zip
$ cd ../my-second-action/
$ zip * action.zip
$ wsk action create my-second-action --kind nodejs:6 action.zip
# ...
```

This approach has a few disadvantages:

  * With that simple script above you only create packages/ actions - Additional logic needs to be added to detect, whether to update or create the objects
  * When adding/ removing actions or packages, the script needs to be adopted.

Thus I created a shell script for myself to deploy a whole OpenWhisk package at once:

```bash
#!/bin/bash

# Configuration
PACKAGE=${PACKAGE}-generic
PACKAGE_API=${PACKAGE}-api

# Create package if not existing
echo "Creating Packages '${PACKAGE}' && '${PACKAGE_API}'"
wsk package create ${PACKAGE} &> /dev/null || true
wsk package create ${PACKAGE_API}  &> /dev/null || true

for dir in `find . -maxdepth 1 -mindepth 1 -type d | grep -v _template`
do
    ACTION=`echo $dir | awk -F'/' '{ print $2 }'`;

    pushd ${dir} > /dev/null
      echo "Creating package for action '${ACTION}' ..."
      NAME="${PACKAGE}/${ACTION}"
      DESCRIPTION=`cat package.json | grep description | awk -F'"' '{ print $4 }'`
      zip -r action.zip * > /dev/null

      echo "Creating action '${NAME}' ..."
      CMD=`wsk action list | grep ${NAME} > /dev/null && echo "update" || echo "create"`
      wsk action ${CMD} ${NAME} \
        --kind nodejs:6 action.zip \
        -a description "${DESCRIPTION}"

      rm action.zip
    popd > /dev/null
done
```

This script is at least somehow generic and it doesn't fail if the package or the action already exists. But still we're facing some disadvantages:

  * Each deployment deploys every action, even if it has not changed - usually you're working on one action only. This simply takes time and if we have 10+ actions it just takes too much time. Especially if you want to integrate the deployment in a CD/CI pipeline.
  * An automated deployment also has to care about the execution of `npm install` and maybe `npm test` for all packages.
  * If actions are deleted or renamed you have to delete it manually from OpenWhisk as the script doesn't know about it.

I lived for a while with these disadvantages, but as projects were getting more and more complex and I spent too much time due to these deployment disadvanatges I started to create a smarter tool for deploying my actions: [openwhisk-action-manager](https://github.com/cokeSchlumpf/openwhisk-action-manager). This tool basically is an enhanced version of the shell script above, to name some of its features:

  * It creates OpenWhisk packages on deployment if not present
  * It creates OpenWhisk actions on deployment if not present
  * It checks if an update of an action is required due to code changes
  * It deletes old actions (e.g. after renaming an action)
  * Before creation/ update of an action it:
    * Executes `npm install` for each action
    * Packages the action as ZIP archive
  * It uploads/ creates the action on OpenWhisk
  * Like the OpenWhisk CLI openwhisk-action-manager reads its OpenWhisk configuration from `~/.wskprops`

The main enhancement this tool gives you, is that it can detect if an action needs to be updated or not. This is done by calculating an MD5 hash before uploading the action and attaching the value as OpenWhisk annotation to the action. When the deployment runs the next time, the new calculated hash is compared to the one stored in the action's annotation.

With openwhisk-action-manager the deployment of multiple actions is straight forward, in the example above we can do the following:

```bash
# execute in my-openwhisk-package
$ npm init
# Enter some meaningful values on the prompts

$ npm install --save-dev openwhisk-action-manager
$ ./node-modules/.bin/openwhisk-action-manager -d any-package
$ ./node-modules/.bin/openwhisk-action-manager -d another-package
```

If you're interested, check the docs of openwhisk-action-manager on [GitHub](https://github.com/cokeSchlumpf/openwhisk-action-manager).

## Conclusion

As of now there is much open space for tooling around OpenWhisk, especially when it comes to more complex application architectures. When I started I also had an eye on the [serverless framework](https://serverless.com/) which also offers an integration to OpenWhisk; but I was not really happy with the functionality and from my point of view it didn't fit well to OpenWhisk - This might have changed in the meantime.

The good thing about OpenWhisk is that it's quite simple to write scripts and tools for it due to it's open nature and architecture which relies on open standards and interfaces.