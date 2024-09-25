# ServiceNow integration with Ansible Automation Platform

ServiceNow (SN) can act as a Service Catalog on top of Ansible Automation Platform (AAP).

Let's see how we can easily integrate those two platforms together!

In this guide, I'll create an entry in the service catalog allowing customers to order Windows or Red Hat Enterprise Linux virtual machines, in 3 different sizes. I also give the option to request monitoring and backup for the requested machine.

The request will automatically be approved in SN and start an AAP workflow with parameters (extra variables) which subsequently will trigger the appropriate playbooks. The goal of this guide is to demonstrate the integration between SN and AAP and not the playbooks, you're in charge of developing your playbooks and workflows according to your automation needs.

As an extra bonus, the last section of this guide explains how to order a service from the ServiceNow mobile app.

## ServiceNow and Ansible Automation Platform instances

If you don't have a working ServiceNow environment, create a developer instance at https://developer.servicenow.com

My ServiceNow instance is `https://devABCXYZ.service-now.com/`

My Ansible Automation Platform instance is `https://aap.example.org/` and has a valid certificate.

It is available and reachable without use of ServiceNow MID servers. MID Servers might require extra steps for the connection.

## Compatibility

This guide has been tested with AAP 2.4 and ServiceNow release "Xanadu".

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

ServiceNow no longer accepts self-signed certificate by default. There is probably a way to force accepting self signed but this is outside of the scope of this doc.

I expect your AAP to have a valid cert.


### Playbook or workflow

In this guide, I do not dive into the playbook that runs after being called by ServiceNow.

In this example, I'll run a workflow that expects the following extra variables.

```yaml
operating_system: [Windows|RHEL]
size: [small|medium|large]
monitoring: [true|false]
backup: [true|false]
```

Adapt your REST message and workflow script accordingly with the parameters you need to pass from ServiceNow to AAP.

## Prepare ServiceNow

> [!NOTE]  
> The application registry allows you to create and maintain an oAuth connection between ServiceNow and Ansible Automation Platform

Go to ServiceNow > System OAuth > Application Registry

Click New

Choose "Connect to a third party OAuth Provider"

- Name: AAP
- Client ID: paste the client ID obtained earlier
- Client Secret: paste your client secret obtained earlier
- Default grant type: Authorization code
- Unlock the field for edit Authorization URL: https://aap.example.org/api/o/authorize/
- Unlock the field and edit Token URL: https://aap.example.org/api/o/token/
- Leave the Token Revocation URL blank
- Redirect URL should already be configured: https://devABCXYZ.service-now.com/oauth_redirect.do
- Send Credentials: As Basic Authorization Header

Click Submit

Go to the "AAP" Application Registry you just created

Click "Oauth Entity Scopes" tab.

Double click on Insert a new row...

Give it a name "Writing Scope" and click the green tick

Right click on the top bar Application Registries and click Save

Click on the newly created "Writing Scope"

Set OAuth scope to "write"

Click Update

Click on "AAP default_profile" on the "OAuth Entity Profiles" tab (AAP is the name you gave to the application registry).

Double click on "Insert a new row" under OAuth Entity Scope.

Type "Writing Scope" (it should auto fill) and click the green tick

Click Update

Go to ServiceNow > System Web Services > Outbound > REST Message

Click New

> [!NOTE]
> The endpoint below points to the workflow template ID 54. You can find the ID of your AAP workflow or jobs in the URL when editing the workflow/job. Replace the ID accordingly.

- name: AAP
- endpoint: https://aap.example.org/api/v2/workflow_job_templates/54/launch/
- Authentication type: OAuth 2.0
- OAuth profile: AAP default_profile

Right click the top bar and click Save

Click "Get OAuth Token"

Click Authorize in the AAP popup window that appears.

If everything goes fine you should get a blue bar at the top of the screen saying "OAuth Refresh token is available and will expire at _some date in the future_"

> [!IMPORTANT]  
> If you get an error at this stage, you need to re-check everything in Application Registry (ServiceNow) and Application (in AAP) are correct.

Click New in regard to the "HTTP Methods" at the bottom of the screen

