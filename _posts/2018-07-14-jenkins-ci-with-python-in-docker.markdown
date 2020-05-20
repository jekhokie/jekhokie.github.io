---
layout: post
title:  "Jenkins CI with Python in Docker"
date:   2018-07-14 18:41:00 -0400
categories: ubuntu linux jenkins docker container ci python
logo: jenkins.jpg
---
An attempt to bring together various components including Jenkins for Continuous Integration (CI),
Python for development, and Docker containers for consistency/quality/speed. The post will incrementally
demonstrate how to configure a Jenkins pipeline to test a Docker container which contains Python
code and associated automated tests. Note that this tutorial is a culmination of several "best practice"
tutorials by the specific vendors of the technologies within.

### Technology Ecosystem

The steps in this post are executed on a VM running the Ubuntu operating system (specifically,
Ubuntu 16.04, 64-bit). Details related to the host machine are as follows:

**Development Host**

- **Hostname**: devbox.localhost
- **OS**: Ubuntu 16.04
- **CPU**: 2
- **RAM**: 4096 MB
- **Network**: Private Network
- **IP**: 10.11.13.15

### Install Docker

As a first step, let's install Docker on the Linux VM that will be hosting the full suite of
applications:

{% highlight bash %}
# Host: devbox.localhost
$ wget -qO- https://get.docker.com/ | sh
$ sudo usermod -aG docker $(whoami)
{% endhighlight %}

At this point, Docker is installed and the current user (`vagrant` if you're using a Vagrant
VM) is part of the docker group. However, you must log out/log back into the VM before you
are able to run commands.

{% highlight bash %}
# Host: devbox.localhost

# log out, then log back in again

# test the docker functionality
$ docker -v                   # show Docker version
$ docker info                 # show Docker global info
$ docker images               # list Docker images
$ docker container ls --all   # list all containers running/stopped

# test docker hello world
$ docker run hello-world
# should output "Hello from Docker!..." message
{% endhighlight %}

### Create a "Hello World" Python Application

Now, let's create a simple Python web application using Python Flask. Create a directory
named `/opt/hello_world` and use `virtualenv` to create an isolated Python working
environment (you may need to install virtualenv using `pip`):

{% highlight bash %}
# Host: devbox.localhost
$ sudo mkdir /opt/hello_world
$ sudo chmod 777 /opt/hello_world
$ cd /opt/hello_world/
$ virtualenv .env
$ source .env/bin/activate
$ mkdir app
{% endhighlight %}

Next, add a few files/install Flask:

{% highlight bash %}
# Host: devbox.localhost
$ vim app/__init__.py
# ensure contains the following:
#    import os
#    from flask import Flask
#    import socket
#    
#    app = Flask(__name__)
#    
#    # a simple page that says hello
#    @app.route('/')
#    def hello():
#        html = "<h3>Hello {name}!</h3>" \
#               "<b>Hostname:</b> {hostname}<br/>"
#        return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())

$ vim requirements.txt
# ensure contains the following:
#    Flask
#    Redis
#    pytest

$ vim run.py
# ensure contains the following:
#    from app import app
#    app.run(host='0.0.0.0', port=80)

$ pip install -r requirements.txt
# should install the dependencies listed in the requirements file
{% endhighlight %}

### Create Dockerfile

Now that we have Docker installed and a Python application, let's create the Dockerfile to build
the Docker container which will host the application:

{% highlight bash %}
# Host: devbox.localhost
$ vim Dockerfile
# ensure contains the following:
#    # Use an official Python runtime as a parent image
#    FROM python:2.7-slim
#    
#    # Set the working directory to /app
#    WORKDIR /app
#    
#    # Copy the current directory contents into the container at /app
#    ADD . /app
#    
#    # Install any needed packages specified in requirements.txt
#    RUN pip install -r requirements.txt
#    
#    # Make port 80 available to the world outside this container
#    EXPOSE 80
#    
#    # Define environment variable
#    ENV NAME World
#    
#    # Run app.py when the container launches
#    CMD ["python", "run.py"]
{% endhighlight %}

### Create Docker Container and Test

Finally, now that we have the Flask Python files and corresponding Dockerfile, we can create the
Docker image and from it, launch a container with the application:

{% highlight bash %}
# Host: devbox.localhost
# build the image
$ docker build -t helloworldapp .
$ docker image ls
# should result in something similar to:
#    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
#    helloworldapp       latest              51733dd166d5        34 seconds ago      132MB

$ docker run -p 4040:80 helloworldapp
{% endhighlight %}

If all goes well, you can now access the application using a browser at the following URL (assuming
the IP address of your VM matches the above configuration):

`http://10.11.13.15:4040/`

You should see a web page displayed with "Hello World!" and a Hostname. If you wanted to run the
container in the background (detached), you could use the following commands to launch/kill the
application:

{% highlight bash %}
# Host: devbox.localhost
# launch the application in daemon/background mode
$ docker run -d -p 4040:80 helloworldapp

# stop the container/kill the container
$ docker container ls
# obtain the ID of the running container, and use it in the next command:
$ docker container stop <CONTAINER_ID>
{% endhighlight %}

### Adding Python Tests

Now we'll add the pytest framework and add a simple unit test to the application.

{% highlight bash %}
# Host: devbox.localhost
$ vim requirements.txt
# add the 'pytest' library and save
$ pip install -r requirements.txt

$ mkdir tests
$ vim tests/test_app.py
# ensure contains the following:
#    import pytest
#    from app import *
#    
#    @pytest.fixture
#    def client():
#        client = app.test_client()
#        return client
#    
#    def test_root(client):
#        """Test the default route."""
#    
#        res = client.get('/')
#        assert b'Hello world!' in res.data
{% endhighlight %}

