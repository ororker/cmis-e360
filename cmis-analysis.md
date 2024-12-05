# CMIS

This document analyses the Echo360 integration and the approach taken by the Echo360 project
that was mothballed in 2020 and also the current batch approach to transferring data from CMIS to Echo360.

It then proposes an approach for integrating the CMIS timetable data with Echo360 in near real-time
including the mapping of CMIS data to the Echo360 Rest API attributes.

## Analysis

### CMIS Batch

Data is currently transferred between CMIS and Echo360 as a one-off task performed at the start of the academic year.
The process starts by providing the "IS - BUSINESS RELATIONSHIPS MGT & ENGAGE" (BRME) team with CSV files 
containing all of the Lecturers, Courses (with subgroups), Campuses, Buildings and Rooms. 
With this data configured in Echo360, the CMIS timetable data is then output to a set of 10 CSV files split by 
the 4 colleges (plus University Services) and room feature (VIDEO or AUDIO).
These files are then passed to the BRME team, who then upload the data into Echo360.

An example of timetable data that is transferred is shown below:

| Start Date | Start Time | End Date | Days Of Week | Exclusion Dates | Duration Minutes | Term Id                              | Name         | Room Id                              | Sections                                      | Instructor Id | Guest Instructor                     | Should Caption | Should Stream Live | Input1 | Input2  | Capture Quality | Stream Quality | External Id |
|------------|------------|----------|--------------|-----------------|------------------|--------------------------------------|--------------|--------------------------------------|-----------------------------------------------|---------------|--------------------------------------|----------------|--------------------|--------|---------|-----------------|----------------|------------|
17/01/2025|09:00||||60|a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0||4d3b4c1e-199f-4f7e-8e4c-89250894a8fd|002545f2-a85e-4fab-b014-5aecb459b4fc=relative_1|541b98ff-6df7-4198-aae0-563e594bf59d||FALSE|FALSE|Display|'%no selection%'|Medium|Medium|214806_22_2024
24/01/2025|09:00||||60|a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0||4d3b4c1e-199f-4f7e-8e4c-89250894a8fd|002545f2-a85e-4fab-b014-5aecb459b4fc=relative_1|541b98ff-6df7-4198-aae0-563e594bf59d||FALSE|FALSE|Display|'%no selection%'|Medium|Medium|214806_23_2024
31/01/2025|09:00||||60|a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0||4d3b4c1e-199f-4f7e-8e4c-89250894a8fd|002545f2-a85e-4fab-b014-5aecb459b4fc=relative_1|541b98ff-6df7-4198-aae0-563e594bf59d||FALSE|FALSE|Display|'%no selection%'|Medium|Medium|214806_24_2024
07/02/2025|09:00||||60|a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0||4d3b4c1e-199f-4f7e-8e4c-89250894a8fd|002545f2-a85e-4fab-b014-5aecb459b4fc=relative_1|541b98ff-6df7-4198-aae0-563e594bf59d||FALSE|FALSE|Display|'%no selection%'|Medium|Medium|214806_25_2024

The configured process extracts the data from the CMIS DB's `TIMETABLE` table and various other child tables.

The method that is executed to generate each CSV file is `TestWorkloadExtractService#testProduceEchoExtract` 
and this code should be referenced when identifying where to source the CMIS data used to generate the payload that needs 
to be sent to Echo360.

### Echo360 Project (2020)

There exists code for an Echo306 project that was mothballed in 2020 and it contains examples of REST API calls that invoke the Echo306 SDK
that still work against the current Echo360 server.
The process to initiate the update from SpaceTT to Echo360 is configured to be triggered by a Spring CRON job.
The process starts by performing a diff of what SpaceTT thinks
Echo360 course data has and what the latest SpaceTT data actually is, 
i.e. the difference between the tables `STT_ECHO_SCHED_ECHO` and `STT_ECHO_SCHED_CMIS`. 

