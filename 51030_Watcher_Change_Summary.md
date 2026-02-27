# 51030 - Cost Anomaly Watcher: Change Summary

## Request Summary

The request was to modify the ticket routing logic in the `sdpmt_51030` watcher so that when a new alert fires for a hostname that already has an open ticket, the system compares the **priority** of the new alert against the **priority** of the existing open ticket. If the new alert is more severe (priority escalation), a new ticket should be created instead of commenting on the existing one.

---

## Scope of Changes

Only the **transform script** was modified. No changes were made to the trigger, inputs, condition, actions, or metadata.

---

## Pseudo-Code: Before vs After

### BEFORE

```
Build map: opened = { hostname → ticket_id }  (from open_tickets search)

For each alert (hostname):
    Determine ticket_priority from total_impact
    
    IF hostname has an open ticket:
        → COMMENT on existing ticket
    ELSE:
        → CREATE new ticket
```

### AFTER

```
Build map: opened = { hostname → { ticket_id, priority } }  (from open_tickets search)

For each alert (hostname):
    Determine ticket_priority from total_impact
    
    IF hostname has an open ticket:
        Extract numeric priority from new alert      (e.g., "1 - Critical" → 1)
        Extract numeric priority from existing ticket (e.g., "3 - Moderate" → 3)
        
        IF new priority number < existing priority number (escalation):
            → CREATE new ticket
            (e.g., existing P3, new P1 → new ticket)
            (e.g., existing P2, new P1 → new ticket)
        ELSE (same or de-escalation):
            → COMMENT on existing ticket
            (e.g., existing P1, new P2 → comment)
            (e.g., existing P2, new P2 → comment)
    ELSE:
        → CREATE new ticket
```

---

## Actual Code Changes

### Change 1: `opened` Map Construction

**BEFORE:**
```painless
Map opened = ctx.payload.open_tickets.hits.hits.stream().collect(
  Collectors.toMap(
    h -> h._source.host.target,
    h -> h._source.ticket_id
  )
);
```

**AFTER:**
```painless
Map opened = ctx.payload.open_tickets.hits.hits.stream().collect(
  Collectors.toMap(
    h -> h._source.host.target,
    h -> ['ticket_id': h._source.ticket_id, 'priority': h._source.priority]
  )
);
```

**What changed:** The map value was changed from a simple `ticket_id` string to a map containing both `ticket_id` and `priority`. The `priority` field is read from the `sdpmt_tickets` index (confirmed to exist with values like `"1 - Critical"`, `"2 - High"`, `"3 - Moderate"`, `"4 - Low"`).

---

### Change 2: Ticket Routing Decision Logic

**BEFORE:**
```painless
if (opened.containsKey(host.key)) {
    host.ticket_id = opened[host.key];
    to_comment.add(host);
} else {
    to_open.add(host);
}
```

**AFTER:**
```painless
if (opened.containsKey(host.key)) {
    def existing = opened[host.key];

    // Extract numeric priority: '1 - Critical' -> 1, '2 - High' -> 2, etc.
    int new_prio = Integer.parseInt(ticket_priority.substring(0, 1));
    int existing_prio = Integer.parseInt(existing.priority.substring(0, 1));

    if (new_prio < existing_prio) {
        // Priority escalation → create a NEW ticket
        to_open.add(host);
    } else {
        // Priority same or de-escalated → comment on existing ticket
        host.ticket_id = existing.ticket_id;
        to_comment.add(host);
    }
} else {
    to_open.add(host);
}
```

**What changed:** Instead of always commenting when an open ticket exists, the logic now extracts the leading digit from both the new alert's priority string and the existing ticket's priority string, then compares them. A lower number means higher severity, so `new_prio < existing_prio` indicates escalation, which triggers a new ticket instead of a comment.

---

## What Was NOT Changed

- **Trigger:** Still runs every 30 minutes
- **Input (sdpmt):** Unchanged — still queries `sdpmt_amdb` for alert status
- **Input (errors):** Unchanged — still queries `prod:filebeat-*-mt-*` for cost anomaly data
- **Input (targets_to_alert_on):** Unchanged — still flattens bucket keys
- **Input (open_tickets):** Unchanged — still queries `sdpmt_tickets` for open tickets (the `priority` field was already present in the index; no `_source` filter was restricting it)
- **Condition:** Unchanged — still checks bucket size > 0 and alert_status == 'enabled'
- **Action (comment_tickets):** Unchanged
- **Action (open_tickets):** Unchanged
- **Metadata:** Unchanged
