# FeedAI - Capstone23
This project involves the automation of teacher feedback analysis and report generation using Google Sheets, Google's Generative AI, and Google Docs. The primary goal is to streamline the process of collecting feedback from students, analyzing the responses, and generating detailed teacher reports.
## S1: Initialization and Dependencies
In this section, we initiate the code by setting the working directory to a specified path and installing the required dependencies. The necessary libraries, including gspread for Google Sheets integration and google.generativeai for natural language generation, are imported. Additionally, standard libraries such as os, io, shutil, and time are imported to facilitate file operations, input/output handling, and time-related functionalities.

```
%cd /content/drive/MyDrive/FeedAI
!pip install gspread==5.4.0
import gspread

import google.generativeai as palm
import os

from googleapiclient.discovery import build
from google.oauth2.service_account import Credentials

import time, os, io, shutil
from googleapiclient.http import MediaIoBaseDownload
```

## S2: Google Sheets Authentication

Here, we authenticate with the Google Sheets API using a service account. The service account credentials are loaded from the service_account.json file, and we access the specific worksheet ("feedbackresponses") within the Google Sheets document.

```
sa = gspread.service_account(filename='sheetskey.json')
sh = sa.open("feedbackresponses")
wks = sh.worksheet("responses")
```

## S3: Data Extraction

This section involves retrieving all records from the Google Sheet through the get_all_records() method provided by the gspread library. We create a list of questions, excluding the header row, which will be used for further data processing.

```
cells = wks.get_all_records()
questions = [i for i in cells[0]]
del questions[0:1]
```

## S4: Data Processing

The process_data function is defined to organize the extracted data into separate dictionaries for each subject. It iterates through the rows of the Google Sheet and appends responses to the corresponding questions for a specified subject. The processed data is stored in dictionaries named subject_eng, subject_cs, subject_mat, subject_phy, and subject_che.

```
r_eng,r_cs,r_mat,r_phy,r_che = {},{},{},{},{}

for i in questions:
  r_eng[i],r_cs[i],r_mat[i],r_phy[i],r_che[i]=[],[],[],[],[]
for i in cells:
  if i['Teacher Name']=='English':
    for j in questions:
      r_eng[j].append(i[j])
  elif 'Computer Science' in i['Teacher Name']:
    for j in questions:
      r_cs[j].append(i[j])
  elif i['Teacher Name']=='Maths':
    for j in questions:
      r_mat[j].append(i[j])
  elif i['Teacher Name']=='Physics':
    for j in questions:
      r_phy[j].append(i[j])
  elif i['Teacher Name']=='Chemistry':
    for j in questions:
      r_che[j].append(i[j])
```

## S5: Teacher Report Generation

This section utilizes Google's Generative AI, palm, to dynamically generate teacher reports based on the collected responses. A loop iterates through each subject's processed data, constructing a prompt for the generative model. The generated responses are then evaluated and stored in the reports list.

```
os.environ['API_KEY']='AIzaSyDsToJOiYVcdL-3AasBAjM6HyLu-Bd4WMs'
palm.configure(api_key=os.environ['API_KEY'])

genout=[]
for i in [r_eng,r_cs,r_mat,r_phy,r_che]:
  ask=f'''
Write a report of the teacher based on the following responses. Take an average of the responses and provide the output according to the format.
questions : list of responses in a dictionary key : value, {i}
This dictionary should be looked out any try to have a unique response everytime.

strictly follow this list format:
[<string subject>,[<int thoroughness in %>, <int engaging in %>, <int caring in %>],"<string brief description of the teacher according to the given responses alteast 100 words>"]'''
  response = palm.generate_text(prompt=ask)
  genout.append(eval(f"{(response.result)}"))

r_eng,r_cs,r_mat,r_phy,r_che = genout[0], genout[1], genout[2], genout[3], genout[4]
```

## S6: Google Docs and Drive Integration

Here, we configure credentials for both Google Docs and Google Drive. We set the timezone, initialize the Google Docs and Drive services using the obtained credentials, and specify the template ID for the document to be copied. Additionally, the Google Drive credentials are used to initialize the service for exporting documents to PDF.

```
os.environ['TZ'] = 'Asia/Kolkata'
time.tzset()
timestr = time.strftime("%Y%m%d-%H%M%S")
SCOPES = ['https://www.googleapis.com/auth/documents','https://www.googleapis.com/auth/drive','https://www.googleapis.com/auth/drive.file']

credentials = Credentials.from_service_account_file('docskey.json', scopes=SCOPES)
dcreds = Credentials.from_service_account_file('drivekey.json', scopes=SCOPES)
service = build('docs', 'v1', credentials=credentials)
news = build('docs', 'v1', credentials=dcreds)
dservice = build('drive', 'v3', credentials=dcreds)
template_id = '1Z8k6QK5lV0MDyTK2toJHBtgfZPYigD2QAfz98zMWbEk'
```

## S7: Generate PDF Reports

In this final section, we loop through the generated teacher reports for each subject. For each report, a new document is created by copying the specified template. Placeholders in the document are replaced with the corresponding values from the generated report. The modified document is then exported to PDF format, and the progress is displayed during the download.

```
for i in [r_eng,r_cs,r_mat,r_phy,r_che]:
    document = dservice.files().copy(
        fileId=template_id,
        body={"name": f"The{i[0]}_TeacherReport_{timestr}"}).execute()
    print(document.get('id'))
    timestr = time.strftime("%Y%m%d-%H%M%S")
    placeholders = {
    '{teacher}': f'{i[0]}',
    '{TH}': f'{i[1][0]}',
    '{EN}': f'{i[1][1]}',
    '{CA}': f'{i[1][2]}',
    '{summary}': f'{i[2]}',
    }
    body = document.get('body')
    for placeholder, replacement in placeholders.items():
        requests = [
        {
                'replaceAllText': {
                    'containsText': {
                        'text' : placeholder
                    },
                    'replaceText': replacement
            }
        }
        ]
        news.documents().batchUpdate(documentId=document.get('id'), body={'requests': requests}).execute()
        request = dservice.files().export_media(fileId=document.get('id'), mimeType='application/pdf')
        fh = io.FileIO(f"The{i[0]}_TeacherReport_{timestr}.pdf", mode='wb')
        downloader = MediaIoBaseDownload(fh, request)
        done = False
        while done is False:
            status, done = downloader.next_chunk()
        print('Download %d%%.' % int(status.progress() * 100))
```
