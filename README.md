# ServiceNow integration with Ansible Automation Platform

ServiceNow can act as a Service Catalog on top of Ansible Automation Platform (AAP).

Let's see how we can integrate those two platforms together.

As an extra bonus, the last section of this how to explains how to order a service from the ServiceNow mobile app.

## ServiceNow instance

If you don't have a working ServiceNow environment, create a developer instance at https://developer.servicenow.com

My ServiceNow instance is https://devABCXYZ.service-now.com/

Search and replace `devABCXYZ` with your instance name.

My Ansible Automation Platform instance is https://aap.example.org/

Ansible Automation Platform has been configured with the demo content from https://github.com/sebw/Automate-AAP2. It contains a workflow that will be started in this demo.

## Compatibility

This has been tested with AAP 2.2 and ServiceNow release "Tokyo.


## Prepare AAP

### oAuth2

Go to AAP > Administration > Applications

Click Add

- Name: ServiceNow
- Organization: Default
- Authorization grant type: Authorization code
- Redirect URIs: https://devABCXYZ.service-now.com/oauth_redirect.do
- Client type: Confidential

Save the Client ID and Client Secret, they will be needed later.

Go to AAP > Settings > Miscellaneous Authentication Settings

Click Edit

Turn on "Allow External Users to Create OAuth2 Tokens"

Click Save

### Certificate

ServiceNow no longer accepts self-signed certificate by default. I expect your AAP to have a valid cert.

SSH into one of your AAP controller nodes.

Copy and save the content of `/etc/tower/tower.cert`, it will be needed later.

It looks something like this:

```
-----BEGIN CERTIFICATE-----
MIIFTTCCBDWgAwIBAgISA6rmcIOanT+pz9dqEvzaSVutMA0GCSqGSIb3DQEBCwUA
MDIxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQswCQYDVQQD
EwJSMzAeFw0yMjExMjMwNzI4MjhaFw0yMzAyMjEwNzI4MjdaMC4xLDAqBgNVBAMT
I3N0dWRlbnQxLnJoODY3Mi5leGFtcGxlLm9wZW50bGMuY29tMIIBIjANBgkqhkiG
[...]
6xJDmmurDLmOnOH/SVQJD2Dg/5XoYJ0Yy3pYD5u/g19SwN+PYBmMzbWVPg3M4lOr
B3RfzUxSTyKG3GeQ0Hkrawlb/nlrliZ1YJz0WCuoLETpfTZTzBljOQvMd9Z1n4tu
YRoFhHtzuTmjZ7wQebjIu1E=
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
MIIFFjCCAv6gAwIBAgIRAJErCErPDBinU/bWLiWnX1owDQYJKoZIhvcNAQELBQAw
TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IFJlc2Vh
cmNoIEdyb3VwMRUwEwYDVQQDEwxJU1JHIFJvb3QgWDEwHhcNMjAwOTA0MDAwMDAw
WhcNMjUwOTE1MTYwMDAwWjAyMQswCQYDVQQGEwJVUzEWMBQGA1UEChMNTGV0J3Mg
[...]
yK5GhDDX8oVfGKF5u+decIsH4YaTw7mP3GFxJSqv3+0lUFJoi5Lc5da149p90Ids
hCExroL1+7mryIkXPeFM5TgO9r0rvZaBFOvV2z0gp35Z0+L4WPlbuEjN/lxPFin+
HlUjr8gRsI3qfJOQFy/9rKIJR0Y/8Omwt/8oTWgy1mdeHmmjk7j1nYsvC9JSQ6Zv
MldlTTKB3zhThV1+XWYp6rjd5JW1zbVWEkLNxE7GJThEUG3szgBVGP7pSWTUTsqX
nLRbwHOoq7hHwg==
-----END CERTIFICATE-----
```

## Prepare ServiceNow

Go to ServiceNow > System Definitions > Certificates

Click New

- Name: AAP
- Format: PEM
- Type: Trust Store Cert
- PEM certificate: paste your certificate copied in the previous step

Click Submit

Go to ServiceNow > System OAuth > Application Registry

Click New

Choose "Connect to a third party OAuth Provider"

- Name: AAP
- Client ID: paste the client ID obtained earlier
- Client Secret: paste your client secret obtained earlier
- Default grant type: Authorization code
- Authorization URL: https://aap.example.org/api/o/authorize/
- Token URL: https://aap.example.org/api/o/token/
- Token Revocation URL: leave blank
- Redirect URL: https://devABCXYZ.service-now.com/oauth_redirect.do
- Send Credentials: As Basic Authorization Header

