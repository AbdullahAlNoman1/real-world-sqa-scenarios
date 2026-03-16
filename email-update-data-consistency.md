# Email Updated in UI but Emails Still Go to Old Address

# Scenario
A user updates their email address in their profile successfully.  
The UI shows the new email everywhere.  
However, password reset links and promotional emails are still being sent to the old email address even days later.

# Assumptions
- User profile update request returns success
- UI reads email from at least one updated source
- Password reset and promotional emails may use different services/data sources
- The system may include caches, background jobs, replicas, and third-party email tools
- Email update should propagate consistently across all dependent systems

# Objective
Identify which systems may be out of sync and validate data consistency across:
- primary databases
- read replicas
- caches
- authentication systems
- marketing/email platforms
- background jobs and event pipelines


# Which Systems Could Be Out of Sync?

# 1. Primary profile database vs other service databases
The profile service may update the main user table, but other systems may still store the old email in:
- auth database
- CRM database
- marketing database
- notification service database
- support/admin database

# 2. UI data source vs backend operational data source
The UI may display the new email from:
- profile service
- frontend cache
- GraphQL/API aggregation layer

But password reset or email sending may still use:
- auth service user table
- legacy identity store
- marketing contact list

So the UI looking correct does not prove all systems are updated.


# 3. Cache still holding the old email
Possible caches:
- Redis
- in-memory service cache
- CDN/session cache
- profile cache in mobile/web app

Even if DB is updated, downstream services may still read stale cached data.


# 4. Read replica lag or wrong data source
Some services may be reading from:
- read replica
- reporting DB
- search index
- denormalized user table

If replication failed or jobs stopped, some services may still see the old email.


# 5. Authentication/identity service not updated
Password reset links are often handled by auth/identity service, not profile service.

Possible issue:
- profile email updated
- auth credential record still tied to old email
- reset flow still queries auth store


# 6. Third-party email/marketing platform not synced
Promotional emails may come from:
- Mailchimp
- SendGrid marketing
- HubSpot
- Customer.io
- Braze
- another ESP/CRM

If contact sync failed, that platform may still hold the old email.


# 7. Background sync/event failure
Email change may depend on event-driven propagation:
- `user_email_updated`
- `profile_updated`
- `contact_sync_requested`

If event publishing or consuming failed:
- profile updated
- downstream systems never received change


# 8. Search index / denormalized store not updated
Admin panel or communication systems may read from:
- Elasticsearch/OpenSearch
- analytics warehouse
- denormalized user snapshot tables

Those may still contain the old email.


# 9. Legacy system still in use
A legacy notification job may still be querying:
- old monolith DB
- old subscriber table
- CSV/imported contact list
- archived preferences store


# 10. Partial update logic
System may update:
- display email
but not:
- login email
- recovery email
- notification email
- marketing email

This happens when business rules separate identity email from contact email.

# How I Would Validate Data Consistency

# Step 1: Start with one real affected user
Collect:
- user ID
- old email
- new email
- profile update timestamp
- password reset attempt timestamp
- promotional email timestamp
- correlation/request ID if available

This helps trace one complete case across systems.


# Step 2: Validate source of truth
First determine:
Which system is supposed to be the source of truth for user email?

Possible source:
- profile service DB
- identity/auth DB
- central user service

Then verify whether that source contains:
- only new email
- both old and new email in different columns
- update history/audit log

# Validate
- user table
- audit log table
- email history/change log
- updated_at timestamp
- changed_by / request source


# Step 3: Compare all dependent databases
Check whether the same user has matching email in:
- profile DB
- auth DB
- marketing/contact DB
- notification DB
- admin/support DB
- subscription/preferences DB

# Validate for each store
- current email value
- update timestamp
- record version
- any soft-deleted old email entries
- duplicate contact records

# Bug indicator
If profile DB has new email but auth/marketing DB still has old email days later, sync is broken.


# Step 4: Validate password reset flow specifically
Password reset usually uses auth/identity system.

# Check:
- which table/service does password reset query?
- does reset service search by user ID or by email string?
- does token generation still map to old email?
- is old email still marked as verified/primary in auth store?

# Test
- request password reset after email change
- inspect logs for recipient email
- inspect auth service DB record
- inspect outbound email event payload

# Possible finding
UI updated profile email, but auth service still stores old login/recovery email.


# Step 5: Validate promotional email flow specifically
Promotional emails often come from third-party marketing systems.

# Check:
- where marketing campaigns get recipient list from
- whether contacts are synced by event, batch job, or CSV export
- whether old and new emails both exist as separate contacts
- whether unsubscribe/preferences are attached to old email record

# Validate
- contact record in third-party ESP
- sync job logs
- segmentation rules
- last successful sync timestamp
- failed webhooks or API calls

# Possible finding
Marketing platform still has old email because contact update failed or created a new contact instead of updating existing one.


# Step 6: Validate caches
Inspect:
- Redis keys
- user profile cache
- auth cache
- notification preference cache
- edge/session cache if relevant

