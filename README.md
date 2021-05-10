# Power Apps - Virtual Machine Requester
![Header]

This is a demo to showcase a number of Microsoft cloud technologies working together to provide a self-service mobile app which automates the provisioning of Azure resources. It utilizes Power Apps, Flow (or Azure Logic Apps), Azure DevOps and Azure. The scenario is based on an Azure Developer environment where individuals can request a Virtual Machine to be provisioned into their shared Dev environment.

![Screenshot]

### ARM Teamplate

**azuredeploy.json**

This demo assumes that there is an existing virtual network and subnet that the requested VM will be provisioned into. If you don’t already have a suitable virtual network you’d like to use with this demo then go ahead and provision one then update the variables in the ARM Template deploy file.

![AzureDeploy]

**azuredeploy.parameters.json**

The ARM Template parameters file does not need to be changed as these values will be dynamically overriding when the DevOps Release is run.


### Azure DevOps

**Repo**

Commit the two ARM Template files into an Azure DevOps Repo.

![Repo]


**Pipeline Release**

Create a new empty Azure DevOps Release that deploys from the Repo configured in the previous stage. On Artifacts, disable the ‘Continuous Deployment’ and ‘Pull Request’ triggers as we only want the Release to be triggered when called when Flow calls the REST API.

![Release]

Add the following task to the first Stage – ‘Azure Deployment’. This task enables you to retrieve and deploy an ARM Template from your Repo. You may need to add a [Service Connection][ServiceConnection] to authenticate with your Azure subscription.

![Task]

Add the following variables into the ‘Override template parameters’ section. The ‘Name’ on the left represents the parameter as its stated in the ARM Template, the ‘Value’ on the right represents the variables that are declared within the Azure DevOps Release.

![Parameters]

Now set the DevOps Release variables to match ensuring the ‘Settable at release time’ is selected. This enables the values to be set when Flow makes the REST API call to the DevOps Release.

![Variables]

**Security - Personal Access Token**

New that you have a DevOps Pipeline setup you’ll need to generate a PAT (Personal Access Token) to enable your Microsoft Flow to authenticate with the Pipeline when making the REST API call. [Follow these steps][PAT] to create a PAT for Flow to use – you should only require ‘Release – Read, write & execute’ permissions to trigger the Release.

## Power Apps

From the Power Apps portal select ‘Import package’ then browse to the export ZIP file. This will import the ‘Azure VM Requester’ Power App as a new canvas app.

![Import]

![ImportApp]

Next create a new Entity Data source to store the data associated with the Azure VM request. This is the data source that Flow is using as a trigger when a new record is created.

![Entities]

![newEntity]

Create the following custom fields.

![Fields]

The form in the canvas app is used to capture the input and submit the values to the data source ‘VM Requests’ created in the previous step.

![CanvasApp]

A new record is created using these fields each time the form is summited using the Power Apps canvas app. Its this new record that triggers the Flow (or Logic App).

![Record]

## Microsoft Flow (or Azure Logic App)

When you imported the pre-created ‘Azure VM Requester’ Power App a new Flow should have also been created however some settings will need to be updated to use your own Power Apps environment and DevOps organization. These steps show you how to create a new Flow using the ‘Automated - from blank’ option if you need to otherwise just update with your own settings.

![Flow]

To trigger the Flow we will watch for when a record is updated in the ‘Common Data Source’ that we use in the ‘Azure VM Requester’ Power App. 

![FlowTrigger]

Configure the trigger ‘When a record is created’ as follows.
- **Environment** – The name of the Power Apps environment that houses the ‘Azure VM Requester’ app.
- **Entity Name** – The ‘Common Data Source’ name defined in the Power App.
- **Scope** – Set this to ‘Organisation’

![FlowRecord]

Add the next step – ‘Office 365 Users – Get my profile (V2)’.

![FlowO365]

Add the next step – ‘Start and wait for and approval’.

![FlowEmail]

Add the next step – ‘Condition’. Set ‘Outcome’, ‘is equal’, ‘Approve. The Flow will pause at this point and wait for the approver to either ‘Approve’ or ‘Reject’ the request.

![FlowCondition]

Add the next step for the ‘Yes’ condition. First send an email to the requester notifying them that their request has been approved.
For the ‘No’ condition you can add an email message notifying the requester that the request was not approved.

![FlowSuccess]

Add the next step – ‘HTTP’. This enables us to use a REAT API POST to trigger the Pipeline Release.
POST https://vsrm.dev.azure.com/{organization}/{project}/_apis/release/releases?api-version=5.1
- **Method** – POST
- **URL** – {organization} = your Azure DevOps Org, {project}, = your Azure DevOps Project

![FlowBody]

Next add your Azure DevOps PAT (Personal Access Token) in the ‘Advanced options’.
- **Authentication** – Basic
- **Username** – PAT name
- **Password** – PAT secret

![FlowRand]

If you’re wanting to first test the REST API call to DevOps Pipeline then you can hardcode the variable values first before using the dynamic content.
To test the [DevOps Pipeline Release REST API][DevOpsRestApi] you can use a free tool such as [Postman][Postman].


<!-- References -->

<!-- Local -->
[Header]: documentation/header.png
[Screenshot]: documentation/screenshot.png
[AzureDeploy]: documentation/azuredeploy.png
[Repo]: documentation/repo.png
[Release]: documentation/release.png
[Task]: documentation/task.png
[Parameters]: documentation/parameters.png
[Variables]: documentation/variables.png
[Import]: documentation/import.png
[ImportApp]: documentation/importApp.png
[Entities]: documentation/entities.png
[NewEntity]: documentation/newEntity.png
[Fields]: documentation/fields.png
[CanvasApp]: documentation/canvasApp.png
[Record]: documentation/record.png
[Flow]: documentation/flow.png
[FlowTrigger]: documentation/flowTrigger.png
[FlowRecord]: documentation/flowRecord.png
[FlowO365]: documentation/flowO365.png
[FlowEmail]: documentation/flowEmail.png
[FlowCondition]: documentation/flowCondition.png
[FlowSuccess]: documentation/flowSuccess.png
[FlowBody]: documentation/flowBody.png
[FlowRand]: documentation/flowRand.png



<!-- External -->
[ServiceConnection]: https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml
[PAT]: https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops
[DevOpsRestApi]: https://docs.microsoft.com/en-us/rest/api/azure/devops/release/releases/create?view=azure-devops-rest-5.1
[Postman]: https://www.getpostman.com/
