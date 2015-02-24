# AWS Lambda + Github + Taskcluster:

AWS Lambda offers very attractive pricing and a ec2 secured environment that is fast/reliable/secure it has a few key downsides though:

 - Tasks must be limited to 60s or less
 - Your node version is restricted (note: you can also compile and ship a go or rust binary here)
 - No automated mechanisms for updating or deploying functions.

This describes how we could extend these apis to build a much more ideal task runner system which would be suited for very
short tasks which either post data back to aws or create taskcluster graphs/tasks.


## Lambda VS Taskcluster:

  - Taskcluster will generally be more or as expensive.

  - For cases where we want to use the security model of AWS instead of
    our own.

  - Very very small tasks which taskcluster overhead is near the cost of
    running the task.

  - For "administrative" or "scheduling" tasks which have very low
    compute requirements and/or low network requirements.

## Namespaces and Packaging

A big issue with the lambda functions is they are limited by the [length of their names](http://docs.aws.amazon.com/lambda/latest/dg/API_FunctionConfiguration.html) using a model similar to how go implements their package namespace we can workaround this with a package scheme like this:

```
<relative resource url>/<path>/<package>
```

For example:

```
github.com/lightsofapollo/my-functions/do-things/
```

From here it is an easy thing to hash the name and you have a easy way
to package the functions.

### Package details

Individual functions can specify their "handler" this means you can
often group related actions into a single "function" then reference the
export of that function. (For example you could specify the handler of
main.<export>).

 - do-things/
 - do-things/package.json
 - do-things/main.js

#### Bundling of packages

The abstract of the packaging details:

 - cd <package>
 - npm install --production
 - <zip the contents of package>


### Sync

A key missing feature is the ability to sync a vcs controlled repository
(something on github/etc...) to lambda functions. This is easily done
now that we have a naming scheme and understand how to package
functions.

Taskcluster (or even lambda if your packages are small) can easily then
generate the lambda functions with the following algorithm:

  - Iterate through each package: derive name of function from hash of repository name + path
     1. Call get function
        a. Function does not exist go to 2.
        b. Function exists and description contains current revision of directory go to 4.
        
     2. Bundle package. Initiate upload of package with hash as function
        name and the description as follows:

        `<repository path>@<revision>`

    3. Continue to next package.

The above could be run as multiple tasks/functions (one checking for
updates another running updates).

In a pure lambda pipeline something like this could be done:

  - Checkout vcs at revision X, upload to s3
  - Checkout vcs from s3 look for changes in each package.
