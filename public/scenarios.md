# From conflict to conflict with one event moving to a different room and clashing with other room's event
Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 2    | 1100  | 1200 |

And the e360 schedule is as follows: // due to previous conflict events were deleted

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
| 1  | 1    | 0900  | 1000 |

# From conflict to no conflict with two events moving, one to a different room

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

# From conflict to conflict with same events within a different room, with two events moving

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
| 1  | 2    | 0900  | 1000 | U      |
| 2  | 2    | 1100  | 1200 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 2    | 0900  | 1000 |
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

# From conflict to no-conflict within the same room, with one event moving

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
| 1  | 1    | 0900  | 1000 | U      |

Then a POST request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

# From conflict in one room, both move to conflict in different room

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

# From no conflict to conflict

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

Then a DEL request will be sent to e360 for events 1 & 2
And the e360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

# Duplicate Insert from CMIS

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
| 1  | 1    | 0900  | 1000 | I      |

Then no messages should be sent to e360
-- this should never happen because the DB constraints should prevent a duplicate slotid entry being entered

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

# Insert from CMIS

Given the cmis schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

And the e360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

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

