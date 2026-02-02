# Heartbeat checklist

> This file is read every heartbeat cycle (default: 30m).
> Keep it short — every line adds tokens to the session context.
> Reply HEARTBEAT_OK if nothing needs attention. Omit it entirely for alerts.

## Inbox & messages
- Scan inbox for anything urgent or time-sensitive
- Check for messages waiting > 2 hours without a reply
- Check if anyone replied to threads I'm tagged in

## Monitoring
- Verify production services are healthy (status page / uptime endpoint)
- Check if any CI/CD pipelines are failing on main branch
- Look for approved PRs that haven't been merged

## Tasks & follow-ups
- Review open tasks for approaching deadlines (next 24 hours)
- Flag blocked items and note what's needed to unblock them
- Check progress on tasks discussed earlier in this session

## Alert rules
- Only alert if something genuinely needs immediate attention
- Batch non-urgent items into the next check-in
- If daytime and nothing pending, reply HEARTBEAT_OK
- Never wake the user during configured quiet hours
