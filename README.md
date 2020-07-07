# Goal

Using this repo in Azure Pipelines will automatically create a web app in azure und deploy codechanges on your master branch to first a staging slot and then manually to the production slot of your web app. Be aware that this web app will be billed in azure and you should manually delete it if you do not need it anymore.

## Prerequirements

- Azure Subscription
- GitHub account

## 1. Create a DevOps Organisation

Go to dev.azure.com and create a new [organization](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/organization-management?view=azure-devops#:~:text=Related%20articles-,Azure%20DevOps%20Services,up%20continuous%20integration%20and%20deployment). Create a [project](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/about-projects?view=azure-devops) within this organization. Feel free to make yourself familiar with the different options during the creation of a project - for now we should be fine with a private project and under advanced you can set the agile process. See [here](https://docs.microsoft.com/en-us/azure/devops/boards/work-items/guidance/choose-process?view=azure-devops&tabs=basic-process) if you want to understand the different options of processes.

## 2. Import the Repo

Import this GitHub repository into an Azure Repository.

1. Go to *Repos* and press the *Import* button under import a repository.

2. You will be asked to enter the link of this GitHub repo. Confirm.

## 3. Changes to be made

Before you can use this repo some changes need to be made. Be careful not to put important keys and ids into a public repository.
In this case we have imported the repo into our private DevOps project and we are going to make the changes here. Go to *Files* under *Repos* for that.

### Changes in the template.json

Go to the *template.json* and choose to *Edit* the file as follows.
In line 30 of the ARM Template set the name of your Azure App Service. This name needs to be unique, therefore you should not use the default. I recommend using your name code: webapp(first two letters of your first name)(first two letters of your last name).

## 4. Create a Pipeline

Under *Pipelines* within your new project create a new pipeline. Choose *Azure Repos Git* as source for this pipeline. Once chosen it should derive the *azure-pipelines.yml* from your repo. Now *RUN* this pipeline. In this step it will create an artifact on which the Release Pipeline will build. Read here what the artifact is and how pipelines are build.

## 5. Create a Release Pipeline

Under *Pipelines* you will also find the tab *Releases*. Here we will build on the Artifact to create the resources in Azure and push the code - we will build our CI/CD-Pipeline.
Create a new Release Pipeline first. Give it a proper name.
A sidebar should open asking you to add jobs to the stages. Close this for now and start with adding the artifact. It should be called after your project and look like this:
**Image of Artifact settings**
Now you can add a New Stage. Name it *Deploy Azure Resources*. 

1. A sidebar will show up again. At this point create an *Empty Job*. 

2. Select *Tasks* under the stagename. With *+* you can search tasks to be created. We are going to create 3 - one being redundant but informative.

3. Search for the "Extract Files" Task. You will need to unzip the Archive to properly work with the ARM template.

Archive file patterns:
```$(System.DefaultWorkingDirectory)/<ARTIFACT_NAME>/drop/arm.zip```

Destination folder:
```$(System.DefaultWorkingDirectory)/<ARTIFACT_NAME>```

Don't forget to delete the tick at *Clean destination folder before extracting*.

4. Now you can add a *Bash Script*. This is a good option to execute custom tasks. Change the Type to *Inline* and add this commandline script:

```ls ./*/*```

You can use this to better understand how your folder structure looks so far.

5. At last add another Task. Search for *ARM Template deployment* 

|Section|Content|Description|
|-------|--------|-------|
|Deployment scope|Subscription|As our ARM template contains the creation of a Resource Group we will directly create it on subscription level.|
|Azure Resource Manager connection|your subscription|The first time you will have to Authorize the connection. As long as you are working out of the same Azure Active Directory this should be easy.|
|Subscription|your subscription|Select your azure subscription again|
|Location|North Europe||
|Template location|Linked artifact||
|Template|$(System.DefaultWorkingDirectory)/<ARTIFACT_NAME>/s/template.json||
|Template parameters|$(System.DefaultWorkingDirectory)/<ARTIFACT_NAME>/s/template.json||

6. Go back to the pipeline and create a new stage. In this stage we will push the code to the newly created Azure Web App. Therefore name the stage *App Deployment*. Here we will only add one task.
Search for *Deploy Azure App Service*.

|Section|Content|Description|
|-------|--------|-------|
|Connection type|Azure Resource Manager||
|Azure subscription|<YOUR_SUBSCRIPTION>|If you cannnot fill this within the task go to the job and fill this information in there.|
|App Service type|Web App on Windows||
|App Service name|webapp(first to letters of your first name)(first two letters of your last name)||
|Package or folder|$(System.DefaultWorkingDirectory)/<ARTIFACT_NAME>/drop/s.zip||

7. Save what you did.

8. Create a release

9. Go to https://webapp(first two letters of your first name)(first two letters of your last name).azurewebsites.net. You should now be able to see your newly deployed Azure Web Application. Go to the Azure Portal to see what was created.

## 6. Add staging

1. Go back to editing the *Release Pipeline*. Activate *Deploy to Slot or App Service Environment*, Choose the Resource Group and under Slot you will find *productive* and *staging*. Choose *staging*.

2. Now create a new Stage. Choose the Task *Manage Azure App Service - Slot Swap* and put all settings in place so the staging slot will be changed to the produciton slot.

3. In the overview of your release pipeline ad a *Pre-deployment condition* to this last stage. Enable *Pre-deployment approvals* and set yourself as approver.

4. Finally create a *Pre-deployment condition* for the first stage to be executed automatically on artifact changes through the master branch.

## 7. Delete Azure Web App


