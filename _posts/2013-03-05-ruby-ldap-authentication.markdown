---
layout: post
title:  "Ruby LDAP Authentication"
date:   2013-03-05 06:17:28 -0400
categories: ruby ldap authentication
---
Quick code snippet related to authenticating users in Ruby using an LDAP endpoint. Note
that this authentication snippet is very basic (successful bind operation is all that is
required).

## Implementation

In order for the code snippet to work, the `ruby-ldap` gem must first be installed:

{% highlight bash %}
$ gem install ruby-ldap
{% endhighlight %}

Next, the `ruby-ldap` gem can be used to authenticate a user via a simple bind operation:

{% highlight ruby %}
require 'ldap'

def ldap_authenticate(username, password)
  begin
    conn = LDAP::SSLConn.new('<LDAP_SERVER_HOSTNAME>', <LDAP_SERVER_PORT>)
    conn.set_option(LDAP::LDAP_OPT_PROTOCOL_VERSION, 3)

    return true if conn.bind("uid=#{username},ou=People,dc=example,dc=com", password)
  rescue
    return false
  end

  return false
end
{% endhighlight %}

Obviously replace `<LDAP_SERVER_HOSTNAME>` and `<LDAP_SERVER_PORT>` with your respective
environment configuration parameters.
