# genesys-cloud-voice-survey

An example of how to use the native web surveys but in the voice channel. This is designed as an example only on whats possible with some creative configuration. Experience using the Genesys Cloud toolsets is required before trying this example.

## UPDATE

With the new feature release on the 15th of April with "Architect post-call actions in voice calls" you no longer need to manually transfer customers to the flow to capture the customers inputs. Detail can be found [here](https://help.mypurecloud.com/articles/clear-post-flow-action/)

The concept behind this is quite simple at a high level. Use the existing [Web Surveys](https://help.mypurecloud.com/articles/about-web-surveys/) that come with Genesys Cloud 3 license but use the voice channel to capture the feedback as DTMF values, store that data and use a [dataAction](https://help.mypurecloud.com/articles/about-the-data-actions-integrations/) to POST the result into the web survey form. This way all the existing reporting features and NPS results are reflected as normal as well as the scores being visible in the interaction view for QA.

Overview of the solution:

![](/docs/images/Voice_Surveys.png?raw=true)

    NOTE: using the voice transcription to capture a voice verbatim while is possible and I have running in this example we are first going to do the basics of capturing the values as DTMF. Hence they are marked as "OPTIONAL"

# Step 1 - Survey Form

Create a [Survey form](https://help.mypurecloud.com/articles/create-a-web-survey-form/). In this case it will contain 2 questions to gather a yes/no and NPS score. You can use this same method to create more questions as you see fit.

![](/docs/images/surveyForm.png?raw=true)

Ensure you save and publish the form to make it active.

# Step 2 - Create DataTable

Create a [Data Table](https://help.mypurecloud.com/articles/create-a-data-table/) and make sure the "KEY" is set as the "conversationId" as this is used to search and the UUID. if you are creating more questions then in this example then include additional columns to store each answer.

![](/docs/images/dataTable.png?raw=true)

    NOTE: each type is "String". Also copy the "datatableId" from thh URL and save it for later as you will need it.

# Step 3 - Create DataActions

So for this basic version without voice verbatium you will need to create data actions to do the following:

- Create dataTable
- Update dataTable
- Trigger workFlow
- GET survey
- POST survey results

1 - Import the first one ["Create-DataTable-Voice-Surveys-record"](/docs/dataAction/Create-DataTable-Voice-Surveys-record.json) as a "Genesys Cloud Data Action" type as this is an internal API call and change the request URL to be your dataTableId that you created in step 2:

![](/docs/images/createDataTable.png?raw=true)

    Then save and publish

2 - Import the second one ["Update-DataTable-Voice-Surveys-record"](/docs/dataAction/Update-DataTable-Voice-Surveys-record.json) as a "Genesys Cloud Data Action" type as this is an internal API call and change the request URL to be your dataTableId that you created in step 2 like above.

    Then save and publish

3 - Import the third one ["Trigger-WorkFlow"](/docs/dataAction/TriggerWorkFlow.json) as a "Genesys Cloud Data Action" type as this is an internal API call. Nothing needs to be changed on this one as long as the KEY of the variable names are the same as in my example of "conversationId".

    Then save and publish

4 - Import the forth one ["Get-Survey"](/docs/dataAction/Get-Survey.json) as a "Genesys Cloud Data Action" type as this is an internal API call. Nothing needs to be changed on this either.

    Then save and publish

5 - Import the fifth one ["Post-Survey-Result"](/docs/dataAction/Post-Survey-Result.json) as a "Genesys Cloud Data Action" type as this is an internal API call. This one will require the "Request Body Template" to be changed to the survey form that you created. But for now leave it as is because there is a very nice trick you can do to get the JSON payload required for your specific form regardless of how you have created the questions.

    For now SAVE AS DRAFT !!!

# Step 4 - Create Survey Invite Flow

Create a new "Survey Invite Flow" then import the [voice_survey_invite](/docs/flow/voice_survey_invite_v1-0.i3SurveyInviteFlow) this is based on the above use case of 2x questions that matches our survey form. If your doing more questions then of course the flow will need to reflect that change. When you import the flow you will need to update the DataActions to reflect the ones that you imported above, the data in the options will stay.

![](/docs/images/surveyInviteFlow.png?raw=true)

This flow is first "getting" existing data in the data table (as if you don't get these it will send in blank values) then it will update the table with details such as the "surveyURl" that is created. Ensure that you put a valid email address in the last block. I suggest sending it to your own email address and creating a filter to file them away.

    THIS NEEDS TO BE A VALID EMAIL ADDRESS !!! if its not your service will get blocked long term as to many bounce backs cause issues.

![](/docs/images/surveyInviteEmail.png?raw=true)

Publish the flow.

# Step 5 - Create Policy

Build up a policy with the required filters you need for example based on queue and wrapup codes etc. Then target the Survey flow and form that you have already created.

Now put in a interaction to ensure that the policy is working and you receive the email with the survey link. NOTE you wont get the Survey IVR yet we are just testing the policy and survey link. Open your email and click on the survey link.

    This is where we get the data to edit the data action

Before you click on the button to send the results open the browser "Devtools" and go to the "Network tab" here you will get to see the JSON payload required to score the survey that we can directly use in the data action when we send the POST to score the interaction:

![](/docs/images/networkTrace.png?raw=true)

I have redacted the data in this screen shot but if you go to the "Request Payload" and view "parsed" you will see the raw JSON and the format that has been used to score the survey. Copy this raw JSON payload and use it to update the data action "Post-Survey-Result" with the data. Where you have answered the question replace the answer with the input variable.

![](/docs/images/updateAction.png?raw=true)

Here you will paste the raw JSON you got from the browser trace and replace the answer with the input variables. Your questionIds and formatting will be different.

Once this is updated Save and Publish the dataAction.

# Step 6 - Create WorkFlow

Create a new "Workflow" then import the [voice_survey_workflow](/docs/flow/voice_survey_workflow_v1-0.i3WorkFlow) this like above is based on the 2x questions that are being used in this example. The first object to "Wait 1min" is so that the system has time to trigger the policy and gather the scores from the IVR. If you have a longer ACW you will need to make this longer as well as if the number of questions are longer then 1min to answer.

the next 2x objects are to get the survey as this needs to be "opened" to score the survey then finally sending the score. When sending the score depending on the question type will depend on weather you need to send it as a "String" or "Integer" ... if you look at the JSON payload you gathered form the browser this will tell you the format and you will need to ensure that the data action is also set to POST it in the same format. The main issue people run into is trying to send a integer as a string and then the data action will error instead of scoring the survey.

![](/docs/images/workflowSend.png?raw=true)

    NOTE the variables can be converted to Int or left as strings.

You will now want to publish the workflow and save the workflowId in the URL

# Step 7 - Create Inbound Voice Flow

Create a new "Inbound Call Flow" then import the [voice_survey_flow](/docs/flow/voice_survey_flow_v1-0.i3InboundFlow) this is based on the above use case of 2x questions that matches our survey form. If your doing more questions then of course the flow will need to reflect that change. When you import the flow you will need to update the DataActions to reflect the ones that you imported above, the data in the options will stay.

![](/docs/images/voiceFlow.png?raw=true)

so the only item you will need to edit in the flow is the "flowId" to trigger the workflow

![](/docs/images/triggerWorkFlow.png?raw=true)

# Step 8 - Create external contact & Button

## UPDATE

**_As mentioned now with the new feature release [here](https://help.mypurecloud.com/articles/clear-post-flow-action/) you don't need to transfer customers to the flow anymore._**

![](/docs/images/post-flow-action-example.png?raw=true)

So now that all the pieces are in place we need to make it easy for the agent to transfer the customer to the survey. You could make a extension and hijack the "disconnect button" but for the scope of this example we sill use a simple transfer option.

Create a new external contact and use the "other" option to save the voice survey flow ID as the number followed by @localhost.com

![](/docs/images/externalContact.png?raw=true)

    flowId@localhost.com

Save this contact, This will enable the agent to easily select transfer and type in the name eg: "Voice Survey" and simply blind transfer the customer to the flow.

Inside an agent script you can also create an transfer button using the same method. Replace the ID with your own flowId of course like above.

![](/docs/images/agentScript.png?raw=true)

# Reporting

Now you will see interactions getting survey data stored against them as well as in the interaction view the actual survey form filled out.

![](/docs/images/interactions.png?raw=true)

![](/docs/images/interaction.png?raw=true)

In my screen shot above I gathered the NPS and the a voice verbatim. This example is based on basic DTMF use cases. To gather the voice and transcribe it this is using the native Genesys Cloud transcription service and adding in some additional data actions in the flows to get the transcript and use that to answer a free form comment field in the survey form. Once you get the basic DTMF going have a go at creating these additional data actions for voice transcription if you like more of a challenge.

    NOTE the final step in the image is to "DELETE" the data table record this is optional and once the table fills up the oldest record will be erased. For the simple example I have not included that optional step as well.
