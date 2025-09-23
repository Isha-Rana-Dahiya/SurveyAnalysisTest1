# SurveyAnalysisTest1
1. create a new godaddy email account
2. change setting on godaddy admin to forward the email of survey to new go daddy account
3. test azure function locally
4. use Singapore datacenter


Resources - 
Azure storage account - blob-Training documents need to be stored in Azure blob storage container with CORS enabled for Azure document intelligence
Use Custom extraction model which is trained using GA API as they would last for 2 years and those models which are trained using preview API lasts for three months.
doc intelligence service
resource group
Create a custom extraction project document intelligence studio
refer to guidelines for labelling custom model

Forming a connection to training data 
    - will use doc intelli resource from azure
    - will link with blob storage having training data(blob container name - train docs)
  Uploaded samples of training data


Email Ingestion
1. Use Azure Logic Apps (or Power Automate) to monitor a mailbox
- Trigger: “When a new email arrives” in your survey inbox.
- Condition: Attachment file type is .pdf.
2. Extract the PDF attachment
- Save the file to a staging area (in-memory or temporary storage).


Storage & Trigger
 1. Upload the PDF to Azure Blob Storage
- Container: survey-submissions/raw/
- File name: <timestamp>_<participantID>.pdf
2.  Emit an Event Grid event on blob creation
- Topic: survey-submissions
- Event Type: Microsoft.Storage.BlobCreated


Document Processing with Azure AI Document Intelligence
1.  Azure Function (or Logic App) subscribes to the Event Grid topic
2. Inside the Function:
- Call your Custom Document Intelligence model endpoint
- Endpoint: /formrecognizer/documentModels/{modelId}:analyze
- Input: Blob SAS URL for the newly uploaded PDF
- Parse the JSON response to extract fields:
- FullName
- Question1_OptionNumber … QuestionN_OptionNumber
3. Handle batch vs. single-page logic
- Your model should be trained on single-page exports (as per previous step).

Score calculation
1. In the same Azure Function (or downstream Function):
- Convert each QuestionX_OptionNumber string to an integer
- Sum all question values into TotalScore
2. Build a structured object:
  {
  "participantId": "12345",
  "fullName": "Jane Doe",
  "scores": {
    "Q1": 2,
    "Q2": 5,
    … 
  },
  "totalScore": 27,
  "submittedAt": "2025-08-30T14:05:00Z"
}

Persistence & Notification
1. Persist the result to a database
- Azure Cosmos DB (NoSQL) or Azure SQL Database
- Container/Table: surveyResults
2. Optionally, send a confirmation or report
- Logic App or Function → send email/Teams message with totalScore
- Or push a message to a Service Bus queue for downstream processing

  Surface in Your App
1. Front-end (Web/Mobile) queries your database via a REST API
- Azure Functions or Azure API Management exposes endpoints like:
- GET /surveyResults/{participantId}
2. Display the fullName, each question score, and totalScore in your UI

Summary of proposed implementation with the azure services we need(can calculate estimated price using pricing calculator)
1. Detect incoming survey PDFs >> Logic Apps / Power Automate >> (serve as an email watcher)
2. Staging + trigger >> Blob Storage + Event Grid >> (File storage)
3. Field recognition >> Document Intelligence + Func >> (Document extraction)
4. Sum option values >> Azure Function >> (Scoring the computation)
5. Persist results >> Cosmos DB / SQL DB >> (serves as a Data Store- can be used for generating reports and analysis through integration with Power BI-can be added later)
6. Serve data to your application >> Functions / API Management >> (Postman serves as front end interface API / UI)

Workflow Overview
Participant submits survey → PDF sent to email → Logic App extracts PDF → PDF uploaded to Blob Storage → Azure Function triggers → Document Intelligence extracts answers → Scores calculated → Response returned to Postman

Created Azure account with ask@curaselects.com
- free trial with $200 credits to be used in 30 days
- 55+ services always free
- Popular services free for 12 months 
- https://portal.azure.com/#view/Microsoft_Azure_Billing/FreeServicesBlade
- After the credit is over, microsoft will ask if we would want to continue with pay-as-you-go. If we do, then we will pay if we use more than the free amounts of services. 

Custom classification model uses a different set of APIs. We need to use document classifiers classify document to call the service. And document classifiers get classify result to get the result. We will copy the model id in the studio.


Budget 60
ZRS storage account
deployed blob storage    
CORS is an HTTP feature that enables a web application running under one domain to access resources in another domain. Web browsers implement a security restriction known as same-origin policy that prevents a web page from calling APIs in a different domain. CORS provides a secure way to allow one domain (the origin domain) to call APIs in another domain. Enable CORS. 


Detailed Step-by-Step Implementation of Your Survey Response Scoring Project
Below is a comprehensive, low-code guide to ingest filled survey PDFs from your GoDaddy mailbox, extract responses via Azure Document Intelligence, map them to scores, compute S1 and S2 totals, and send the results back via email.

