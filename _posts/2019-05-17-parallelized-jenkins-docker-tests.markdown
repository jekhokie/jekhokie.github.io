---
layout: post
title:  "Parallelized Jenkins Jobs using Docker"
date:   2019-05-16 19:15:02 -0400
categories: ci cd jenkins virtualbox vm infrastructure docker ansible parallel
---
As software test suites grow, so does the timeline for feedback to a developer given the length of time it takes to
run the tests. In a monolithic environment this problem can be compounded if the suite of tests is singular and
inclusive of all functionality. This post is an attempt to solve this problem using Docker containers with parallel
test jobs.

### Project Materials

There is a sample repository that contains the end-state code that can be used for this tutorial if desired.
The project can be found [here](https://github.com/jekhokie/python-parallel-ci-tests), and can be cloned to obtain
the materials (i.e. Jenkinsfile and Python code/tests) for this tutorial.

### Proposed Pipeline and Functionality

This tutorial assumes a standard Jenkins pipeline that consists of the following stages:

{% highlight bash %}
       1                  2                   3
 |-----------|       |----------|        |----------|
 |  Print    |       |  Test    |        |  Report  |
 |  Info     | ----> |  Suite   | -----> |  Results |
 |           |  |    |    1     |    |   |          |
 |-----------|  |    |----------|    |   |----------|
                |                    |
                |    |----------|    |
                |    |  Test    |    |
                |--> |  Suite   | -->|
                |    |    2     |    |
                |    |----------|    |
                |                    |
                |    |----------|    |
                |    |  Test    |    |
                |--> |  Suite   | -->|
                     |    3     |
                     |----------|
{% endhighlight %}

As you can see above, the idea is to have a Jenkins pipeline that first checks out the source code, then triggers 3
parallel test builds (one for each "suite" of tests in a project), and then aggregates/reports results from all 3
of the tests, representing the full test suite. The various stages proposed are detailed below:

1. Print out some information for use - just a debug step.
2. Parallelize 3 jobs, each running a single test file from the project repository:
  2a. Test Suite 1: Run the tests/test_app_no_jobid.py tests
  2b. Test Suite 2: Run the tests/test_app_job_id1.py tests
  2c. Test Suite 3: Run the tests/test_app_job_id2.py tests
3. Aggregate all results from previous tests into a single status/report.

### Required Software and Architecture

For the purposes of this tutorial, we will set up, configure, and realize the above pipeline architecture using the
following software stack:

* [Docker for Mac](https://docs.docker.com/v17.12/docker-for-mac/install/): Software to deploy and manage Docker containers on a MacBook.
* [Jenkins](https://jenkins.io/): Traditional CI pipeline, deployed via Docker container locally on the MacBook laptop.
* [Git](https://git-scm.com/): Source control used for the sample project which will realize the Jenkins pipeline, installed on the MacBook laptop.
* [GitHub](https://github.com/): Remote source control repository storage.
* [Socat](https://linux.die.net/man/1/socat): Multipurpose relay for exposing Docker services on MacBook laptop.

The following architecture will be realized by the end of this tutorial:

{% highlight bash %}
 |-----------------------------------|  |--------------------------------|
 |             MacBook Pro           |  |          github.com            |
 |  |-----------------------------|  |  |  |--------------------------|  |
 |  |         Docker Engine       |  |  |  | python-parallel-ci-tests |  |
 |  |                             |  |  |  |       git repository     |  |
 |  | |-----------| |-----------| |  |  |  |--------------------------|  | 
 |  | | Jenkins   | | Agent 2   | |  |  |                                |
 |  | | Container | | Container | |  |  |--------------------------------|
 |  | |-----------| |-----------| |  |
 |  |                             |  |
 |  | |-----------| |-----------| |  |
 |  | | Agent 3   | | Agent 4   | |  |
 |  | | Container | | Container | |  |
 |  | |-----------| |-----------| |  |
 |  |                             |  |
 |  |-----------------------------|  |
 |                                   |
 |-----------------------------------|
{% endhighlight %}

In the above architecture, the MacBook laptop is running a Docker engine with Jenkins running within and will handle the
Jenkins Docker agent creation. Jenkins will be reaching out to GitHub to access the sample repository used for the CI builds.

### Install Dependent Software

On the MacBook, install the dependent git and virtualbox software required:

{% highlight bash %}
$ brew install git socat
{% endhighlight %}

The instructions for installing Docker for Mac are pretty clear and we won't repeat them here. Instead, follow
[these instructions](https://docs.docker.com/v17.12/docker-for-mac/install/). In addition to the instructions, you will need to
expose the Docker services via API. Run the following command locallly on the Mac once the Docker Engine has been installed, which
will expose the Docker services by way of the Unix socket created on port 2376:

{% highlight bash %}
$ socat -d TCP-LISTEN:2376,reuseaddr,fork UNIX:/var/run/docker.sock
{% endhighlight %}

If you open a browser and navigate to `http://localhost:2376` you should see a JSON structure with the message "Page not found",
indicating the services are listening.

### Create Docker Agent Image

The docker image that is best suited for the Jenkins integration (at least in this tutorial) is the `jenkins/ssh-slave` image.
However, this image only comes with JDK installed (required for Jenkins integration) and none of the Python tools. Therefore, we
will build our own variation of the image so each part of the build does not require installing the dependent software into the
container as a prerequisite (the python, pip, and virtualenv binaries will already be present). The `Dockerfile` file is in the
software repository example we are using, and here are the contents:

{% highlight bash %}
FROM jenkins/ssh-slave

RUN apt-get update && apt-get install -y software-properties-common
RUN apt-get update && apt-get install -y \
    python-pip \
    virtualenv
{% endhighlight %}

The file is pretty straightforward - it starts with the `jenkins/ssh-slave` base image and adds the software repositories and
dependent software needed for the tests to run. In order to build the image, run the following commands on the MacBook laptop
in the same directory as the `Dockerfile`:

{% highlight bash %}
$ docker build -t jenkins/ssh-slave-modified .
{% endhighlight %}

The above command will take the base image and apply the Dockerfile changes to it, and save it locally as an image tagged with
the name `jenkins/ssh-slave-modified`. You can see the image by running the command `docker image ls`.

### Deploy and Configure Jenkins

You can now deploy Jenkins in a container using a vendor-provided distribution. Run the following command to download the blueocean
Docker container image and run it as a container:

{% highlight bash %}
$ docker run -p 8080:8080 jenkinsci/blueocean
{% endhighlight %}

On startup, you should see an admin key scroll by in your terminal - copy this. Launch a browser and navigate to
[http://localhost:8080](http://localhost:8080) and you should be prompted to insert the Jenkins admin key to activate the Jenkins
instance. Paste the key captured in the logs scrolling by and progress to the next step, where you should select to install the
"Standard" plugins. After a few minutes, you should be greeted with the Jenkins home page.

#### Jenkins Docker Plugin

In Jenkins, navigate to the "Manage Jenkins" menu option, and select "Manage Plugins". Select the "Available" tab and search for
the "Docker" plugin, select the checkbox, and click "Install without restart". In a few minutes, the Docker plugin should be
installed and ready to be configured.

Navigate to "Manage Jenkins" -> "Configure System". Near the bottom is a "Cloud" option - from the "Add a new cloud" drop-down,
select "Docker". Click the "Docker Cloud Details..." button and enter the following details, where `<MACBOOK_HOSTNAME>` is the
resolvable hostname of your MacBook laptop where the Docker engine is running:

* **Name**: docker
* **Docker Host URI**: tcp://\<MACBOOK_HOSTNAME\>:2376
* **Server credentials**: none
* **Enabled**: Checked
* **Expose DOCKER_HOST**: Checked
* **Container Cap**: 5

Once the configuration has been entered, click the "Test Connection" button. If all goes well, you should be able to see the Docker
version and API version listed. Don't yet hit the "Save" button - we'll next configure the base container image for the build agents.
Click the "Docker Agent templates..." button, then click "Add Docker Template". Specify the following details:

* **Labels**: docker-python-1
* **Enabled**: Checked
* **Docker Image**: jenkins/ssh-slave-modified
* **Instance Capacity**: 5
* **Remote File System Root**: /home/jenkins
* **Usage**: Only build jobs with label expressions matching this node
* **Idle Timeout**: 10
* **Connect method**: Connect with SSH
  * **SSH Key**: Inject SSH key
  * **User**: jenkins
* **Pull strategy**: Never pull

**Note**: The Pull Strategy above is very important - you must specify/configure it as "Never pull"
in order to ensure that the Docker engine uses the local image repository (whatever is available), and since we built the image locally,
this is applicable.

You can now click the "Save" button to save the configuration.

### Configure Jobs

Now that we have our Jenkins configuration in place for using Docker containers as build nodes, we will configure a job using the
remote GitHub repository containing the sample files. The repository we will be using is located [here](https://github.com/jekhokie/python-parallel-ci-tests.git).

In Jenkins, select "New Item". Name the project "parallel-ci" and select "Multibranch Pipeline" as the option. After clicking "OK",
enter the following configuration items (anything not mentioned can be left as default):

* **Display Name:**: parallel-ci
* **Branch Sources** -> **Add source** -> **Git**:
  * **Project Repository**: https://github.com/jekhokie/python-parallel-ci-tests.git
* **Scan Multibranch Pipeline Triggers** -> **Periodically if not otherwise run**: Checked
  * **Interval**: 1 minute

Click "Save". Once the repository is checked out and a job kicks off, you should see the pipeline start to execute indicating things are
working the way you expect them to. Below are a few images that may be useful/interesting to determine if you've successfully configured
the jobs the way they are expected to be configured.

##### **Native Jenkins View of Pipeline**

---
[![Jenkins Native Pipeline][1]][1]
<br/>
<br/>

##### **Blue Ocean View of Pipeline - Parallel Jobs**

---
[![Blue Ocean Pipeline][2]][2]
<br/>
<br/>

##### **Aggregated Test Results**

---
[![Aggregated Test Results][3]][3]
<br/>

### Closing Comments

There is a lot more to do in order to make this parallel pipeline effective, especially as it relates to end-to-end testing. For instance,
how to deal with databases, how multiple containers will talk with each other, etc. This tutorial serves as a starting point for parallel
Jenkins testing using Docker.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [How to Setup Docker Containers as Build Slaves for Jenkins](https://devopscube.com/docker-containers-as-build-slaves-jenkins/)
* [Some Useful Socat Commands](http://technostuff.blogspot.com/2008/10/some-useful-socat-commands.html)
* [Remote API with Docker for Mac (BETA)](https://forums.docker.com/t/remote-api-with-docker-for-mac-beta/15639)
* [Jenkins SSH slave Docker Image](https://hub.docker.com/r/jenkinsci/ssh-slave/)
* [How to create Docker Images with a Dockerfile](https://www.howtoforge.com/tutorial/how-to-create-docker-images-with-dockerfile/)
* [JUnit - Basic Usage](https://www.tutorialspoint.com/junit/junit_basic_usage.htm)
* [StackOverflow - Python Unittests in Jenkins?](https://stackoverflow.com/questions/11241781/python-unittests-in-jenkins)

[1]: /assets/images/2019-05-17-parallelized-jenkins-docker-tests-native-pipeline.png
[2]: /assets/images/2019-05-17-parallelized-jenkins-docker-tests-blue-ocean-pipeline.png
[3]: /assets/images/2019-05-17-parallelized-jenkins-docker-tests-test-results.png