# Validate
- cached email value
- TTL/expiry
- cache invalidation event after profile update
- whether some services bypass DB and read only from cache

# Bug indicator
Old email present in cache days later suggests invalidation failure or cache key mismatch.


# Step 7: Validate event/message flow
If the architecture is event-driven, inspect whether email change event was published and consumed.

# Events to check
- user_profile_updated
- user_email_updated
- contact_updated
- auth_identity_updated
- marketing_sync_requested

# Validate
- event published successfully
- payload contains new email and user ID
- all intended consumers processed it
- retries / dead-letter queue entries
- no schema mismatch

# Bug indicator
Event published but one consumer failed, leaving only some systems updated.


# Step 8: Validate background jobs / batch sync
Some systems update eventually through scheduled jobs.

# Check:
- sync job schedule
- last run time
- failed job history
- retry queue
- partial batch failures

# Test
- trigger manual sync for affected user
- compare before/after values

# Possible finding
Job stopped, failed silently, or excluded updated users due to filter bug.


# Step 9: Validate third-party email providers
For promotional and transactional emails, inspect provider-side data.

# Check:
- recipient email on sent event
- template event payload
- contact profile in provider
- suppression lists
- webhook delivery records

# Validate questions
- did our system send old email to provider?
- or did provider already have outdated contact data?
- was old email stored as a separate subscriber/contact?

# Step 10: Validate admin/support views
Support tools may read from different sources than the UI.

# Check:
- admin panel source DB
- support CRM
- search index
- customer timeline/contact history

# Why this matters
If support sees old email while UI shows new email, it confirms split data sources.

# Systems I Would Explicitly Validate

# Core systems
- Profile service
- User database
- Auth/identity service
- Password reset service
- Notification service

# Storage layers
- Primary DB
- Read replica
- Cache layer
- Search index
- Analytics/denormalized tables

# Integration systems
- Event bus / queue
- Cron/batch sync jobs
- Third-party email service
- CRM / marketing platform


# Logs I Would Check

# Application logs
- profile update API logs
- auth service logs
- password reset logs
- notification service logs
- marketing sync service logs

# Infrastructure logs
- message queue logs
- cache invalidation logs
- DB replication logs
- job scheduler logs

# Provider logs
- outbound email request logs
- webhook logs
- send event logs
- recipient resolution logs

# Data Points to Compare
For one affected user, compare these fields everywhere:
- user_id
- old_email
- new_email
- primary_email
- login_email
- recovery_email
- marketing_email
- verified_email
- updated_at
- version number
- sync status
- event timestamp

---

# High-Value Test Cases

# Test Case 1: Immediate consistency check
- change email from old to new
- verify UI shows new email
- trigger password reset
Expected:
- reset email goes only to new email


# Test Case 2: Delayed consistency check
- update email
- wait for expected async sync window
- trigger promo/test campaign
Expected:
- only new email receives communication

# Test Case 3: Old email should stop receiving
- update email
- trigger password reset and marketing email
Expected:
- old email must receive nothing


# Test Case 4: Cross-system verification
- query profile DB, auth DB, cache, and email platform for same user
Expected:
- all systems contain same effective email


# Test Case 5: Event failure simulation
- update email while blocking one consumer/service
Expected:
- monitoring should show failed sync
- inconsistent systems should be detectable and recoverable

---

# Test Case 6: Duplicate contact handling
- update email multiple times
Expected:
- third-party marketing service should not retain multiple active contacts incorrectly

---

# Possible Root Causes

- profile DB updated, auth DB not updated
- password reset service using legacy user table
- cache not invalidated
- read replica stale or broken
- event published but consumer failed
- marketing sync failed silently
- third-party ESP created new contact but kept old contact active
- old email still stored as recovery/primary email in another service
- UI reading from updated source while emails use different backend source


# What Would Indicate a Bug?

- UI shows new email, but password reset goes to old email
- promotional emails continue to old email after sync window
- different DBs show different emails for same user
- cache contains old email after update
- event exists but not consumed by all downstream services
- third-party ESP still has active old contact record
- support/admin views differ from profile system

---

## Risks
- user account recovery failure
- privacy/security issue if old email belongs to someone else
- compliance risk for sending communications to outdated address
- loss of trust
- marketing and transactional communication inconsistency
- support confusion due to mismatched data across tools

---

## Recommendations
To prevent this, the system should have:
- single source of truth for email
- clear ownership between profile email and auth email
- reliable event propagation
- cache invalidation on profile update
- reconciliation job to detect mismatched emails across systems
- audit log for email changes
- monitoring alert when old email is used after update

---

## Conclusion
The likely issue is that the profile update succeeded in one system, but downstream systems such as auth, cache, or third-party email platform remained stale.

I would validate consistency by tracing one affected user across:
- primary databases
- caches
- auth/password reset flow
- event pipeline
- background sync jobs
- third-party email provider records

The key goal is to confirm that after email update, **all services use the same current email as the effective communication address**.
