# EventForce Management System: Salesforce Implementation

A centralized Salesforce CRM solution for managing event planning operations: client bookings, event scheduling, venue reservations, vendor coordination, cancellation approvals, automated reminders, and feedback collection.

Built on the Salesforce platform using custom objects, Flows, an Approval Process, Apex (classes, triggers, Batch and Scheduled Apex), role-based security, and a Lightning App with Reports and Dashboards.

> **Note:** This project was developed and tested in a Salesforce **Developer Edition** org for academic purposes. It has not been deployed to a production environment.

---

## Table of Contents
- [Problem Statement](#problem-statement)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Data Model](#data-model)
- [Automation and Business Logic](#automation-and-business-logic)
- [Security Model](#security-model)
- [Reports and Dashboards](#reports-and-dashboards)
- [User Roles](#user-roles)
- [Setup Guide](#setup-guide)
- [Project Planning](#project-planning)
- [Advantages and Limitations](#advantages-and-limitations)
- [Future Scope](#future-scope)
- [Team](#team)
- [License](#license)

---

## Problem Statement
Event teams often track clients, venues, vendors, and feedback in spreadsheets or registers. This leads to:
- Fragmented information spread across multiple places
- Venues being **double-booked**
- Manual, easily missed client reminders
- Uncontrolled event cancellations
- Little visibility into operations for decision-making

EventForce solves these by moving the full event lifecycle onto one automated Salesforce platform.

## Features
| # | Feature | Description |
|---|---------|-------------|
| 1 | Event management | Create and track events with type, date, status, and budget |
| 2 | Client management | Store client contact details with email validation |
| 3 | Venue management | Track venue capacity, location, and availability status |
| 4 | Vendor management | Manage vendors by service type and status |
| 5 | Event–vendor assignment | Assign multiple vendors to an event (many-to-many via junction object) |
| 6 | Double-booking prevention | Apex trigger blocks two events on the same venue and date |
| 7 | Venue availability updates | Venue is marked *Reserved* when an event is confirmed and *Available* when canceled |
| 8 | Automated reminders | Record-triggered Flow emails the client 3 days before the event |
| 9 | Cancellation approvals | Approval Process with email notifications and status updates |
| 10 | Auto-completion of events | Scheduled Batch Apex marks past events as *Completed* nightly |
| 11 | Feedback management | Ratings and comments linked to both the event and the client |
| 12 | Reports and dashboards | Real-time view of upcoming events and operations |
| 13 | Role-based access | Profiles, roles, permission sets, OWD, and sharing rules |
| 14 | Data migration | Bulk import with the Data Import Wizard after pre-validation |

## Tech Stack
| Layer | Technology |
|-------|------------|
| Platform | Salesforce CRM (Developer Edition) |
| UI | Salesforce Lightning App ("Event Planner") |
| Database | Salesforce Custom Objects |
| Declarative logic | Record-Triggered Flows, Formula Fields, Validation Rules |
| Approvals | Salesforce Approval Process, Classic Email Templates |
| Programmatic logic | Apex Classes, Apex Triggers, Batch Apex, Scheduled Apex |
| Security | Profiles, Roles, Permission Sets, Sharing Rules, OWD |
| Reporting | Salesforce Reports and Dashboards |
| Data loading | Data Import Wizard |

## Data Model

### Custom Objects
| Object | API Name | Key Fields |
|--------|----------|------------|
| Event | `Event__c` | Event Name, Event Date, Event Type, Event Status, Event Budget (formula) |
| Client | `Client__c` | Client Name, Email, Phone, Address, Country, City |
| Vendor | `Vendor__c` | Vendor Name, Email, Phone, Service Type (multi-select), Status |
| Venue | `Venue__c` | Venue Name, Address, Location (URL), Capacity, Availability Status |
| Feedback | `Feedback__c` | Feedback ID (auto number, `F-{0000}`), Rating, Comments |
| Event Vendor | `EventVendor__c` | Junction object linking Event and Vendor |

### Relationships
| From | To | Type |
|------|----|------|
| Event | Client | Lookup |
| Event | Venue | Lookup |
| Feedback | Event | Lookup |
| Feedback | Client | Lookup |
| EventVendor | Event | Master-Detail |
| EventVendor | Vendor | Master-Detail |

### Picklist Values
- **Event Type:** Wedding, Corporate, Birthday, Anniversary, Festival, Concert, Other
- **Event Status:** Planned, Confirmed, Completed, Pending Cancellation, Canceled, Rejected
- **Vendor Service Type:** Catering, Decor, Photography, Videography, Lighting, Stage Setup, Makeup Artist

## Automation and Business Logic

### Validation Rule
- `Email_Valid_Address` on `Client__c` rejects invalid email addresses.

### Flow: Client Reminder – 3 Days Before
A record-triggered Flow on `Event__c` with a scheduled path (`3_Days_Before`) based on **Event Date**, offset 3 days before. It sends a reminder email to the client.

### Approval Process: `Event_Cancellation_Process`
- Cancellation requests are routed to the **Event Coordinator** role.
- Email templates notify the approver, the client (on cancellation), and the event owner (on approval or rejection).
- Final actions update the **Event Status** accordingly.

### Apex

| Component | Type | Purpose |
|-----------|------|---------|
| `VenueStatusHelper` | Class | Sets the venue to `Reserved` when an event is `Confirmed`, and back to `Available` when `Canceled` |
| `EventTrigger13` | Trigger (`after insert, after update` on `Event__c`) | Calls `VenueStatusHelper.updateVenueStatus` |
| `PreventDoubleBooking` | Trigger (`before insert, before update` on `Event__c`) | Prevents two events from being booked at the same venue on the same date |
| `BatchCompleteEvents` | Batch Apex | Finds events with a past date that are not yet `Completed` and marks them `Completed` |
| `ScheduleCompleteEvents` | Schedulable | Runs `BatchCompleteEvents` (batch size 200) |

The Scheduled Apex job **Daily Event Completion** runs every day at 8 PM.

```apex
public with sharing class VenueStatusHelper {
    public static void updateVenueStatus(List<Event__c> events){
        List<Venue__c> venues = new List<Venue__c>();
        for(Event__c ev : events){
            if(ev.Venue__c != null){
                if(ev.Event_Status__c == 'Confirmed'){
                    venues.add(new Venue__c(Id=ev.Venue__c, Availability_Status__c='Reserved'));
                } else if(ev.Event_Status__c == 'Canceled'){
                    venues.add(new Venue__c(Id=ev.Venue__c, Availability_Status__c='Available'));
                }
            }
        }
        if(!venues.isEmpty()) update venues;
    }
}
```

## Security Model
- **Organization-Wide Defaults:** Private
- **Role hierarchy:** Event Admin at the top, with Event Coordinator, Vendor Manager, and Client roles beneath it
- **Profiles:**
  - Event Admin (cloned from System Administrator)
  - Event Coordinator, Vendor Manager, and Client Profile (cloned from Standard Platform User)
  - Session timeout of 2 hours of inactivity and a minimum password length of 8
- **Permission Set:** Feedback Manager
- **Sharing Rule:** `Event_Sharing_For_Vendors` (criteria-based sharing of Event records)

## Reports and Dashboards
- **Lightning App:** *Event Planner* with tabs for Event, Client, Vendor, Venue, Feedback, Reports, and Dashboards
- **Report folder:** EventForce Report (shared with Event Coordinator and Vendor Manager roles as view-only)
- **Report:** *Upcoming Events by Month*, grouped by calendar month with row count and total budget, excluding canceled events
- **Dashboard:** *EventForce Operations Dashboard*

## User Roles
| Role | Responsibility |
|------|----------------|
| Client | Requests event services, gives feedback, views event updates |
| Event Coordinator | Manages bookings, scheduling, client communication, and approvals |
| Vendor Manager | Coordinates vendors and their assigned services |
| Venue Manager | Manages venue availability and reservations |
| Event Admin | Oversees the system, security, and data |

## Setup Guide

This project is configured in a Salesforce org. To reproduce it:

1. **Create a Developer Edition org** at [developer.salesforce.com/signup](https://developer.salesforce.com/signup) and verify your account.
2. **Create the custom objects** (`Event__c`, `Client__c`, `Vendor__c`, `Venue__c`, `Feedback__c`, `EventVendor__c`) and their custom tabs.
3. **Add fields and relationships** (picklists, formula field, lookups, master-detail on the junction object).
4. **Create the validation rule** `Email_Valid_Address` on `Client__c`.
5. **Create profiles, roles, and users** (Event Admin, Event Coordinator, Vendor Manager, Client).
6. **Create the email templates and the Approval Process** `Event_Cancellation_Process`.
7. **Build the Flow** *Client Reminder – 3 Days Before* and activate it.
8. **Add the Apex code** in the Developer Console: `VenueStatusHelper`, `EventTrigger13`, `PreventDoubleBooking`, `BatchCompleteEvents`, `ScheduleCompleteEvents`.
9. **Schedule the batch job:** Setup → Apex Classes → *Schedule Apex* → job name *Daily Event Completion*, class `ScheduleCompleteEvents`, daily at 8 PM.
10. **Create the Lightning App** *Event Planner* and add the objects, Reports, and Dashboards.
11. **Configure security:** OWD, Feedback Manager permission set, and the `Event_Sharing_For_Vendors` sharing rule.
12. **Load data** with the Data Import Wizard (add at least 10 records per object before building reports).
13. **Create the report and dashboard** described above.

## Project Planning
The project followed an Agile approach (Epics → User Stories → Story Points) across four sprints:

| Sprint | Focus | Story Points |
|--------|-------|--------------|
| Sprint 1 | Event management, client and venue association | 20 |
| Sprint 2 | Vendor management, notifications and automation | 20 |
| Sprint 3 | Cancellation approvals, venue management and Apex | 20 |
| Sprint 4 | Security and access control, reports and dashboards | 20 |

## Advantages and Limitations

**Advantages**
- Centralized, structured data for events, clients, vendors, venues, and feedback
- Automated reminders, approvals, and status updates reduce manual effort
- Double-booking prevention keeps venue schedules accurate
- Secure, role-based access to records
- Real-time operational visibility through reports and dashboards
- Scalable on the Salesforce platform

**Limitations**
- Built in a Developer Edition org; no production deployment performed
- Real-world deployment would need sandboxes, Change Sets, Salesforce DX, or CI/CD tooling
- Flows, Apex jobs, approvals, and sharing rules need ongoing monitoring and maintenance

## Future Scope
- Production deployment using sandboxes and DevOps tooling
- More advanced event automation
- Enhanced vendor tracking and performance management
- Additional reports and dashboards
- Integration with external systems
- Mobile accessibility
- Support for larger volumes of events, clients, vendors, venues, and users

## Team
| Name | Role |
|------|------|
| _Your Name_ | _Role_ |
| _Team Member 2_ | _Role_ |

## License
_Add your license here (for example, MIT) or state that this project was created for academic purposes._
