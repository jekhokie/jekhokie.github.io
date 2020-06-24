---
layout: post
title:  "Apache Solr for Web Search"
date:   2020-06-23 22:13:00 -0400
categories: solr docker flask docker-compose
logo: solr.jpg
---

[Apache Solr](https://lucene.apache.org/solr/) is an open source search platform that enables fast search and filtering on data sets. This
tutorial is a very basic introduction to Apache Solr and demonstrates its capabilities by running a simple Python Flask web application that
enables the user to perform a search against a Solr back-end and return the result to the web page. It is intended to serve as a first
stepping-stone introduction to Apache Solr that can lay the groundwork for future work and improvements.

### Disclaimer

Per the usual pattern, do not attmept to use this functionality as-is in a production setting. There are many aspects (security, code
formatting, edge condition testing, etc.) of this setup that are sub-optimal but are done in a rapid development fashion to enable focusing
on the various pieces of functioanlity that are being learned.

### Prerequisites

This tutorial primarily relies on the Docker engine and `docker-compose`. Before proceeding, ensure you have a Docker engine running and
`docker-compose` installed on the device you wish to use for this tutorial.

### Project Folder

First, let's get the root project folder set up for our experiment:

```bash
# construct root project folder
$ mkdir solr-demo
$ cd solr-demo/
```

### Flask Application

The search app will be a Python Flask web-based application with a simple search field that enables running a query against the Solr back-end.
We will create the Docker image for this application first, which will then be used in the `docker-compose.yaml` file (specified later) to
create and orchestrate the interaction between the web application and the Solr search platform. Let's create the main project directory for
flask and populate it with a `requirements.txt` which will specify the required libraries we need:

```bash
# create flask app folder
$ mkdir flask-app
$ cd flask-app/

# create the templates folder to be used
# later in this tutorial
$ mkdir templates

# specify libraries required for app
$ vim requirements.txt
# ensure contains the following:
#   flask
#   simplejson
```

Next, we'll create the main Flask application functionality. Create a file named `app.py` containing the following code, which handles
incoming requests and attempts to query the Solr back-end (if a query is submitted):

```python
from flask import Flask, render_template, request
from urllib.request import urlopen
import simplejson

app = Flask(__name__)

BASE_PATH='http://solr:8983/solr/demo/select?wt=json&df=name&rows=250&q='

@app.route('/', methods=["GET","POST"])
def index():
    query = None
    numresults = None
    results = None

    # get the search term if entered, and attempt
    # to gather results to be displayed
    if request.method == "POST":
        query = request.form["searchTerm"]

        # return all results if no data was provided
        if query is None or query == "":
            query = "*:*"

        # query for information and return results
        connection = urlopen("{}{}".format(BASE_PATH, query))
        response = simplejson.load(connection)
        numresults = response['response']['numFound']
        results = response['response']['docs']

    return render_template('index.html', query=query, numresults=numresults, results=results)

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

As you'll see in the above code, the main route handles both `GET` requests (simple request for the web page) as well as `POST` requests
where it is expected that the request contains the query string that the user wishes to search the Solr search platform for. Once
records are retrieved, the results are returned and rendered in the template `index.html`.

One other important thing to note is the construction of the `BASE_PATH`, which uses a hostname of `solr`. This is possible because the
service name for Solr within the context of our Docker network will be automatically resolvable by the `flask-app` once deployed using
`docker-compose`.

Let's create the template file to be rendered as `templates/index.html` containing the following:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css"
                           integrity="sha384-9aIt2nRpC12Uk9gS9baDl411NQApFmC26EwAOH8WgZl5MYYxFfc+NcPb1dKGj7Sk" crossorigin="anonymous">

    <title>Flask Solr Tutorial</title>
  </head>
  <body>
    <div class="container">
      <h1>Flask Solr Tutorial!</h1>

      <form class="form-inline" action="/" method="post">
        <div class="form-group mx-sm-3 mb-2">
          <input type="text" class="form-control" name="searchTerm" value="{% if query %}{{ query }}{% endif %}" placeholder="Enter search term(s)">
        </div>
        <button type="submit" class="btn btn-primary mb-2">Search</button>
      </form>

      <div class="numresults" style="font-weight: bold;">
        {% raw %}{% if numresults is not none %}{% endraw %}
        Number of Results:
        <span style="margin-left: 12px;">{% raw %}{{ numresults }}{% endraw %}</span>
        {% raw %}{% endif %}{% endraw %}
      </div>

      {% raw %}{% if results and results|length > 0 %}{% endraw %}
        <table class="table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Name</th>
              <th>In Stock?</th>
              <th>Price</th>
            </tr>
          </thead>
          <tbody>
            {% raw %}{% for document in results %}{% endraw %}
              <tr>
                <td>{% raw %}{{ document['id'] }}<{% endraw %}/td>
                <td>{% raw %}{% if document['name'] %}{{ document['name'][0] }}{% endif %}<{% endraw %}/td>
                <td>{% raw %}{% if document['inStock']%}{{ document['inStock'][0] }}{% endif %}<{% endraw %}/td>
                <td>{% raw %}{% if document['price']%}${{ document['price'][0] }}{% endif %}<{% endraw %}/td>
              </tr>
            {% raw %}{% endfor %}{% endraw %}
          </tbody>
        </table>
      {% raw %}{% endif %}{% endraw %}
    </div>
  </body>
</html>
```

You'll notice in the above that certain specific fields are pulled out of the result set. These fields are present in the example core that
will be instantiated and populated as part of the section dealing with linking the Flask app and the Solr search platform later in this
tutorial.

### Create Docker Image

Now that we have a working Flask application (you can test it locally by running `python app.py` and attempting to visit the page, but the
search functionality will not work because we do not yet have a Solr search platform stood up and configured), we will package the
application as a Docker image. Create a `Dockerfile` with the following contents in the Flask application directory `flask-app`:

```bash
FROM python:3-alpine

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "./app.py" ]
```

Next, create the Docker image via the following command:

```bash
$ docker build -t flask-app:latest .
```

You can inspect that the image was created by running the following command, which should print out the image
details:

```bash
$ docker image ls flask-app
```

### Linking Solr and Flask Application

We'll now configure a `docker-compose.yaml` file that will enable us to launch and interact with our application, which will communicate with
a back-end Solr search platform running in a container and seeded with demo data. Create a file in the root project directory
`solr-demo/docker-compose.yaml` with the following contents:

```yaml
version: '3.7'

services:
  flask-app:
    image: flask-app:latest
    container_name: flask-app
    ports:
      - 5000:5000
    networks:
      - solr
    depends_on:
      - solr

  solr:
    image: solr:8.5
    container_name: solr
    ports:
      - "8983:8983"
    networks:
      - solr
    command:
      - solr-demo

networks:
  solr:
```

The above file creates a network (for connectivity between the applications), launches the Solr search platform, and launches the Flask application.
Let's go ahead and create this ecosystem:

```bash
$ docker-compose up
```

It may take a minute or two for Solr to come up cleanly - once it does, you can visit the `flask-app` web page by visiting `http://localhost:5000`
in a browser. Perform a search (something like `*:*`, which should return all results) and you should see results populated on the page when you
click "Search", indicating that your application is working correctly with the Solr search platform on the back end!

As a note, you can also visit `http://localhost:8983` to see the Apache Solr web interface directly, including configuration information and other
information about the demo core that was automatically created as part of the bootstrap process.

### Conclusion

There is still a lot of content and capabilities to discuss around Apache Solr as a search platform, as well as many improvements in the code
and flow demonstrated in this post. Feel free to use this test layout to expand your knowledge and learning of the Apache Solr platform,
including experimenting with Solr Cloud and clustering using ZooKeeper for high availability and other beneficial features.

### Credit

The above tutorial was pieced together with some information from the following sites/resources:

* [Apache Solr](https://lucene.apache.org/solr/)
* [Docker Solr](https://github.com/docker-solr/docker-solr/blob/master/README.md)
* [Docker Compose Solr - Examples](https://github.com/docker-solr/docker-solr-examples/blob/master/docker-compose/docker-compose.yml)
* [Apache Solr - Basic Commands](https://www.tutorialspoint.com/apache_solr/apache_solr_basic_commands.htm)
* [Docker Compose Example](https://gist.github.com/makuk66/0812f70b77aa92230c203cec41acac64#file-docker-compose-yml-L64)
* [Solr Cloud Issue](https://github.com/docker-solr/docker-solr/issues/183)