- Name: VM
- HTTP method: POST
- Endpoint: https://aap.example.org/api/v2/workflow_job_templates/54/launch/

Click HTTP Request

Double click "Insert a new row..." under HTTP Headers

Type "Content-Type" and click the green tick

Double click the Value field for "Content-Type"

Type "application/json" and click the green tick

Add extra vars under "HTTP Query Parameters" in the Content field.

```json
{
  "extra_vars":
  {
    "operating_system": "${operating_system}",
    "size": "${size}",
    "monitoring": ${monitoring},
    "backup": ${backup}
  }
}
```

> [!IMPORTANT]  
> The monitoring and backup variables are not surrounded by double quotes. Check boxes in SN are passed as booleans (true/false).

Click Submit

Click Auto-generate variables. This will generate variable substitutions below.

You can set test values for variable substitutions.

Right click the upper menu and save.

Click "Preview Script Usage" and save the script somewhere safe, we'll need it later.

It should look something like this (it contains the test values I have specified):

```
 try { 
 var r = new sn_ws.RESTMessageV2('AAP', 'VM');
 r.setStringParameterNoEscape('operating_system', 'Windows');
 r.setStringParameterNoEscape('size', 'medium');
 r.setStringParameterNoEscape('monitoring', 'true');
 r.setStringParameterNoEscape('backup', 'false');

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

You can now validate the integration by clicking "Test".

In the menu that appears you should see HTTP status 201.

Go to Ansible Automation Platform, you should see your workflow running.

## Create the ServiceNow workflow

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

Paste the code you have obtained earlier when you clicked "Preview Script Usage".

Adjust the script so it looks like this:

```
try { 
 var r = new sn_ws.RESTMessageV2('AAP', 'VM');

 var operating_system = current.variables["operating_system"];
 var size = current.variables["size"];
 var backup = current.variables["backup"];
 var monitoring = current.variables["monitoring"];

 r.setStringParameterNoEscape('operating_system', operating_system);
 r.setStringParameterNoEscape('size', size);
 r.setStringParameterNoEscape('backup', backup);
 r.setStringParameterNoEscape('monitoring', monitoring);

 var response = r.execute();
 var responseBody = response.getBody();
 var httpStatus = response.getStatusCode();
}
catch(ex) {
 var message = ex.message;
}
```

I added 4 var lines in the script. Those allow to pass the parameters of the requested VM to the REST call.

Click Submit

Link the Begin block to the Run script block by dragging the mouse between both blocks.

Link the Run script block to the End block

Click the Hamburger menu (3 lines icon in upper left corner)

Click Publish

Leave the editor

## Create the catalog item

Go to ServiceNow > Service Catalog > Catalog Definitions > Maintain Items

Click New

- name: Virtual Machine
- Catalog (unlock first): Service catalog
- Category: Hardware

Click Process Engine tab

- Flow: leave blank
- Workflow: Provision VM (it should auto fill)

Click Submit

Choose "Fully automated" under Fullfillment automation level.

Create 4 variables under the variables tab.

Create two multiple choices for operating system and size.

Create two checkboxes for monitoring and backup agent.

Click Update at the top.

## Order a service from a web browser

Go to ServiceNow > Self-service > Service Catalog

Choose Hardware

Click "Virtual Machine"

Click Order now

Go to AAP > Views > Logs

You should see the workflow running.

## Order a service from the ServiceNow mobile app

Install the ServiceNow app called "Now Mobile"

https://apps.apple.com/us/app/now-mobile/id1469616608

https://play.google.com/store/apps/details?id=com.servicenow.requestor&hl=en&gl=US&pli=1

Log into your instance on your mobile app

In your web browser, go to ServiceNow > System Mobile > Now Mobile App > Navigation Bar

Click on Now Mobile Nav

Click Create New Tab

- Navigation bar: Now Mobile Nav
- Launcher Screen: Services

Click Submit

Refresh your Mobile app, you should now see a "Services" section appear in the lower menu.

Go to Mobile App > Services > Hardware > Others > Virtual Machine

Click "Order now"

Click "Checkout"

In your web browser, go to AAP > Views > Logs

You should see your workflow running.