# CMIS Design

## Definitions

Lesson
: An event where a specified lecturer presents part of a course in a particular room at certain date and time.

E360
: The GU Echo360 server that captures and makes available video and audio recordings.

SpaceTT2
: A new component that may be implemented by GU IT department to automate the integration between CMIS and Echo360.

## Different data model structures

The data models used by CMIS and E360 differ in one very distinct aspect,
namely how they represent classes.

In E360 the main entity is called SCHEDULE and it represents
one lesson, taking place in a particular room and at a particular date and time.

In CMIS the main entity is called TIMETABLE. 
This represents a set of classes taking place in a particular room and 
at a particular time and day of the week but across 
1 or more weeks. 

NB even though multiple rows within TIMETABLE can have the same SETID and SLOTID, (but with different SLOTENTRY values)
each SETID and SLOTID combination always have the same WEEKID and WEEKDAY values.

When publishing CMIS schedule information to E360 a REST API must be used.
Due to the data model structures mentioned above when a change is made to one 
row in CMIS, many REST API calls may need to be made to the E360 REST API in order to keep
the schedules in sync.

## Criteria for sending schedule information to E360

When a TIMETABLE entry is inserted or updated, it is only of interest to E360 if it meets the following criteria:
- the TIMETABLE column SOURCESID must have value 'TEACH'
- the TIMETABLE column STATUS must have value '2'
- the TIMETABLE column SLOTENTRY must have value '1'
- the TIMETABLE column ROOMID must be non-null
- a row must exist in MODULE table with the same (SETID, MODULEID)
- a row must exist in ROOMREQUESTS with matching (SETID) and FEATUREID must have value 'AUDIOREC' or 'VIDEOREC'.
- a row must exist in ROOMFEATURES with matching (SETID, ROOMID) and FEATUREID must have value 'AUDIOREC' or 'VIDEOREC'.
- a row must exist in MODULEGROUPS with matching (SETID, MODULEID) and the first 2 chars of MODULEGROUPS.GRPCODE must match a row in STT_COMPONENT_TYPE.MODULE_SUBGROUP_CODE
- the MODULEGROUPS.GRPCODE value from above must have a length of 4 and characters 3 and 4 must be digits.

The criteria above is an interpretation of the logic used the SpaceTT code `TestWorkloadExtractService#testProduceEchoExtract`.

## Scenarios

The CMIS analysis identified 9 scenarios that need to be considered when transferring data from CMIS to E360.
These are laid out in the following sub-sections.
In these sections we assume that previous messages were successfully sent to E360.
It will be the SpaceTT2 Dispatcher's responsibility to ensure that messages are sent in the correct order.

### New schedule - meets criteria

The most straight forward scenario is when a new schedule is submitted that meets the criteria
such that it needs to be sent to E360. This requires that a POST request be sent to E360 for each 
week associated with the weekId.

### New schedule - doesn't meet criteria

When a new schedule is submitted that does not meet the criteria for it to be sent to E360 then this schedule can 
simply be ignored.

### Update schedule - previously did not meet criteria - now meets criteria

