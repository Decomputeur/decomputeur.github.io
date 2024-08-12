---
title: Report Manager Owner Check
date: 2024-08-07 13:00:00 +0200
categories: [Report Manager, N-Central]
tags: [report manager, n-central]
image:
  path: /assets/images/logos/ReportManager.png
  alt: Report Manager Logo.
---

After a few reports that failed to be send from the N-Central Report maanger, we went to investigate why those reports weren't sent to our customers.

A few hours into hour investigation we noticed that all the reports that weren't owned by an existing, non-deleted user, were sent without any problems and all reports that exhibited this issue were owned by a user that we had previously deleted from N-Central.
By deleting the user from N-Central, you mark the user as deleted in Report Manager and thus all reports attached to that specific user won't be able to send anymore.

By digging around in the underlying SQL, we found that the table RptSubscriptions in the config database, has a field named owner.

As soon as we replaced the owner in the Report Manager for the non-working reports with a owner that is still active, this number indeed changed.
After some more trial and error, we decided to replace the owner of all reports by our reports-admin user, thus eliminating this issue entirely.

After a few months, we had a customer complaining about not receiving any reports. A quick glance at the owner of these reports revealed the same issue. We deleted a user from N-Central that was now marked as deleted in the Report Manager.

Quickly we decided to change those non-functioning reports back to the reports-admin user.

A internal meeting later, we decided that to combat any issues in the future, everybody must create the reports he/she needs under the reports-admin user, but this doesn't fix if someone forgets this rule and still use it's own account to create a report.

To combat this, we created a simple and quick Automation Policy that executes the following SQL query on the config table to get a count of all reports that doesn't have the reports-admin user set as owner.

```SQL
SELECT COUNT(*) FROM [config].[dbo].[RptSubscriptions] WHERE [RptSubscriptions].OwnerID != 1;
```

This will retreive the total number of reports that don't have the reports-admin as it's owner.

Download AMP for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Report%20Manager%20Owner%20Check/Report%20Manager%20Owner%20Check.amp)

Download XML file for direct import into N-Central: [GitHub](https://github.com/eagle00789/N-Central/blob/master/Report%20Manager%20Owner%20Check/Report%20Manager%20Owner%20Check.xml)