The 2020 implementation fetches from Echo360 all data related to room, building, campus, section and lecturers 
at the start of each batch run
so that it can identify missing reference data prior to sending schedule data to Echo360.
If room or lecturer data cannot be found for a particular schedule then an error is logged and the schedule data is not transferred.
If course/sub-group data cannot be found for a particular schedule then a new section (course + sub-group) is created within Echo360 
and the schedule is then transferred.
Some other validations are performed namely:
1. if the event date is in the past then the schedule data is ignored.
2. if the event duration is greater than 4 hours then the schedule data is ignored.
3. if the event name is blank (and there is currently no code to set it) then it defaults to `CMIS event`.
4. if the event is already booked and the event is of type `INSERT` then schedule data is ignored. 
5. if the event is already booked and the event is of type `UPDATE` then an update is performed only if the external id in Echo360 matches, otherwise the schedule data is ignored. 

### Integration from CMIS to SpaceTT

Whenever an INSERT/UPDATE/DELETE query is run on the CMIS `TIMETABLE` table a trigger is fired that inserts a row into the table `STT_EVENT_ACTION` containing the columns:

| COLUMN_NAME    |
|----------------|
| ACTION_EVENTID |
| SETID          |
| SLOTID         |
| SLOTENTRY      |
| SLOT_ACTION    |
| PROCESSED_FLG  |
| DATE_CREATED   |

The columns (SETID, SLOTID, SLOTENTRY) are a composite primary key on the `TIMETABLE` table.

ASSUMPTION : when the weekid changes then the row, represented by (SETID, SLOTID, SLOTENTRY), 
will be deleted (logically i.e. status = 3?) and a new row with a new SLOTID will be created.

SpaceTT has CRON job configured to run the method `mis.spacett.service.ScheduleService#processTimetableEvents` at
"Every minute, between 05:00 AM and 11:59 PM" (`0 */1 5-23 * * *`).
This initiates a process that retrieves upto 75 unprocessed rows at a time from the `STT_EVENT_ACTION` table.

| CODE | STATE       |
|------|-------------|
| U    | UNPROCESSED |
| I    | IN_PROGRESS | 
| P    | PROCESSED   |
| F    | FAILED      |

The status of these rows is then set to IN_PROGRESS.
The IN_PROGRESS rows are then retrieved and the CMIS `TIMETABLE` data identified by the composite key is then 
transferred to the SpaceTT tables.
If successful the `PROCESSED_FLG` is set to `P` else `F`.

#### Algorithm to transfer from CMIS to SPACETT

The first step is to identify the STT_EVENT_ACTION ids to process, which requires the following view and SQL:

```sql
  select  action_eventid as eventActionId
  from    (
          select  rownum as rn, action_eventid, slot_action
          from    (
                  select  action_eventid, slot_action, date_created
                  from    stt_event_action
                  where   processed_flg = 'U'
                  and     setid= :setId
                  and     slotid not in (select slotid from v_stt_in_progress_booking where setid = :setId)
                  order   by date_created
                  ) unprocessed
          ) batch
  where   (rn = :batchSize and slot_action != 'D')
  or      rn < :batchSize
```

Now update these rows in `STT_EVENT_ACTION` to have a status of `IN_PROGRESS`.

There is now special processing to handle "late changes".
The first step is to retrieve the associated weeks.

```sql
  select distinct
   t.slotid          as slotId, 
   t.weekday         as weekDay,
   t.starttime       as startTime,
   t.finishtime      as finishTime, 
   t.weekid          as weekId,
   w.weeks           as weekString,
   t.status          as statusInt
  from    
          timetable t,        
          weekmapstring  w
  where   t.slotentry=1
  and     t.slotid= :eventId 
  and     t.setid =:setId
  and     t.weekid = w.weekid 
  and     w.setid = :setId
```

This code operates in 2 modes either LATE_CHANGING (happening today) or not.
If in LATE_CHANGING mode then the events are now checked to verify whether 
or not the `TIMETABLE` or `STT_TIMETABLE` table have this event as
one that is occurring today. NB code comment - even if the event is not specified as such in `TIMETABLE` it
may still be if the date was moved from today to another date.

Irrespective of the mode all of the old events in `STT_TIMETABLE` are now deleted 
for a particular setId/slotId combination using the following SQL:
```sql
 delete from stt_timetable 
 where setid= :setId and slotid in 
 (select distinct slotid 
  from stt_event_action 
  where slot_action in ('U','D') 
  and processed_flg = 'I' and slotentry=1)
```

