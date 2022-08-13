---
layout: post
title:  "Kong API Gateway"
date:   2022-08-12 19:25:00 -0400
categories: gateway infra
logo: kong.jpg
---

This tutorial walks through installing, configuring, and testing various features of the
[Kong API Gateway](https://konghq.com/products/api-gateway-platform). API gateways are pivotal to largely scaled application
architectures that support microservices patterns to support features such as authentication/authorization, rate limiting,
IP restrictions, etc.

## Prerequisites

This tutorial assumes you have the Docker engine installed and running as the examples will use Docker to launch various
containers for use of the Kong API Gateway and supporting services.

To ensure all containers can communicate easily, let's create a Docker network on which all containers will be launched:

```bash
$ docker network create kong
```

## Launching a PostgreSQL Database

Kong uses a database to store the state and configuration information you manipulate for the instance. It is possible to launch
a Kong gateway without a database backend, but it complicates the creation of services (you'll need to manipulate configuration
files and inject them, which is clunky). So let's launch a PostgreSQL database instance that we can use for the Kong API gateway:

```bash
$ docker run -d --name kong-database \
                --network=kong \
                -p 5432:5432 \
                -e "POSTGRES_USER=kong" \
                -e "POSTGRES_DB=kong" \
                -e "POSTGRES_PASSWORD=kong" \
                postgres:9.6
```

Next, we'll want to run the database migrations to prepare the database for the Kong API Gateway instance:

```bash
$ docker run --rm \
             --link kong-database:kong-database \
             --network=kong \
             -e "KONG_DATABASE=postgres" \
             -e "KONG_PG_HOST=kong-database" \
             -e "KONG_PG_USER=kong" \
             -e "KONG_PG_PASSWORD=kong" \
             -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
             kong/kong-gateway kong migrations bootstrap
```

Once the migrations complete, you should see a completion message similar to `Database is up-to-date`, indicating you're now
ready to launch the Kong API Gateway instance.

## Launching a Kong API Gateway

Now, let's launch the Kong API Gateway instance using the PostgreSQL instance we launched:

```bash
$ docker run -d --name kong \
                --network=kong \
                --link kong-database:kong-database \
                -e "KONG_DATABASE=postgres" \
                -e "KONG_PG_HOST=kong-database" \
                -e "KONG_PG_PASSWORD=kong" \
                -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
                -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
                -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
                -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
                -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
                -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
                -p 8000:8000 \
                -p 8443:8443 \
                -p 8001:8001 \
                -p 8444:8444 \
                kong/kong-gateway
```

If the container launches successfully, you can validate the Kong API Gateway is running by issuing a curl command to the API endpoint
like so: `curl http://localhost:8001/`. If you receive a large JSON response with valid data, you can be confident your API gateway
instance is running as expected.

## Launching the Konga Admin UI

To ease the administration and testing of Kong features, we'll install the [Konga](https://pantsel.github.io/konga/) administrative
User Interface for configuring Kong.

Launch the Konga docker container:

```bash
$ docker run -d --name konga \
                --network=kong \
                -p 1337:1337 \
                pantsel/konga
```

Once the Konga Docker container is launched, you can visit the web interface in a browser by visiting
[http://localhost:1337](http://localhost:1337). From here, you'll be asked to create an administrative user - go ahead and
enter details that allow you to create the admin account. Once you click create, you'll need to log into the interface using
the same account credentials. If all goes well, you'll land on a Dashboard page that asks about connecting to your Kong instance.

Enter the details for your Kong instance - create a name for the connection (e.g. "Kong") and enter the URL of the admin API which,
if you followed the above directions verbatim, should be `http://kong:8001` (using the hostname `kong` will be recognizable by
the Konga interface by the nature of naming your Docker container that was launched within the `kong` network itself). Click
the "Create Connection" button and if all goes well, you'll land on the Dashboard with some basic metrics information about your
Kong instance.

## Launch an Echo Server for Service

Let's use a basic echo server container for configuring and testing our API gateway. Launch an echo server using the following command:

```bash
$ docker run -d --name echo-server \
                --network=kong \
                -p 3000:80 \
                ealen/echo-server
```

Issue a curl request to validate the container is working as expected: `curl http://localhost:3000/param?query=hello-world`. If all
goes well, you should receive a JSON payload which contains (among other data) the following bit: `...,"query":{"query":"hello-world"},`.

Note that we're querying the echo server on port 3000 yet binding that port to port 80. Port 80 will be used to access the echo server
by the Kong API Gateway on the `kong` network which all Docker containers are connected to.

## Configure the Echo Service

Now let's configure a Service and Route through the Konga interface. Log into Konga, and navigate to the `Services` menu item in the
left navbar. Click the `Add New Service` button, and enter the following details:

* **Name**: EchoTest
* **Protocol**: http
* **Host**: echo-server
* **Port**: 80
* **Path**: /

Click the `Submit Changes` button, which should bring you to the list of Services. Then, click on the service name `EchoTest` to
bring up the options for the Service, and select the `Routes` option for the Service. Click the `Add Route` button, and enter the
following details:

* **Name**: TestRoute
* **Paths**: `/testPath` (make sure to press "Enter" after entering this value)
* **Protocols**: `http` (make sure to press "Enter" after entering this value)

Now click the `Submit Changes` button. You now have a Service and Route defined which you can use your API gateway to route
requests to your back-end echo server service. Let's test it by issuing the following curl request:

```bash
$ curl http://localhost:8000/testPath/param?query=hello-world
```

Note above that the curl request is targeting localhost, but on port 8000 which is where the Kong API gateway is listening.
If successful, the above curl request should return a JSON response that has the following in its payload, indicating a succesful
configuration of your API gateway routing to the back-end echo server: `...,"query":{"query":"hello-world"},...`

## Plugin Testing

The power of Kong is not just in the gateway pass-through/routing but in the plugins which perform various functions. This section
explores several of the open-source (non-enterprise) plugins that can be used with the Kong API Gateway.

### API Key Authentication

To test some basic functionality of the API gateway, let's configure the API Key global plugin. In the Konga web interface, navigate
to the `Plugins` page in the left navbar. Click the `Add Global Plugins` button, and select `Add Plugin` under the "Key Auth" plugin in
the `Authentication` plugin sub-menu. This specific plugin allows the management of consumers within the Kong API Gateway itself,
which is a reasonable first test for authentication. When the modal pops open, simply navigate to the bottom and click `Add Plugin`
without changing any values.

Now that the plugin is installed, let's try to re-issue a curl request to the echo server:

```bash
$ curl http://localhost:8000/testPath/param?query=hello-world
```

You should get a response that looks like the following:

```json
{
  "message":"No API key found in request"
}
```

This indicates that we need an API key to authenticate to access the endpoint. Navigate to the `Consumers` page via the left
navbar in the Konga interface, and click the `Create Consumer` button. For `username`, specify `test-consumer`, and leave the rest
of the fields default/click the `Submit Consumer` button. By default (as seen in the `Routes` view for the consumer created) the
consumer is permitted to access the `TestRoute` route for our echo server.

Next, for the consumer, click on the `Credentials` navigation, followed by the `API Keys` sub-navigation. Then, click the `Create
API Key` button and click `Submit` (leave fields blank/default). You should then see an API key generated - copy this value.

Let's re-issue the same curl command but this time, pass the API key (replace `<APIKEY>` with the API Key value you copied
from the Konga interface):

```bash
$ curl --header "apikey: <APIKEY>" \
       http://localhost:8000/testPath/param?query=hello-world
```

If successful, you should be authenticated and receive the echo response as expected. Congratulations, you've now configured
authentication at your Gateway layer for your back-end echo service!

It's worth noting there is a rich set of authentication plugins available for authentication and at large scale, it's likely better
to offload the user and auth management to a proper identity provider which Kong can use vs. managing users within Kong itself.

### Rate Limiting

In order to help prevent DDoS attacks, the rate limiting plugin can be used. First, install the plugin under the `Plugins` menu in
the left navbar, and under the `Traffic Control` sub-menu by clicking the `Add Plugin` button under `Rate Limiting`. In the form that
opens, leave all values blank except the `second` value - specify `1` in the `second` field and then click the `Submit Changes` button.
This should configure a maximum request rate of 1 request per second globally (for all consumers). This plugin allows a per-consumer
rate limit, but to keep the testing easy, we'll specify a 1 per second rate limit for all consumers/request sources.

Let's test this - issue multiple curl requests in a for loop (again replacing `<APIKEY>` with your consumer API key):

```bash
$ for x in {1..10}; do curl --header "apikey: <APIKEY>" \
       http://localhost:8000/testPath/param?query=hello-world; \
done
```

What you should observe is that your first request returns a result from the echo server, but several of the subsequent requests
(depending on how fast your device issues the requests) should return a JSON response of `{ "message":"API rate limit exceeded" }`.
This indicates that our rate limiting plugin is working as expected!

### Request Transformer

Often it's useful to have requests transformed prior to forwarding to the back-end service. Let's add the Request Transformer plugin
via the Global Plugins menu under the `Transformations` sub-menu. When you click `Add Plugin`, modify the `append` -> `querystring`
field to read as `query:added`, then press "Enter" (this is important! if you don't press the enter key, the querystring won't add
the actual value). This should add a query string to the end of the requested URL. Let's try this - issue a curl command like so,
again replacing the `<APIKEY>` parameter with your actual API key. Note that we have removed the `query` parameter in the querystring,
expecting our API Gateway to add it to our request with a specific/different value than previous tests:

```bash
$ curl --header "apikey: <APIKEY>" \
       http://localhost:8000/testPath/param
```

When submitting the request, you should get back the typical JSON payload, except this time you should see the JSON contents of
`...,{"query":{"query":"added"},`, indicating the API Gateway has successfully injected the querystring in the request!

## Conclusion

The Kong API Gateway can be a very powerful component of your overall architecture. With the multitude of plugins available for
content manipulation, authentication, and security controls, Kong is sure to offer an enhanced edge to your complex ecosystem of
microservice functionality, offloading many of the likely custom functionality built into your software.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [Kong Gateway](https://hub.docker.com/r/kong/kong-gateway)
- [Konga Installation](https://github.com/pantsel/konga/blob/master/README.md#installation)
- [Kong Docker Container README](https://hub.docker.com/_/kong)

