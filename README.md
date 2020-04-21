Learn how to take advantage of the Docker image layering model to run unit or component tests in a Docker container without polluting the production software.


### Introduction
Testing in the right environment is always a challenge. When building an application, a developer can't be sure that the test environment (for example, Jenkins machine) is using the same version of node with the same configurations as in production.

Docker introduced the option to package the application so the environment is part of the deployed application. In this article, I will show how to take advantage of the Docker image layering model in order to run unit tests or component tests within a Docker container without polluting the production software with test code, data, or configuration.

We will use a node.js and express application and build a Docker image for production. Then, we will build another Docker image from the production image as a base image, add the test code, and run a container from this test image.

Basic knowledge in Docker and node.js is required to understand the application ecplained below.

### The Application
For this we will build a simple HTTP calculator  server. The server responses to the following APIs:

```http
GET /calculator/add - add two floating point numbers.
GET /calculator/sub - subtract a floating point number from a second one.
GET /calculator/mul - multiple two floating point numbers.
GET /calculator/div - divide a floating point number by another one.
```
Each one of these API requires two query parameters: first and second. For example,

`GET /calculator/sub?first=3.4&second=1.4`

will return a JSON object of:

`{"result" : 2}`

The Tests
The Git repository includes the test.js file under the tests folder. The tests use the supertest library and run using mocha.  

#### Dockerfiles
There are two Dockerfiles in the Git repository. The first one is the Dockerfile for the production image, named "Dockerfle.production":

```dockerfile
# This official base image contains node.js and npm
FROM node:7
ARG VERSION=1.0.0
# Copy the application files
WORKDIR /usr/src/app
COPY package.json app.js LICENSE /usr/src/app/
COPY lib /usr/src/app/lib/
LABEL license=MIT \
      version=$VERSION
# Set required environment variables
ENV NODE_ENV production
# Download the required packages for production
RUN npm update
# Make the application run when running the container
CMD ["node", "app.js"]
```

We'll use the following command (including the *`.`* at the end) in order to build this image, and name the new image "calculator":

`docker build -t calculator --build-arg VERSION=$MY_COMPONENT_VERSION -f Dockerfle.production .`

Now, we have a non-tested image for production. It can be run, but first, let's test it.

To do that, we will build an additional image, using the new "calculator" image we just built. We'll use Dockerfile.test file for this purpose:

```dockerfile
# Use the production image as base image
FROM calculator
# Copy the test files
COPY tests tests
# Override the NODE_ENV environment variable to 'dev', in order to get required test packages
ENV NODE_ENV dev
# 1. Get test packages; AND
# 2. Install our test framework - mocha
RUN npm update && \
    npm install -g mocha
# Override the command, to run the test instead of the application
CMD ["mocha", "tests/test.js", "--reporter", "spec"]
```

Here is the build command for the test image, named calculator-test:

`docker build -t calculator-test -f Dockerfile.test .`

Now, let's run the tests:

`docker run --rm calculator-test`

**Note**: We're using the **`--rm`** flag to automatically remove the test container after the test finished because we don't need the test container anymore, and with a build machine, when many teams are building many images and testing containers, we want to reduce the disk space our build task is creating.

Here is the output from this run. Notice that the status is valid (0).

#### Test Output

If some tests will fail, the status of this command will be not valid.

So, what has happened here? We have a container with the production code, as well as the required environment (node and NPM). We added some tests and development libraries and we ran these tests in the same environment production environment with the same code and the same version of libraries and frameworks. This is a great advantage of this method, because testing in a build machine like Jenkins (and only then build the Docker image) may lead to unexpected surprises in production, caused by changes in the environment. 

The Next Step
As we saw above, the Docker run command of the test container, returns a status code of 0 (tests passed) or 1 (something went wrong). The meaning of this is that we can use this technique in a CI building system. Here is an example for a script in a Jenkins job that builds the image, tests it, and pushes it if the test passed. The script also updates the version of the software and update Git with the new version.

```dockerfile
# Update the version in the package.json file and start a new
# git tag. The version variable contains the new version.
git_tag=`npm version prerelease`
#remove the 'v' prefix from the version string
version=${git_tag:1}
# Prepare the image prodution image name, with the new version.
export TAG_BASE=my_repository/calculator
export TAG=$TAG_BASE:${version}
# add a label with the version. We're using a new file, 
# because we are going to push the changes to git,
#and we don't want to modify the original Dockerfile.
cp Dockerfile.production Dockerfile
# Building the image (we don't nee the -f, because Dockerfile
# is the default):
docker build -t $TAG --no-cache --build-arg VERSION=$version.
# Adding a local tag to the new image, for the test image's FROM:
docker tag $TAG calcolator
# Building the test image:
docker build -t calculator-test -f Dockerfile.test --no-cache .
# Running the test:
docker run --rm calculator-test
# Remove the test image (cleanup)
docker rmi calculator-test
# Push the new version back to git
git checkout master
git merge ${git_tag}
git push --follow-tags
# Push the new tested production image to a docker 
# repository:
docker push $TAG
# [optional, based on the deployment policies] push 
# the latest tag for this image:
docker tag $TAG $BASE_TAG:latest
docker push $BASE_TAG:latest

```
If the test will fail, the script exits with an error and both the Git push and the image push will not happen, and will not pollute the Git tags nor the docker repository with a broken version.

Original Post : https://dzone.com/articles/testing-nodejs-application-using-mocha-and-docker
