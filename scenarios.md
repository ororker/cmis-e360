# Insert from CMIS

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 1000  | 1100 | I      |

Then an POST request will be sent to e360
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

# Update from CMIS

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |

Then an PUT request will be sent to e360
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |

# Delete from CMIS

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | D      |

Then an DEL request will be sent to e360
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

# From no-conflict to conflict after event inserted to conflict with existing event

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 0900  | 1000 | I      |

Then the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

[//]: # (A PUT request will initially be sent receiving a response "Venue / Time slot is already taken")

[//]: # (The Room/Day will then be refreshed resulting in DEL requests being sent to e360 for events 1 & 2)

[//]: # (and because of the conflict at 1000 the Room/Day will be marked as In Conflict)

# From no-conflict to conflict after event updated to conflict with existing event

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |

Then the e360 schedule will be as follows:
| id | room | start | end |
|----|------|-------|-----|

[//]: # (A PUT request will initially be sent receiving a response "Venue / Time slot is already taken")

[//]: # (The Room/Day will then be refreshed resulting in DEL requests being sent to e360 for events 1 & 2)

[//]: # (and because of the conflict at 1000 the Room/Day will be marked as In Conflict)

# From conflict to no-conflict with one event deleted

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | D      |

Then the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

# From conflict to no-conflict with one event updated

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 2    | 0900  | 1000 |
| 2  | 2    | 1100  | 1200 |

# From conflict to no-conflict with one event updated to a different room

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 2    | 1000  | 1100 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 2    | 1000  | 1100 |

# From conflict to no-conflict with two events moving, one to a different room

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | U      |
| 2  | 2    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 2    | 1100  | 1200 |

# From conflict to no-conflict within the same room, with two events moving

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | U      |
| 2  | 1    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1100  | 1200 |

# From conflict to conflict with one event inserted but conflict still remains

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

# From conflict to conflict with one event updated but conflict still remains

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1100  | 1200 |

# From conflict to conflict with one event deleted but conflict still remains

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1000  | 1100 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | D      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|


# From conflict to conflict with one event updated to clash with a different event

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1100  | 1200 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

| id | room | start | end |
|----|------|-------|-----|
| 3  | 1    | 1100  | 1200 |

When an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|
| 1  | 1    | 1000  | 1100 |

# From conflict to conflict both events moving to a different room

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

And the e360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

When the following events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 2    | 1000  | 1100 | U      |
| 2  | 2    | 1000  | 1100 | U      |

Then no requests will be sent to E360
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

[//]: # (A PUT request will be made for e1 moving it to room 2)

[//]: # (A refresh will be performed on room 1 resulting in a POST for e2)

[//]: # (A PUT request will be made for e2 receiving a response "Venue / Time slot is already taken")

[//]: # (Room 1 will be refreshed resulting in a DEL for e2)

[//]: # (Room 2 will be refreshed resulting in a DEL for e1 due to conflicting with e2)

[//]: # (Room 1 will no longer be in conflict)

[//]: # (Room 2 will now be in conflict)

# From conflict to conflict with one event moving to a different room and clashing with other room's event

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 2    | 1100  | 1200 |

And the e360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 2    | 1100  | 1200 |

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 2    | 1100  | 1200 | U      |

Then a POST/DEL requests will be sent to e360 for events 1/3
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |

# From conflict to conflict with both events moving in same room and clashing with each other

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |
| 2  | 1    | 1000  | 1100 | U      |

Then no messages should be sent to e360
[//]: # The Room/Day will be refreshed when processing Insert and see new event at 1000
[//]: # When processing the Update its timestamp will be before the last Refresh timestamp and be ignored.
And e360 schedule remains as follows:

| id | room | start | end |
|----|------|-------|-----|

# From conflict to conflict a new event is added not conflicting

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |

Then no messages should be sent to e360
And e360 schedule remains as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 1    | 1000  | 1100 |

# From conflict to conflict a new event is added initially not conflicting and then moved to conflict

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

When events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |
| 3  | 1    | 0900  | 1000 | U      |

[//]: # The Room/Day will be refreshed when processing Insert and see triple clash at 0900.
[//]: # When processing the Update its timestamp will be before the last Refresh timestamp and be ignored.
And e360 schedule remains as follows:
| id | room | start | end |
|----|------|-------|------|