Click Submit

Go to the "AAP" Application Registry you just created

Click Oauth Entity Scopes

Double click on Insert a new row...

Give it a name "Writing Scope" and click the green tick

Right click on the top bar Application Registries and click Save

Click "Writing Scope"

Set OAuth scope to "write"

Click Update

Click on OAuth Entity Profiles

Click on AAP default_profile

Double click on Insert a new row...

Type "Writing Scope" (it should auto fill) and click the green tick

Click Update

Go to ServiceNow > System Web Services > Outbound > REST Message

Click New

- name: AAP workflow
- endpoint: https://aap.example.org/api/v2/workflow_job_templates/54/launch/
- Authentication type: OAuth 2.0
- OAuth profile: AAP default_profile

Right click the top bar and click Save

Click "Get OAuth Token"

Click Authorize in the popup window

Click New in regard to the "HTTP Methods" at the bottom of the screen

- Name: Provision VM
- HTTP method: POST
- Endpoint: https://aap.example.org/api/v2/workflow_job_templates/54/launch/

Click HTTP Request

Double click "Insert a new row..." under HTTP Headers

Type "Content-Type" and click the green tick

Double click the Value field for "Content-Type"

Type "application/json" and click the green tick

Add extra vars

```json
{
  "extra_vars":
  {
    "my_var": "This is a workflow"
  }
}
```

Click Submit

## Test connectivity

Go to ServiceNow > System Web Services > Outbound > REST Message

Click AAP workflow

Click Provision VM

Click Test

Go to Ansible Automation Platform, you should see your workflow running.

## Create a ServiceNow catalog item

Go to ServiceNow > Workflow > Workflow Editor

Click New Workflow

- Name: Provision VM
- Table: Requested Item (sc_req_item)

Click Submit

Click the line connecting the two blocks (it turns blue when clicked)

Delete the line by pressing Delete on your keyboard

Select the Core tab (upper right corner)

Expand Core Activities > Utilities

Drag "Run script" to the workflow editor

Paste this code under "Script"

```
try { 
 var r = new sn_ws.RESTMessageV2('AAP workflow', 'Provision VM');

//override authentication profile 
//authentication type ='basic'/ 'oauth2'
//r.setAuthenticationProfile(authentication type, profile name);

//set a MID server name if one wants to run the message on MID
//r.setMIDServer('MY_MID_SERVER');

//if the message is configured to communicate through ECC queue, either
//by setting a MID server or calling executeAsync, one needs to set skip_sensor
//to true. Otherwise, one may get an intermittent error that the response body is null
//r.setEccParameter('skip_sensor', true);

 var response = r.execute();
 var responseBody = response.getBody();
 var httpStatus = response.getStatusCode();
}
catch(ex) {
 var message = ex.message;
}
```

Click Submit

Link the Begin block to the Run script block

Link the Run script block to the End block

Click the Hamburger menu (3 lines icon in upper left corner)

Click Publish

Leave the editor

Go to ServiceNow > Service Catalog > Catalog Definitions > Maintain Items

Click New

- name: Provision VM
- Catalog: Service catalog
- Category: Software

Click Process Engine

- Flow: leave blank
- Workflow: Provision VM (it should auto fill)

## Order a service from a web browser

Go to ServiceNow > Self-service > Service Catalog

Choose Software

Click "Provision VM"

Click Order now

Go to AAP > Views > Logs

You should see the workflow running

## Order a service from the ServiceNow mobile app

Install the ServiceNow app called "Now Mobile"

https://apps.apple.com/us/app/now-mobile/id1469616608

https://play.google.com/store/apps/details?id=com.servicenow.requestor&hl=en&gl=US&pli=1

Log into your instance on your mobile app

In your web browser, go to ServiceNow > System Mobile > Now Mobile App > Navigation Bar

Click on Now Mobile Nav

Click Create New Tab

- Label: Services
- Launcher Screen: Services

Save

Refresh your Mobile app, you should now see a service section appear in the lower menu.

From your mobile phone, navigate to Services > Software > Provision VM

Click "Order now"

Click "Checkout"

In your web browser, go to AAP > Views > Logs

You should see the workflow running
