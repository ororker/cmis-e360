# CMIS to Echo360 Integration Scenarios

## Insert from CMIS

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 1000  | 1100 | I      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

## Update from CMIS

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |

## Delete from CMIS

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | D      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From no-conflict to conflict after event inserted to conflict with existing event

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 0900  | 1000 | I      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From no-conflict to conflict after event inserted to conflict with two existing event

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 0930  | 1030 | I      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From no-conflict to conflict after event updated to conflict with existing event

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1000  | 1100 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to no-conflict with one event deleted

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | D      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 2  | 1    | 1000  | 1100 |

## From conflict to no-conflict with one event updated

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 2    | 0900  | 1000 |
| 2  | 2    | 1000  | 1100 |

## From conflict to no-conflict with one event updated to a different room

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 2    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 2    | 1000  | 1100 |

## From conflict to no-conflict with two events moving, one to a different room

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | U      |
| 2  | 2    | 1100  | 1200 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 2    | 1100  | 1200 |

## From conflict to no-conflict with two events moving within the same room

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0900  | 1000 | U      |
| 2  | 1    | 1100  | 1200 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 1100  | 1200 |

## From conflict to no-conflict with one event overlapping two events

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0930  | 1030 |
| 3  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 0930  | 1030 | D      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 3  | 1    | 1000  | 1100 |

## From conflict to conflict with one event inserted but conflict still remains

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to conflict with one event updated but conflict still remains

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1100  | 1200 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1100  | 1200 |

## From conflict to conflict with one event deleted but conflict still remains

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | D      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to conflict with one event updated to clash with a different event

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 1    | 1100  | 1200 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 1    | 1100  | 1200 |

**When** an event is received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 1    | 1100  | 1200 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |

## From conflict to conflict with both events moving to a different room

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** the following events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 2    | 1000  | 1100 | U      |
| 2  | 2    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to conflict with one event moving to a different room and clashing with other room's event

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |
| 2  | 1    | 1000  | 1100 |
| 3  | 2    | 1100  | 1200 |

**And** the E360 schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 2    | 1100  | 1200 |

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 2  | 2    | 1100  | 1200 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 1000  | 1100 |

## From conflict to conflict with both events moving in same room and clashing with each other

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 1  | 1    | 1000  | 1100 | U      |
| 2  | 1    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to conflict with one event inserted at non conflicting time

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 1    | 1000  | 1100 |

## From conflict to conflict with one event inserted initially not conflicting and then moved to conflict

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 1000  | 1100 | I      |
| 3  | 1    | 0900  | 1000 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end |
|----|------|-------|-----|

## From conflict to conflict with one event inserted initially conflicting and then moved to non-conflicting

**Given** the CMIS schedule is as follows:

| id | room | start | end  |
|----|------|-------|------|
| 1  | 1    | 0900  | 1000 |
| 2  | 1    | 0900  | 1000 |

**And** the E360 schedule is as follows:

| id | room | start | end |
|----|------|-------|-----|

**When** events are received:

| id | room | start | end  | action | 
|----|------|-------|------|--------|
| 3  | 1    | 0900  | 1000 | I      |
| 3  | 1    | 1000  | 1100 | U      |

**Then** the E360 schedule will be as follows:

| id | room | start | end  |
|----|------|-------|------|
| 3  | 1    | 1000  | 1100 |