To run the unit tests, run the following command:

{% highlight bash %}
# Host: devbox.localhost
$ python -m pytest tests/test_app.py
# should output similar to the following (green output):
#    ================================ test session starts =================================
#    platform linux2 -- Python 2.7.12, pytest-3.6.3, py-1.5.4, pluggy-0.6.0
#    rootdir: /opt/hello_world, inifile:
#    plugins: flask-0.10.0, testinfra-1.12.0
#    collected 1 item
#    
#    tests/test_app.py .                                                             [100%]
#    
#    ==============================1 passed in 0.01 seconds ===============================
{% endhighlight %}

### Installing Jenkins

Now that we have Docker installed and a demo/test Python Flask application with a basic unit test,
we will install and configure Jenkins so we can develop a CI/CD pipeline for the code base.

{% highlight bash %}
# Host: devbox.localhost
# add the repository for installation and install the Jenkins application
$ wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
$ echo deb https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
$ sudo apt-get update
$ sudo apt-get install jenkins

# add the jenkins user to the correct docker group to enable running docker commands
$ sudo usermod -G docker -a jenkins

# add the jenkins user to the vagrant group so it can access the files created by
# the jenkins user (the hello_world project) - this is not ideal, but works for
# the purposes of this tutorial
$ sudo usermod -G vagrant -a jenkins

# start the jenkins process and check the status
$ sudo systemctl start jenkins
$ sudo systemctl status jenkins
# should show jenkins as "Active"
{% endhighlight %}

Access the Jenkins site via a browser at the following URL (again, assuming your IP address matches that
of the IP of the test VM used above):

`http://10.11.13.15:8080/`

Follow the steps in the browser to initialize the Jenkins instance and configure the administrative
user for the application.

### Creating a Jenkins CI Job

We can now create a Jenkins CI job for the Jenkins application to run. The following steps will be
performed within the same browser session as previously logged into.

First, create a "New Job". Use the name "Hello World App" and select "Pipeline" as the project type.

For the new project being created, enter the following details:

1. **Build Triggers**: Specify "Poll SCM" as the option, and enter the schedule `* * * * *` to have
Jenkins poll the file system for changes every minute (configure this to your liking).
2. **Pipeline**: Add the following code, which triggers the various steps in the pipeline:
{% highlight bash %}
node {
   stage('Get Source') {
      // copy source code from local file system and test
      // for a Dockerfile to build the Docker image
      deleteDir()
      sh "cp -rf /opt/hello_world hello_world"
      if (!fileExists("hello_world/Dockerfile")) {
         error('Dockerfile missing.')
      }
   }
   stage('Unit Test') {
      // run the unit tests
      dir("hello_world") {
         sh ". .env/bin/activate"
         sh "pip install -r requirements.txt"
         sh "python -m pytest tests/test_app.py"
      }
   }
   stage('Build Docker') {
       // build the docker image from the source code using the BUILD_ID parameter in image name
       dir("hello_world") {
         sh "docker build -t helloworldapp-${BUILD_ID} ."
       }
   }
}
{% endhighlight %}

As a note, before you run a build, you will likely need to remove the `__pycache__` directory from
the `hello_world` directory to prevent access permission issues. Ideally, this would be handled
differently (and as stated previously, even more ideally, using a real git endpoint is desired over
a local file system approach), but for the purposes of expedience for this tutorial, the immediate
recommendation is to remove all directories using a command like the following when in the
`hello_world` directory:

`find . -name __pycache__ -type d -exec rm -rf {} \;`

### Running the CI Job and Inspecting Docker

Now that we have our Jenkins project set up, we can trigger a build and inspect the output. Within
the Jenkins UI, navigate to your project and trigger a build by selecting the "Build Now" option.
You should see all 3 steps for the build executed/green. If you see any red/errors, select the Build
and inspect the "Console Output" to see what went wrong.

Once the build succeeds, you can inspect the output by running the docker image commands on the
command line:

{% highlight bash %}
# Host: devbox.localhost
$ docker image ls
# should output something similar to the following:
#    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
#    ...
#    helloworldapp-22    latest              7cb9c39823c4        5 minutes ago       157MB
#    ...
{% endhighlight %}

Note that in the above output, the image name includes the build number - you could also get fancier
and include a date/timestamp for builds, or something different altogether. In order to fully close
the loop on this testing, we should spin up a Docker container using the above built image and test
that it works as we expect:

{% highlight bash %}
# Host: devbox.localhost
$ docker run -p 4040:80 helloworldapp-22
{% endhighlight %}

Once you run the above, you should be able to navigate to the URL previously visited to see the
"Hello World!" message as previously seen:

`http://10.11.13.15:4040`

Congratulations, you've successfully built your CI/CD pipeline!

### Room for Improvement

There are obviously *MANY* improvements to make to the above. Again, this is a basics tutorial that
is not DRY in code (Jenkins pipeline script repeating variables), not optimized for usability
(`__pycache__` directory issue), and in general lacks the finesse that one should expect for a
production-grade implementation. Please use the above to build your basic understanding of how Docker,
Python, and Jenkins CI all integrate for a CI pipeline to produce Docker-based Python applications.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [How To Configure a Continuous Integration Testing Environment with Docker and Docker Compose on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-a-continuous-integration-testing-environment-with-docker-and-docker-compose-on-ubuntu-14-04)
* [Getting Started with Docker](https://docs.docker.com/get-started/part2/#define-a-container-with-dockerfile)
* [Python Import Mechanisms and sys.path/PYTHONPATH](https://docs.pytest.org/en/latest/pythonpath.html)
* [Flask Official Site](http://flask.pocoo.org/docs/1.0/)
* [How to Install Jenkins on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-16-04)