The query `getTimetableSlotsForInsertUpdate` retrieves all the timetable related data for the insert/update actions 
that are currently in progress and that data is now inserted into the STT_* tables.

The `STT_EVENT_ACTION` status is then updated to `PROCESSED`. 


## Design

The design to transfer data from CMIS to SpaceTT should follow a standard Extract, Transform and Load (ETL) pattern.

The process will be initiated on the same schedule as that described in the 
section [Integration from CMIS to SpaceTT](#Integration from CMIS to SpaceTT), i.e. every minute.

In the absence of JMS infrastructure the `STT_EVENT_ACTION` table will also be used to control which timetable records are processed.
This table will have two additional VARCHAR columns added called `ECHO360_PROCESSED` and `ECHO360_ERROR_MESSAGE`.
The existing column `PROCESSED_FLAG` will be renamed to `SPACETT_PROCESSED` to make its purpose more explicit.
The Echo360 records to process will be identified by those rows where the `ECHO360_PROCESSED` value is `UNPROCESSED` 
and the `SPACETT_PROCESSED` value is `PROCESSED`. Allowing the SpaceTT processing to complete first 
will hopefully avoid any contention to update the same row by two different processes.

To cope with error scenarios an additional column will be added `ECHO360_ERROR_MESSAGE`.
When an error occurs while processing a message to Echo360, the `ECHO360_PROCESSED` value will be set to `F` 
and `ECHO360_ERROR_MESSAGE` columns will be populated with an appropriate error message. 
These rows will not be re-attempted until the `ECHO360_PROCESSED` and `ECHO360_ERROR_MESSAGE` values are cleared 
which is described in the section [Recovering from Echo360 Error](#Recovering from Echo360 Error)
Subsequent rows with the same primary key will also not be processed.

### Selection Criteria

The selection criteria used to identify data to transfer to Echo360 will follow the logic 
currently used by the [CMIS Batch](#CMIS Batch).

First get a list of the classes (modules) that we are interested in sending to Echo360. 
Do this by joining `MODULE` to `TIMETABLE` and select where the `slotentry` is `1`,
the `status` is `2` and `sourcesid` is `TEACH`. The associated roomid must also have a `featureid` of `AUDIOREC`
or `VIDEOREC`. 

Then get the extract items with associated moduleid, grpcode, component and grpSize. 

Produce the echo schedules by filtering `TIMETABLE` rows by `slotentry` is `1`,
the `status` is `2` and `sourcesid` is `TEACH`. The associated roomid must also have a `featureid` of `AUDIOREC`
or `VIDEOREC`.

### Loading the data changes to Echo360

#### Inserting

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

#### Updating

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

The response to an update request is structurally the same as for a create request.

#### Deleting

An example of a request that can be sent to delete an event is shown below:
```json
delete /public/api/v2/schedules/{schedule}
```
where {schedule} can be the external id.

### Extracting and transform CMIS data

#### How does SpaceTT deal with updates to `WEEKID` or `WEEKDAY`?

#### How do we deal with changes to data items in the filtering criteria?

i.e. `SLOTENTRY`, `STATUS`, `SOURCESID`, `ROOMID`, `FEATUREID`.

One possibility is to record all rows sent to Echo360 so that we can determine
whether or not an update or delete request needs to be sent.

### Reference data not in Echo 360

Prior to any schedules being transferred to Echo360, all of the reference data associated with:
- rooms
- buildings
- campuses
- lecturers
- sections
- courses
- departments
- organisations

will be retrieved from Echo360 and cached. 
Checks will be made using this cached data to ensure that Echo360 contains the necessary reference data
prior to an insert or update request being made.

#### Start and end times
The start and end, date and times can be read from the `STT_TIMETABLE` columns: `STARTDATE` and `FINISHDATE`.

#### Rooms, Buildings and Campuses
It is proposed that when communicating the room to Echo360, the externalId attribute is used.
This value can be sourced from `TIMETABLE.ROOMID`.
The room definition within Echo360 contains a reference to a Building, which contains 
a reference to a Campus so there is no need to specify the Building or Campus
within a create/update schedule request.

In theory all the CMIS room/building/campus data should be in sync with Echo360.
In integration testing we can evaluate the prevalence of missing data in Echo360
and enhance this solution to populate the data automatically when it is missing if necessary.

#### Lecturers

In the CSV file generated at the start of the year, the lecturer value used is the Echo360 user id.
This value is sourced from the table 'STT_ECHO_USER' after the id of the first lecturer or contact is identified.
So that we don't need to maintain this table it is proposed that in our requests we use the user/contact's email
address which is available from `STT_EXT_PERSON.BUSINESSEMAIL`. 

#### Sub-group, course, department and organisation

| Name         | Source                          |
|--------------|---------------------------------|
| sub-group    | TIMETABLE.MODGRPCODE            |
| course       | TIMETABLE.MODULEID              |
| department   | TIMETABLE.DEPTID                |
| organisation | first 3 chars of dept + 5 '0's? |

NB the retrieval method for `course` is more complex in the Echo360 code. See `<sql-query name="getSttSections">`.

### Transforming the data

### Error handling

#### External Id not unique
If an insert request is made but the external id is already known to Echo360 then we can assume that this request is 
being retried and therefore if it fails on this occasion then we can assume it was successful previously and we can ignore the error.

#### Venue / Time slot is already taken
When an insert or update request receives a `Schedule timing clash with another Schedule` error then record this error in 
`STT_EVENT_ACTION.ECHO360_ERROR_MESSAGE` and do not resend until the issue has been resolved manually.

#### Other errors
When any other errors occur note them in `STT_EVENT_ACTION.ECHO360_ERROR_MESSAGE`
and await manual intervention before retrying.

## User Interface

A user interface will be available to allow administrators to view the number of schedules that have been 
transferred in the last 24 hours and the number (if any) that are pending transfer.

Administrators will also be able to see the schedules that failed to transfer and re-trigger the sending of the schedule change.


---




The records that will be processed 

The columns (SETID, SLOTID, SLOTENTRY) are a composite primary key on the TIMETABLE table and therefore should be used as the EXTERNAL_ID when communicating schedule events to Echo360.

It is proposed that instead of this batch approach we queue changes to the schedule in CMIS for transfer to Echo360 within one minute.
When CMIS is rolled-over we estimate that this could result in 185,000 being queued for transfer which could take a day to fully transfer to Echo360.

An event table already exists within CMIS to facilitate the transfer of schedule changes to SpaceTT.
It is proposed that this approach is extended to support the integration to Echo360. 
JMS would be a natural architectural component to support this type of integration however additional time would be required to setup a JMS server.


## Tasks to integrate CMIS with ECHO360


## Extraction
To identify the data that needs to be extracted from CMIS a combination of examining the current batch process to see what data it extracts with reviewing the data that needs to 
be submitted to the 


The current process extracts all timetable records for the year however the new implementation should process records as they appear in the table `stt_event_action`.
The current process extracts the data using many different SQL calls.

These should be analysed and refactored/implemented in a Java21 compliant project.

An example of a few of the rows from the CSV file that was extracted from CMIS is shown below:

| Start Date | Start Time | End Date | Days Of Week | Exclusion Dates | Duration Minutes | Term Id                              | Name         | Room Id                              | Sections                                      | Instructor Id | Guest Instructor                     | Should Caption | Should Stream Live | Input1 | Input2  | Capture Quality | Stream Quality | External Id |
|------------|------------|----------|--------------|-----------------|------------------|--------------------------------------|--------------|--------------------------------------|-----------------------------------------------|---------------|--------------------------------------|----------------|--------------------|--------|---------|-----------------|----------------|-------------|
| 14/01/2025 | 13:00      |          |              |                 | 60               | a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0 | CC1B Lecture | 0a2140d3-1258-413c-949d-9dd6d753d463 | 008539bf-c2db-470c-841d-7ba85bc5a92c=relative | 1             | 30dd631a-7b24-4d7b-aa21-70cfbf901955 |                | FALSE              | FALSE  | Display | Video           | Medium         | 4301_22_2024|
| 21/01/2025 | 13:00      |          |              |                 | 60               | a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0 | CC1B Lecture | 0a2140d3-1258-413c-949d-9dd6d753d463 | 008539bf-c2db-470c-841d-7ba85bc5a92c=relative | 1             | 30dd631a-7b24-4d7b-aa21-70cfbf901955 |                | FALSE              | FALSE  | Display | Video           | Medium         | 4301_23_2024|
| 25/03/2025 | 13:00      |          |              |                 | 60               | a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0 | CC1B Lecture | 0a2140d3-1258-413c-949d-9dd6d753d463 | 008539bf-c2db-470c-841d-7ba85bc5a92c=relative | 1             | 30dd631a-7b24-4d7b-aa21-70cfbf901955 |                | FALSE              | FALSE  | Display | Video           | Medium         | 4301_32_2024|
| 13/01/2025 | 13:00      |          |              |                 | 60               | a2cbad7b-a9d1-4c3c-ad72-df7f71c062a0 | CC1B Lecture | 0a2140d3-1258-413c-949d-9dd6d753d463 | 008539bf-c2db-470c-841d-7ba85bc5a92c=relative | 1             | 695b080c-6bcf-4c72-b0c3-7671a739ff9f |                | FALSE              | FALSE  | Display | Video           | Medium         | 690452_22_2024|


## Transform
The transformation will not be the same as the current process because that transforms to a CSV file. 
A rough description of the transformations for 5 of the key elements is shown in Table 1.
The new transformation will need to map (presumably) the same data to the Echo360 REST interface.

| CMIS                             | EchoSchedule                      | Echo360      | Notes                                                                                                                                               |
|----------------------------------|-----------------------------------|--------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| stt_echo_term.echo_term_id       | echoTermId                        | terms        |                                                                                                                                                     |
| stt_echo_room.echo_room_id       | echoRoomId                        | rooms        | handle split rooms as per WorkloadExtractService#produceEchoSchedules. From timetable.roomid => timetable.roomgrpcode => stt_echo_room.echo_room_id |
| stt_echo_section.echo_section_id | echoSections+AvailabilitySections | sections     | lookup by <module_id> + <grp_code> + <year>                                                                                                         |
| stt_echo_user.echo_user_id       | echoInstructorEchoId              | instructors  | lookup based on timetable.lecturerid or slotdetails.contact                                                                                         |
| timetable.slot_id                | eventId + weekNumber + year       | external_ids |                                                                                                                                                     |

**Table 1:** Transformation from CMIS to Echo360 CSV file

## Load
Calling the Echo360 REST interface should be relatively straightforward except that we will need to handle 
a number of error scenarios, e.g. where the identifiers for terms/rooms/sections?/instructors/external_ids don't match the data that is currently configured within Echo360.


## Prerequisites

Wildfly instance running Java 21.
Echo360 instance we can use for DEV/TEST.
GitHub repository for the new project.
Jenkins jobs to build the project and deploy to the Test/Live environments.
SMTT/Echo360 users available to verify the integration.

## Tasks

Scenario: Schedule class and integrate with Echo360
Given I am a Central Timetabler 
And in CMIS, class C1 is currently unroomed and requires video recording
And Echo360 is not aware of class C1
And Echo306 knows lecturer L
And Echo306 knows class C1's course C
And Echo306 knows about course C's Organisation Structure
And Echo306 knows about room R
When I schedule the class C1, with lecturer L, in room R that has video equipment
Then the schedule for this new class should be sent to Echo360




2. 

## Notes

The `Echo360` project in Subversion just has some code to invoke the REST API.
This project makes use of 6 tables:

| TABLE_NAME             | NOTES                                                                                                                      |
|------------------------|----------------------------------------------------------------------------------------------------------------------------|
| STT_ECHO_SCHED_CMIS    | Populated by [SPACETT]EchoExtractService#produceEchoScheduleExtract() but nothing invokes                                  |
| STT_ECHO_SCHED_ECHO    | Not referenced in SPACETT. Insert/Updated in UploadService#insert/updateSchedule() when event successfully sent to Echo360 |
| STT_ECHO_SCHED_HISTORY | After comparing the 2 tables above, writes the messages that will be sent to Echo360 to this table                         |
| STT_ECHO_USER          |                                                                                                                            |
| STT_ECHO_COURSE        |                                                                                                                            |
| STT_ECHO_SECTION       |                                                                                                                            |




- create a new Wildfly instance running Java 21.
- migrate the existing code that updates the SpaceTT tables from the CMIS tables.
- extend the code that triggers the updates to SpaceTT so that it will also initiate the transfer to Echo360
- refactor the code currently in CMIS that generates the CSV files
  - this code is initiated from `mis.spacett.service.TestWorkloadExtractService#testProduceEchoExtract`
  - it ultimately generates CSV files as shown here: https://gla.sharepoint.com/:f:/s/Echo360CMISIntegration/EsdLT8nBFN9CpV1dXtsvDX0BxiRxHR72Fg0Vgn_lkxL8tw?e=xb99qT
  - the columns in this file are:
    - Start Date
      Start Time
      End Date
      Days Of Week
      Exclusion Dates
      Duration Minutes
      Term Id
      Name
      Room Id
      Sections
      Instructor Id
      Guest Instructor
      Should Caption
      Should Stream Live
      Input1
      Input2
      Capture Quality
      Stream Quality
      External Id
  - identify from the [Echo360 documentation](https://echo360.org.uk/api-documentation) the fields that need to be supplied to communicate a change
```json
[
  {
    "institutionId": "string",
    "id": "string",
    "name": "string",
    "startDate": "2017-02-01",
    "startTime": "09:00",
    "endTime": "10:00",
    "endDate": "2017-05-31",
    "daysOfWeek": [
      "string"
    ],
    "exclusionDates": [
      "string"
    ],
    "venue": {
      "campusId": "string",
      "campusName": "string",
      "buildingId": "string",
      "buildingName": "string",
      "roomId": "string",
      "roomName": "string"
    },
    "presenter": {
      "userId": "string",
      "userEmail": "string",
      "userFullName": "string"
    },
    "guestPresenter": "string",
    "sections": [
      {
        "courseId": "string",
        "courseIdentifier": "string",
        "termId": "string",
        "termName": "string",
        "sectionId": "string",
        "sectionName": "string",
        "availability": {
          "availability": "string",
          "relativeDelay": 0,
          "concreteTime": "2024-10-29",
          "unavailabilityDelay": 0
        }
      }
    ],
    "shouldCaption": true,
    "shouldStreamLive": true,
    "shouldAutoPublish": true,
    "shouldRecurCapture": true,
    "input1": "string",
    "input2": "string",
    "captureQuality": "string",
    "streamQuality": "string",
    "externalId": "string"
  }
]
```



## Echo360 [CTS-204](https://uofglasgow.atlassian.net/browse/CTS-204)

## CMIS Integration with SpaceTT

Timetable events are maintained within the `TIMETABLE` table within the CMIS DB.
When a DML statement is executed on the `TIMETABLE` table an INSERT/UPDATE/DELETE trigger fires and inserts a row 
into the table `STT_EVENT_ACTION`, using the stored procedure `proc_stt_event_action`.
```sql
create or replace procedure proc_stt_event_action (
  setid_in           in   varchar2,
  slotid_in         in     number,
  slotentry_in      in     number,
  slot_action_in    in     varchar2) is
BEGIN
	insert into stt_event_action(
	          setid,
	          action_eventid	,
              slotid	,
              slotentry,
              slot_action   ,
              processed_flg  ,
              date_created
                         )
                      values
                        (
                         setid_in,
                         stt_event_action_seq.nextval,
                         slotid_in,
                         slotentry_in,
                         slot_action_in,
                         'U',
                         SYSTIMESTAMP);

 END
```

What populates the STT_TIMETABLE table when the STT_EVENT_ACTION row is processed?
SpaceTT has CRON job configured to run the method `mis.spacett.service.ScheduleService#processTimetableEvents` at
"Every minute, between 05:00 AM and 11:59 PM" `0 */1 5-23 * * *`.
This process 
```sql
CREATE OR REPLACE FORCE EDITIONABLE VIEW "CMISMGR"."V_STT_IN_PROGRESS_BOOKING" ("SLOTID", "SETID") AS 
  select  sd.slotid as slotid,
        sd.setid as setid
from    timetable t, 
        slotdetails sd
where   t.setid = sd.setid
and     t.slotid = sd.slotid
and     t.userchange like 'facility%'
and     sd.slotline = 0
and     sd.sourcesid is null;

select  action_eventid as eventActionId
from    (
            select  rownum as rn, action_eventid, slot_action
            from    (
                        select  action_eventid, slot_action, date_created
                        from    stt_event_action
                        where   processed_flg = 'U'
                          and     setid= :setId
                          and     slotid not in (select slotid from v_stt_in_progress_booking where setid = :setId)
                        order   by date_created
                    ) unprocessed
        ) batch
where   (rn = :batchSize and slot_action != 'D')
   or      rn < :batchSize
```
This sql retrieves 75 rows of unprocessed rows from stt_event_action table.
The status of these rows is then set to IN_PROGRESS.
The IN_PROGRESS rows are then retrieved.

Scenarios that need to be addressed.
- re-sync CMIS data to Echo360
  - should this be done for all records in one year, just those in the future, something else?
- propagate updates in CMIS data to Echo360

## How was the synchronization done this year

Use the method `mis.spacett.service.EchoExtractService#produceEchoSchedules` as a reference for the information that needs to be extracted from CMIS and passed to Echo360.



Anna asked me to create some files by running the process;

mis.spacett.service.TestWorkloadExtractService#testProduceEchoExtract
mis.spacett.service.WorkloadExtractService#produceEchoScheduleExtract
mis.spacett.service.WorkloadExtractService#writeEchoScheduleFile

The process begins by running the following SQL:
```json
  select distinct m.moduleid as moduleId
  from  module m,timetable t
    where  m.setid= :setId
    and t.slotentry=1 
    and m.owner like '1%'
    and t.setid=m.setid and t.moduleid=m.moduleid and t.sourcesid='TEACH' and t.status =2
   and exists
      (select * from roomrequests r where r.setid=t.setid and  r.featureid='AUDIOREC')
   AND EXISTS
      (SELECT * FROM roomfeatures f WHERE f.setid=t.setid AND f.roomid= t.roomid AND   f.featureid='AUDIOREC' )
```
The digit in the owner clause needed to be updated to reflect the particuar college data that was
being extracted e.g. 1 - Arts, 2 - MVLS, ..., 9 - University Services.
The clauses featuring the `featureid` also had to be modified for each run switching between `AUDIOREC` and `VIDEOREC`.

For each module that is extracted the following SQL is run:
```sql
 	select	g.moduleid as moduleId, 	
 			g.grpcode as grpCode,
			c.component_type_code as component,
			g.csize as grpSize
	from 	modulegroups g
	full outer join stt_component_type c on substr(g.grpcode,1,2) = c.module_subgroup_code
	where 	g.setid= :setId
	and     g.moduleid = :moduleId
```





mis.spacett.service.TestEchoExtractService#testProduceEchoExtract
- gets the string identifying the NEXT_DATASET i.e. 2024-25
- produceClassExtractFileForEcho
   - getCourseForExtractForEchoExtract
      - calls SQL getCourseForExtractForEchoExtract2 with the NEXT_DATASET param and returns 662 rows
```sql
  select distinct m.moduleid as moduleId
  from   module m
  where  m.setid= :setId
  and   (owner  like '401%') 
``` 
        - returns a list of ClassExtract objects but only the moduleId attribute is populated. termCode and moduleName will be null. ouCode, items and status are other fields in this class.
      - getModuleSubgroupByModuleId
```sql
  select  grpcode as moduleGroupId
  from    modulegroups
  where   setid = :setId
  and moduleid  = :moduleId
```
    - if a module sub group exists
        - 
```sql
 	select	g.moduleid as moduleId, 	
 			g.grpcode as grpCode,
			c.component_type_code as component,
			g.csize as grpSize
	from 	modulegroups g
	full outer join stt_component_type c on substr(g.grpcode,1,2) = c.module_subgroup_code
	where 	g.setid= :setId
	and     g.moduleid = :moduleId
```
getTimetableTeachingEventsForEchoExtract
```sql
   select distinct
   t.slotid          as slotId,
   t.slotentry       as slotEntry,
   t.slottotal       as slotTotal,
   t.setid           as setId,
   t.status          as statusInt,
   t.weekday         as weekDay,
   t.starttime       as startTime,
   t.finishtime      as finishTime,
   t.duration        as duration,
   t.weekid          as weekId,
   w.weeks           as weekString,
   t.lecturerid      as lecturerId,
   t.moduleid        as moduleId,
   t.modgrpcode      as modGrpCode
  from    
          timetable t,
          weekmapstring  w
  where   t.setid =:setId
  and     t.sourcesid='TEACH' 
  and     t.status=2 
  and     t.slotentry =1
  and     t.roomid is  not null
  and     t.moduleid=:moduleId
  and     t.modgrpcode=:modGroupCode
  and     t.weekid = w.weekid (+)
  and     (w.setid = :setId  or w.setid is null)
  and exists
  (select * from roomrequests r
  where r.setid=t.setid
  and r.slotid=t.slotid
  and (r.featureid='AUDIOREC' or r.featureid='VIDEOREC'))
  and exists
  (select *
  from  roomfeatures f
  where f.roomid=t.roomid
  and (f.featureid='AUDIOREC' or f.featureid='VIDEOREC') 
  and f.setid=t.setid)
  order by t.slotid, t.slotentry 
```
The above sql returns SLOTS.
For each slot:
1. get the lecturers
2. get the event details
3. resolve the week-string to weeks
4. get the roomid for the slot
5. get the roomgrpcode 
6. get the capacity from the rooms table 
7. populate the MeetingExtractItem and EchoScheduleItem objects
```
  select distinct descrip  as details 
  from  v_stt_slotdetails
  where setid =:setId
  and slotid = :slotId
```


## Access

### Web

Test server: https://fdtest.spa.gla.ac.uk/spacett/web/home.jsf
Live server: via Staff Portal

### DB

See SqlDeveloper

### CMIS Client

RDP link
- then login in with email/password
- within CMIS app select live/test environment then login again with credentials from BitWarden 

### CMIS FUNCTIONALITY

#### Physical
- campuses
- buildings
- rooms
- room pools
- equipment/features
- room features/equipment
- moveable equipment
- travel

#### Academic
- departments
- courses
- instances
- plans
- plan parts
- plan structures
- plan part structure
- class groups
- sub group equivalence
- change sub group code

#### Lecturers


#### Students


#### Miscellaneous


Notes

users
- create classes: school timetablers
- smtt

teaching/non-teaching
non-teaching: lib book
unibus - graduation
reqtype: loc/ctt



datasets: 2024/25


tables timetable/slotid
slotdetails
audittrail
stt_booking_set - links stt booked events to cmis booking events



event_action 
copies values to stt_timetable


stt_campus_class



inputs
mycampus 
every morning read data from mycampus/quemus/hr external tables

outputs
to mycampus 



Outbound data from MyCampus

stt_echo_course
- permission to be recorded
 

SMTT

J:\DevandIntegration\BusinessSystems\Timetabling\Support\ECHO_360
J Drive is \\campus.gla.ac.uk

[Echo360 API](https://echo360.org.uk/api-documentation#!/schedules_v2/List)
- create an access token and then copy it to the `access token` textbox in the top right of the page.



an entry is made into the table `STT_EVENT_ACTION`.
When CMIS is rolled over this can generate 185,000 entries in the `STT_EVENT_ACTION` table in just one day.
The Echo360 project has an API rate limiter limiting calls to ~120/min, meaning that it could take one day to clear the backlog.

The java process is run 10 times to cover the different combinations of the 5 owners of events
(the 4 colleges plus University Services) and 2 recording options (Audio/Video).
The configured process extracts the data from the CMIS DB's TIMETABLE table and various other child tables.

It is proposed that instead of this batch approach we trigger the transfer of events to Echo360 when an entry is made into the table `STT_EVENT_ACTION`.