Prerequisites
- An Azure Document Intelligence (formerly Form Recognizer) resource with a trained or prebuilt model
- An Azure Storage Account with a private Blob container
- A scoreLookup.json file uploaded to that container
- A licensed Power Automate user account (e.g., your Azure AD ask@curaselects.com)
- GoDaddy email credentials for IMAP (incoming) and SMTP (outgoing)


Step 2: Build the Power Automate Flow
2.1 Sign In and Create Flow
- Go to https://make.powerautomate.com and sign in with your Azure AD account.
- In the left nav, select Create → Automated cloud flow.
- Give it a name (e.g., SurveyScoringFlow) and click Skip, then Create.
  
2.2 Trigger: When a New Email Arrives (IMAP)
1. Click + New step → search IMAP → select When a new email arrives (IMAP).
2. Add a new connection using your GoDaddy settings:
- Server: imap.secureserver.net
- Port: 993
- Encryption: SSL
- Username/password: your GoDaddy email and password

3. - For Folder, choose /Inbox or a dedicated folder.
4.  Under Advanced options, set Include Attachments to Yes and Only with Attachments as needed.

Get the PDF Attachment
1. - Click + New step inside the flow.
2. - Search IMAP → select Get attachment (IMAP).
3. - For Message Id, select Message Id from the trigger.
4. - (Optional) Filter on attachments ending in .pdf.
2.4 Analyze PDF with Azure Document Intelligence
1. - + New step → search Document Intelligence → select Analyze document.
2. - Choose your Azure resource and the appropriate model.
3. - For Document Content, use the PDF attachment from the prior action.
4. - Rename this action to ExtractedResponses.
  
   - 2.5 Retrieve and Parse Your Lookup JSON
1. + New step → search Azure Blob → select Get blob content.
2. Pick your storage account, container lookupfiles, and blob scoreLookup.json.
3. + New step → search Parse JSON.
4. For Content, use the file content from Get blob content.
5. Paste or auto-generate the JSON schema from your local file.
2.6 Initialize Score Variables
- + New step → Initialize variable:
- Name: S1
- Type: Integer
- Value: 0
- Repeat to initialize S2 with value 0.
2.7 Score Questions 1–12 (Compute S1)
1. + New step → Apply to each over the array
["Q1_Response","Q2_Response",…,"Q12_Response"].
2. Inside the loop:
- Compose CurrentAnswer:

body('ExtractedResponses')?[item()]

- Filter array on parsed JSON where
item()?['Response'] equals outputs('Compose_CurrentAnswer').
- Compose CurrentScore:

first(body('Filter_array'))?['Score']

- Increment variable S1 by
int(outputs('Compose_CurrentScore')).
2.8 Score Questions 13–20 (Compute S2)
1. + New step → Apply to each over
["Q13_Response",…,"Q20_Response"].
2. Repeat the same actions as in 2.7, but Increment variable S2.
2.9 Send Email with S1 and S2 via SMTP
1. + New step → search SMTP → select Send email (SMTP).
2. Create a connection using your GoDaddy SMTP settings:
- Server: smtpout.secureserver.net
- Port: 465
- Encryption: SSL
- Username/password: GoDaddy credentials
3. To: dynamic content From from the IMAP trigger.
4. Subject:
  Your Survey Scores – S1=@{variables('S1')}, S2=@{variables('S2')}
5. Body:
  Hello,

Thanks for completing our survey.
Your total for questions 1–12 (S1) is @{variables('S1')}.
Your total for questions 13–20 (S2) is @{variables('S2')}.

Regards,
Your Team

Step 3: Test, Monitor, and Refine
1. Save the flow and click Test → Manually.
2. Send a sample survey PDF to your GoDaddy email.
3. Verify the flow run history, check S1/S2 values, and confirm the return email.
4. Adjust parsing logic or lookup entries if any responses aren’t matching.

**Custom connectors**
You can write custom connectors to access services that don't have prebuilt connectors. These services must have a REST or SOAP API, which isn't surprising because a connector is just a wrapper around an API.

https://learn.microsoft.com/en-us/training/modules/intro-to-logic-apps/3-how-logic-apps-works

https://learn.microsoft.com/en-us/training/modules/intro-to-logic-apps/4-when-to-use-logic-apps

**Detailed Step-by-Step Implementation for the GoDaddy-to-Logic Apps Workaround**

1. Forward GoDaddy Survey Emails to a Microsoft 365 Mailbox
- Sign in to your GoDaddy Workspace Email Admin Center.
- Select the mailbox that receives survey submissions.
- In Mail Settings, choose Auto-Forwarding (or Rules).
- Create a rule to forward all incoming survey emails to your dedicated Microsoft 365 address (e.g. surveys@yourdomain.onmicrosoft.com).
- Save and test by sending a sample survey from the original sender.

2. Provision a Dedicated Microsoft 365 Mailbox
- In the Azure portal, go to Azure Active Directory → Users → New user.




