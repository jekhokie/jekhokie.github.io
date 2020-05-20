---
layout: post
title:  "SumoLogic ADFS Auth with Auth0"
date:   2016-10-25 14:38:21 -0400
categories: sumologic auth0 adfs saml authentication
logo: sumologic.jpg
---
Steps to integrate [Auth0](https://auth0.com/) with the [SumoLogic](https://www.sumologic.com/) cloud-hosted
analytics platform. This tutorial specifically focuses on a typical [ADFS](https://msdn.microsoft.com/en-us/library/bb897402.aspx)
integration, including steps to auto-provision authenticated users with a default role and pre-populated
attributes from their Active Directory account.

### Background

There are a few sources of documentation that walk through setting up Active Directory Federation Services
Single Sign-On for SumoLogic. However, if you wish to use a service provider such as Auth0 to leverage
a multi-auth solution for users to access your SumoLogic account, it becomes increasingly difficult to
figure out the correct knobs to turn, and requires quite an in-depth knowledge of the SAML standard.

This tutorial compiles a succinct list of steps required to configure authentication through Auth0 using
an ADFS back-end for logging into SumoLogic. It also explains how to configure auto-provisioning of
accounts that authenticate through Auth0/ADFS, with default attributes for first name, last name, and
email address which are populated from the Active Directory back-end which served the auth request.

### Prerequisites/Assumptions

In order for these steps to work as expected, you must have an Enterprise Account (at the time of this
post). Additionally, you must also have an Auth0 account. If either of these are not true, your mileage
may vary with regards to these steps resulting in a fully-functional solution.

### Auth0

First, log into your Auth0 account and create a client application - follow the instructions on the Auth0
support site for instructions on how to do this. Take note of the following attributes for the newly-created
client application as they will be used in the SumoLogic setup:

- **Domain**: will map to the "Issuer" attribute of the SumoLogic SAML configuration.
- **Advanced -> Endpoints -> SAML Protocol URL**: will map to the "Authn Request URL" attribute of the
SumoLogic SAML configuration. 
- **Advanced -> Certificates -> Signing Certificate**: will map to the "X.509 Certificate" attribute of the
SumoLogic SAML configuration.

### SumoLogic - Configure SAML

As an administrator, log into your SumoLogic account. Navigate to the *Manage -> Security* menu option.
Once there, you will see a blue *SAML* button (at the time of this post) in the upper right corner -
click it:

[![Sumo SAML Button][0]][0]

A modal will launch with many various inputs. Fill in the following inputs at a minimum:

[![Sumo SAML][1]][1]

- **Configuration Name**: Any name that will help you remember that this is an Auth0 ADFS integration.
- **Issuer**: Fill this in with "urn:<Domain>", where "<Domain>" is the **Domain** attribute from the Auth0
configuration as explained in the **Auth0** section above.
- **Authn Request URL**: Fill this in with the **SAML Protocol URL** attribute from the Auth0 configuration
as explained in the **Auth0** section above.
- **X.509 Certificate**: Fill this in with the **Signing Certificate** attribute from the Auth0 configuration
as explained in the **Auth0** section above.
- **Email Attribute**: Select the **Use SAML attribute** radio button and specify the value
- **http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress**. This will ensure that the SAML
response attribute **emailaddress** will be used to pre-populate the email attribute of the incoming user.
If **emailaddress** is not the attribute used in your SAML response document, change the tail-end of the URL
to reflect the actual attribute that is returned from your SAML provider.

Once you have specified the above, your SumoLogic endpoint is configured for Single Sign-On. Click the
blue **Save** button in the lower corner of the modal to save your configurations.

Once the configuration is confirmed saved, re-open the SAML configuration by again clicking on the blue
**SAML** button in the upper right corner of the Security screen. Once open, take note/copy the value
inside the **Authentication Request** input - this will be used in the final Auth0 configuration section
following.

[![Sumo Auth URL][2]][2]

### Auth0 - Final Configuration

Log back into your Auth0 account and navigate to the client application being used for the SumoLogic
SSO integration. In the **Allowed Callback URLs** attribute, paste the value captured from the SumoLogic
section above titled **Authentication Request**.

[![Auth0 Callback URL][3]][3]

Once the callback URL has been configured, navigate to the application's **Addons** tab, and click the
slider to enable the **SAML2 Web App** option - this will open a modal containing a request for the
**Application Callback URL**:

[![Auth0 SAML Enable][4]][4]
[![Auth0 SAML Config][5]][5]

In this field, paste the exact same URL as placed in the previous setting
**Allowed Callback URLs**, and ensure the **Settings** section below is completely commented out except
for the opening and closing curly braces (should be this way by default). Click the blue **Save** button
at the very bottom of the modal.

### Test

Now that you have configured the respective SumoLogic and corresponding Auth0 attributes, you can
test your SSO solution. In order for this to work with the way this is currently configured, you must
first manually create an account of the user you wish to test authentication for - this should be a
valid user out of your Active Directory domain, and the email address MUST match **EXACTLY** to the
email address that is reported by the Active Directory SAML response.

Follow the SumoLogic instructions for how to create new users. Once you have created a test user,
again take note of the **Authn Request URL** in the SAML properties in SumoLogic (reference the
above "SumoLogic - Configure SAML" instructions/steps above). Log out of your current SumoLogic
session.

Open a new browser/incognito session and navigate to the **Authn Request URL** that you copied from
the SAML configurations. Once navigating to the location, you should be prompted with an Auth0
login prompt - use the Active Directory credentials for the test account created to log into the
application. If all goes well, you should be successfully logged into the SumoLogic application
with the account specified!

### User Friendly URL

It is unlikely that users will appreciate having to remember and type in the **Authn Request URL**
into their browser each and every time they wish to access the application. To foster greater
adoption and help users, it is likely a better idea to create a very simple DNS record that
resolves to an endpoint/handler which redirects to the **Authn Request URL** so that users can
easily remember the endpoint. Consider creating something simple, such as "sumologic.my.domain"
to ensure users can easily remember how to access and utilize the SumoLogic service.

### Optional - Auth Account Provisioning

The previous steps detailed how to set up an ADFS integration with SumoLogic through Auth0. However,
in the above setup, it is up to an administrator to manually configure accounts that should have
access to the main account via manually creating users in SumoLogic that have the exact email address
as the user within Active Directory. This is sometimes desirable if you wish to control costs and
overall account usage for billing purposes.

However, in the interest of self-service, it is likely more desirable to auto-create user accounts
when they successfully authenticate through Auth0. We can configure SumoLogic to auto-create
any authenticated accounts as valid users that immediately have access to the application - these
steps will detail how to configure such a setup.

***WARNING***: This will create a new SumoLogic account for **ANYONE** authenticating into SumoLogic
with a valid Active Directory account - this will likely impact your total users count and, possibly,
your overall billing and usage.

Navigate to the SumoLogic site and log in with your administrator account. Once logged in, navigate
to **Manage -> Security**, and click the blue **SAML** button in the upper right corner. Once the modal
opens, if you've followed this tutorial, you should see a summary related to your Auth0 SAML
configuration - click the blue **Configure** button to open details.

In the available options, there is a checkbox next to "On demand provisioning (Optional)". Select/
check this check box, and fill in the corresponding attributes:

- **First Name Attribute**: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname
- **Last Name Attribute**: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname
- **On Demand Provisioning Roles**: analyst

[![Sumo SAML Provision][6]][6]

The above configuration settings will use the **givenname** attribute returned from the SAML response
to create the user "First Name", and the **surname** attribute to create the "Last Name". If these are
not the actual attributes returned from your SAML provider, adjust the schema references to
reflect the actual attributes desired. If you are unsure about what the SAML response looks like/how
it is formatted, you can either request an example from your identity provider or, alternatively,
investigate the format yourself. [This](http://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_saml_view-saml-response.html)
post on the Amazon AWS docs site gives a good explanation of how to obtain an actual SAML response
for investigation.

The **On Demand Provisioning Roles** parameter specified as **analyst** means that any newly-created user
will automatically receive a role of "analyst". You can see more information about the SumoLogic
roles [here](https://help.sumologic.com/Manage/Users_and_Roles).

Now that you have configured the above, click the blue **Save** button. To test, attempt to access the
SSO endpoint for your account and log in with a user that does not yet have an account in the "Users"
section of your SumoLogic account. If successful, your user should be automatically created as a
valid SumoLogic user with the "First Name", "Last Name", and "Email" attributes specified, as well
as the "analyst" role assigned.

### Credit

Much of the above was compiled with help from the SumoLogic support staff - specifically, Kevin Keech.
Additionally, the Auth0 team was instrumental in providing sample documentation, and also explained
that they are currently working on an integration solution between Auth0 and SumoLogic, which I'm
assuming will make this entire post obsolete (hopefully, because these steps are not exactly trivial).

[0]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-sumo-saml-button.png
[1]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-sumo-saml.png
[2]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-sumo-auth-url.png
[3]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-auth0-callback-url.png
[4]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-auth0-saml-enable.png
[5]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-auth0-saml-config.png
[6]: /assets/images/2016-10-25-sumologic-adfs-auth-with-auth0-sumo-saml-provision.png