When a schedule is updated that previously did not meet the criteria for it to be sent to E360
then treat it as a [New schedule](#New schedule - meets criteria)

### Update schedule - previously did not meet criteria - still does not meet criteria

When a schedule is updated that previously did not meet the criteria for it to be sent to E360,
and it still does not meet the criteria then the update can be ignored.

### Update schedule - previously met criteria - still meets criteria - week id not updated

When a schedule previously met the criteria and is updated such that it still meets the criteria
and the WEEKID value has not been updated then a PUT request must be sent to E360 for each week associated
with the weekId.

### Update schedule - previously met criteria - still meets criteria - week id updated

When a schedule previously met the criteria for sending to E360 and 
is then updated, where the update includes a change to the weekid, and still satisfies the
criteria for sending to E360 then a series is insert, update and delete messages 
will be sent to E360.

Where the new WEEKID references weeks that weren't associated with the original schedule then a POST request will be sent for each of these weeks.
Where the new WEEKID no longer references weeks that were associated with the original schedule then a DELETE request will be sent for each of these weeks.
Where the new WEEKID references weeks that were also associated with the original schedule then an UPDATE request will be sent for each of these weeks.

### Update schedule - previously met criteria - no longer meets criteria

When a schedule previously met the criteria and is updated such that it no longer meets the criteria
then for each of the weeks associated with the weekid of the schedule before it was updated
a DELETE request must be sent to E360. 

### Cancel schedule - previously met criteria

When a schedule previously met the criteria and is cancelled a DELETE request myst be sent to E360.

### Cancel schedule - previously did not meet criteria 

When a schedule is cancelled that previously did not meet the criteria for it to be sent to E360
then the cancellation can be ignored.

## Component Interactions
![Sequence Diagram](img/seq.png)

## Triggers
### DB Insert Trigger
A new DB Insert trigger (after each row) will be created on the TIMETABLE table.
This trigger will verify the criteria in
[Criteria for sending schedule information to E360](#Criteria for sending schedule information to E360)
has been met and if so then a new row will be inserted into
[DB Table STT_ECHO_QUEUE](#DB Table STT_ECHO_QUEUE).

### DB Update Trigger
A new DB Update trigger (before each row) will be created on the TIMETABLE table.
This trigger will identify which of the scenarios
specified above this update matches and depending on that scenario may 
insert the appropriate values into STT_ECHO_QUEUE.

### DB Delete Trigger
A new DB Delete trigger (before each row) will be created on the TIMETABLE table.
This trigger will insert a new row in STT_ECHO_QUEUE with an action DELETE
if the row that is being deleted or cancelled (status = 3) 
meets the [Previous met criteria](#Previous met criteria) test.

### Previoiusly met criteria

The Update and Delete triggers above need to determine whether
the schedule's previous version met the criteria whereby the data needed to be sent 
to E360.

The test that will be used to determine if the previous version of this schedule
did meet the criteria requiring it to be sent to E360 is:
- there must exist a row in STT_ECHO_QUEUE matching the schedule's (setid, slotid) that was successfully sent
- the most recent of these rows must have an action of INSERT or UPDATE (not DELETE). 


## DB Table STT_ECHO_QUEUE
To support the integration with E360 a new table will be created.

| SOURCE         | NAME          | TYPE          | DESC                                              |
|----------------|---------------|---------------|---------------------------------------------------|
| TIMETABLE      | ID            | NUMBER        | Sequence SEQ_STT_ECHO_QUEUE                       |
|                | CREATED       | DATE          | Date/timestamp when row was inserted              |
|                | PROCESSED     | DATE          | Date/timestamp when row was processed             |
|                | STATUS        | VARCHAR(1)    | [UNPROCESSED\|IN_PROGRESS \| PROCESSED \| FAILED] |
|                | ERROR         | VARCHAR2(100) | A description of the error                        |
| TIMETABLE      | ACTION        | VARCHAR(1)    | The type of request to be sent to E360: [I\|U\|D] |
| TIMETABLE      | SETID         | VARCHAR2(10)  |                                                   |
| TIMETABLE      | SLOTID        | NUMBER        |                                                   |
| TIMETABLE      | WEEKID_OLD    | NUMBER        |                                                   |
| TIMETABLE      | WEEKID_NEW    | NUMBER        |                                                   |
| TIMETABLE      | WEEKDAY       | NUMBER        |                                                   |
| TIMETABLE      | STARTTIME     | VARCHAR2(5)   |                                                   |
| TIMETABLE      | FINISHTIME    | VARCHAR2(5)   |                                                   |
| TIMETABLE      | MODULEID      | VARCHAR2(20)  |                                                   |
| TIMETABLE      | MODGRPCODE    | VARCHAR2(10)  |                                                   |
| TIMETABLE      | ROOMID        | VARCHAR2(16)  |                                                   |
| TIMETABLE      | ROOMGRPCODE   | VARCHAR2(12)  |                                                   |
| TIMETABLE      | LECTURERID    | VARCHAR2(10)  |                                                   |
| STT_EXT_PERSON | BUSINESSEMAIL | VARCHAR2(80)  |                                                   |
| ROOMREQUESTS   | FEATUREID     | VARCHAR2(10)  |                                                   |

NB We don't need the slot entry because it is always 1.

## Spring CRON job

A new application will be created called SpaceTT2.
This application will run on Java21 which is the latest LTS Java version.
This application will be configured to run Spring CRON jobs.
The library that will be used to provide the CRON functionality will be the [Quartz Scheduler](https://docs.spring.io/spring-boot/reference/io/quartz.html).
This library can be easily configured to ensure that CRON jobs only run on one server at a time
should WildFly ever be configured to use multiple servers.

The CRON job will initiate a call to the E360Servce component that will
process messages waiting to be processed on the STT_E360_QUEUE.

## E360Service

The E360Service will read all unprocessed messages from the STT_E360_QUEUE table.
For each row a request will be created and posted to the E360 REST service.

All of the data that needs to be sent to E360 in the Schedule message is contained in the STT_E360_QUEUE table.

Examples of each of the request types are shown below.

### E360 Insert Request

An example of an Echo360 request to notify it of a new timetable entry is shown below:

```json
{
  "startDate": "2024-11-15",
  "startTime": "15:00",
  "endTime": "16:00",
  "sections": [
    {
      "sectionExternalId": "ENGLANG1003_LC01"
    }
  ],
  "name": "Test New Schedule Name",
  "externalId": "617747-1-2023-24",
  "venue": {
    "roomExternalId": "ROOM1"
  },
  "presenter": {
    "userEmail": "David.J.Forrest@glasgow.ac.uk"
  },
  "input1": "Display",
  "input2": "Video",
  "captureQuality": "High"
}
```

TODO where does captureQuality come from?

An example of a response to a successful create event looks as shown below:
```json
{
  "institutionId": "53a8f99c-fb41-4139-9c3c-c471c49f8cf7",
  "id": "43884be2-4906-4bd6-b626-aa0e2f4e2013",
  "name": "Test New Schedule Name",
  "startDate": "2024-11-15",
  "startTime": "15:00",
  "endTime": "16:00",
  "venue": {
    "campusId": "5287b36c-3675-4589-9917-d7173077fbef",
    "campusName": "GILMOREHILL CAMPUS",
    "buildingId": "8479ac03-735f-4982-bdb8-3617978bfd00",
    "buildingName": "Test",
    "roomId": "d73fa127-9cd5-44f9-9cfa-d2445dcafd23",
    "roomName": "Main Meeting Room"
  },
  "presenter": {
    "userId": "14590435-35ff-4650-adf2-796c0e612ad0",
    "userEmail": "David.J.Forrest@glasgow.ac.uk",
    "userFullName": "David Forrest"
  },
  "sections": [
    {
      "courseId": "66567d3e-b575-43cc-ba1a-4f4510bbbb72",
      "courseIdentifier": "ENGLANG1003",
      "termId": "ac00a9be-15f9-4f84-a85e-f9edf50ea38d",
      "termName": "2020/21",
      "sectionId": "8b2437e9-3639-4f9f-9683-c7c869bd187e",
      "sectionName": "LC01"
    }
  ],
  "shouldCaption": false,
  "shouldStreamLive": false,
  "shouldAutoPublish": true,
  "shouldRecurCapture": false,
  "input1": "Display",
  "input2": "Video",
  "captureQuality": "High",
  "externalId": "617747-1-2023-24"
}
```

The full schema for the operation `Add Schedule` is available from the [API documentation](https://echo360.org.uk/api-documentation#!/schedules_v2/Create).

### E360 Update Request

An example of updating the time is shown below.

```json
{
   "startDate": "2024-11-15",
   "startTime": "16:00",
   "endTime": "17:00",
   "sections": [
      {
         "sectionExternalId": "ENGLANG1003_LC01"
      }
   ],
   "name": "Test New Schedule Name",
   "externalId": "617747-1-2023-24",
   "venue": {
     "roomExternalId": "ROOM1"
   },
   "presenter": {
      "userEmail": "David.J.Forrest@glasgow.ac.uk"
   },
   "input1": "Display",
   "input2": "Video",
   "captureQuality": "High"
}
```

The response to an update request is structurally the same as for an insert request.

### E360 Delete Request

An example of a request that can be sent to delete an event is shown below:
```json
delete /public/api/v2/schedules/{schedule}
```
where {schedule} can be an external id or an E360 id. We shall use the external id.

## Error Handling

When an error is received while processing a row within STT_ECHO_QUEUE
then an attempt will be made to recover, e.g. to fix missing reference data, 
however if that is not possible then the STT_ECHO_QUEUE.STATUS value for that row will be set to `FAILED`
and a description of the error added to the STT_ECHO_QUEUE.ERROR column.

A UI will be created to allow administrators to view the messages that have failed 
and allow them to retry 1 or more messages.
When the user submits a retry request via the UI then a new row(s) will be inserted into
STT_ECHO_QUEUE and these will be processed when a subsequent CRON job runs.

### Missing reference data

The reference data contained within E360 will be largely complete however we cannot rely upon all
the rooms, lecturers and sections (course + sub-group) and their parent entities being known to E360.

![Class Diagram](img/class.png)

When a request to insert or update the schedule data in E360 fails because an entity
the schedule depends on is missing then SpaceTT2 will identify the type of 
entity that is missing from the error response, insert the missing entity and any missing parent entities,
and then attempt to insert or update the schedule data again.

The approach may result in repeated attempts to insert or update the same schedule, for example,
when the room, lecturer and course information are all missing from E360.

#### Identifying missing reference data type

When E360 is unable to identify a particular entity specified in a schedule request then it will 
return an error code of `400` and a JSON response document containing `RoomNotFound`, `UserNotFound` or 
`ScheduleNotFound` respectively. An example response document for the room not found scenario 
would be:
```json
{
  "error": "RoomNotFound",
  "message": "Room not found"
}
```

The parents of the Room and Section entities are shown in the diagram above.

NB Users/Lecturers cannot be created via the REST API and therefore will require manual intervention.

### External Id not unique
If an insert request is made but the external id is already known to Echo360 then we can assume that this request is
being retried and therefore if it fails on this occasion then we can assume it was successful previously and we can ignore the error.

### Venue / Time slot is already taken
When an insert or update request receives a `Schedule timing clash with another Schedule` error then record this error in
`STT_ECHO_QUEUE.ERROR` and do not resend until the issue has been resolved manually.

### Other errors
When any other errors occur note them in `STT_ECHO_QUEUE.ERROR`
and await manual intervention before retrying.

## User Interface

A user interface will be available to allow administrators to view the number of schedules that have been
transferred in the last 24 hours and the number (if any) that are pending transfer.

Administrators will also be able to see the schedules that failed to transfer 
and re-trigger the sending of the schedule change.

## Tasks to integrate CMIS with ECHO360

- Setup Wildfly server running Java 21 in the TEST and LIVE environments
- Create a template project running Java 21
- Create the table STT_ECHO_QUEUE.
- Create Stored Procedures to write to STT_ECHO_QUEUE.
- Create the new DB triggers on the TIMETABLE table.
- Add new CRON job to process files to E360
- Add E360Service method to read messages from STT_ECHO_QUEUE
- Add E360Service method to send messages to E360 REST API
- Add error handling code for when the following entities are missing:
  - Room, Building
  - Section, Course, Department, Organisation
- Add UI page to allow administrators to view messages that failed with their error messages.
- Add UI page to allow administrators to retry one or more messages to be retried
- Add UI page to allow administrators to pause the sending of messages to E360
