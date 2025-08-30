# SurveyAnalysisTest1
1. create a new godaddy email account
2. change setting on godaddy admin to forward the email of survey to new go daddy account
3. test azure function locally
4. use Singapore datacenter


Resources - 
Azure storage account - blob
doc intelligence service
resource group
Create a custom extraction project document intelligence studio

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



