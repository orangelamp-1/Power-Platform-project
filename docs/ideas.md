# Ideas

## Tasks
- Tasks can be childs of other tasks.
- Only allow a task to be closed/cancelled if all child tasks are closed.
- Add an option to close/cancel all child tasks with same closing notes as parent?
- Business process

## Submissions
- Canvas app for submitting new tasks for users.
- Form flow that can make tasks from a MS form. (Just for PoC)
- Setup a work queue where unassigned tickets are auto routed to.

## Security
Security role for user / agent
- Users can only use Canvas app.
- Users can see own submitted tasks in Canvas app.
- Agents can view queue and own tasks.
- One field for user notes, one field for agent notes.
- Closing notes (auto sent to the user when closed).
- A user is added to the access team of their own task? That was they don't need any permissions to general read?

## Due Dates
- Task priority sets the due date.
- Low/Medium/High/NA 5-3-1 working days (control with config)
- Recalucate due date if prio is changed. Account for bank holidays.
- If N/A then manually set due date.
- Need to store a submission date that can't be changed?

## Working Days
- Sch Flow that calls a Bank holiday API and store dates in holiday table, this table also allows for custom business related holidays days to be added, or mark a bank holiday as a working day. (Runs once/month or manual)

- A working day table that shows the working days in next 30 days.
- Use this for due dates as more efficient to just pull workingday[x] from table.
- Recalculate working day table every Monday 5am OR if there is a change to the holiday table where the holiday was within 30 days of today.

- Triggered flow - when a holiday is added or removed or editied - recalcaulte the working day table if holiday was within, then use that record to loop over all relevant records and recalcualte the due dates for Low/Medium/High.
- Will need to store the previous date in a second field to be able to see OG value if a holiday date is changed.

## Error/audit
- Error log table
- Use error handling flow to patch to - capture flow link, context, flow name, time etc.
- API audit table, capture request source, content, responce, timestanp, calling source address.
- Add both to admin console.
- Patch to error log table from canvas app - security role - create only for owned records
- Console.SessionID - userUPN, timestamp. Silently patch.