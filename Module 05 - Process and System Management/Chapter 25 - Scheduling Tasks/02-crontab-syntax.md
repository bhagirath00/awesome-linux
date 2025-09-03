# Chapter 25: Scheduling Tasks - Crontab Syntax and Expressions

## 1. The Anatomy of a Crontab Line

A standard user crontab entry consists of **five time-and-date fields** followed by the command to be executed:

```
 * * * * *  command_to_execute
 - - - - -
 | | | | |
 | | | | +----- Day of the week (0 - 7) (Sunday is 0 or 7)
 | | | +------- Month of the year (1 - 12)
 | | +--------- Day of the month (1 - 31)
 | +----------- Hour of the day (0 - 23)
 +------------- Minute of the hour (0 - 59)
```

> [!NOTE]
> In **System-wide** crontabs (such as `/etc/crontab` and `/etc/cron.d/*`), there is a **6th field** specifying the username under which the command executes before the command itself:
> ```
> # Minute Hour Day Month DayOfWeek   USER   Command
>   0      2    *   *     *           root   /opt/scripts/backup.sh
> ```

---

## 2. Special Characters and Operators

Crontab expressions support four versatile operators:

| Operator | Character | Meaning | Example |
| :--- | :--- | :--- | :--- |
| **Wildcard** | `*` | Matches **every** possible value for that field. | `* * * * *` (Every minute) |
| **Comma** | `,` | Value list separator (runs on specific listed numbers). | `15,45 * * * *` (At minutes 15 and 45) |
| **Hyphen** | `-` | Defines an inclusive integer range. | `0 9-17 * * *` (Every hour from 9 AM to 5 PM) |
| **Slash** | `/` | Step / interval value (skips every `$N` units). | `*/15 * * * *` (Every 15 minutes) |

---

## 3. Common Production Schedule Patterns

```bash
# 1. Run every single minute (Health checks)
* * * * * /usr/local/bin/ping-service.sh

# 2. Run every 5 minutes
*/5 * * * * /usr/local/bin/check-queue.sh

# 3. Run every hour at the top of the hour (00)
0 * * * * /usr/local/bin/hourly-summary.sh

# 4. Run every day at 02:30 AM (Nightly backups)
30 2 * * * /usr/local/bin/db-backup.sh

# 5. Run twice a day: at 06:00 AM and 06:00 PM (18:00)
0 6,18 * * * /usr/local/bin/sync-inventory.sh

# 6. Run Monday through Friday at 08:00 AM (Business days only)
0 8 * * 1-5 /usr/local/bin/send-daily-digest.sh

# 7. Run on the 1st of every month at midnight (00:00)
0 0 1 * * /usr/local/bin/monthly-billing.sh
```

---

## 4. Special Predefined Shortcut Macros

Cron provides predefined human-readable shortcuts that replace the 5-field syntax:

| Macro String | Equivalent Expression | Meaning |
| :--- | :--- | :--- |
| `@reboot` | None (Special event) | **Executes once immediately when the system boots up.** |
| `@yearly` / `@annually`| `0 0 1 1 *` | Run once a year at midnight on January 1st. |
| `@monthly` | `0 0 1 * *` | Run once a month at midnight on the first day. |
| `@weekly` | `0 0 * * 0` | Run once a week at midnight on Sunday. |
| `@daily` / `@midnight` | `0 0 * * *` | Run once a day at midnight (00:00). |
| `@hourly` | `0 * * * *` | Run once an hour at the top of the hour. |

### Practical `@reboot` Example:
```crontab
# Start a custom monitoring background agent upon every boot
@reboot /opt/agent/bin/monitor-daemon > /var/log/agent.log 2>&1
```

---

## 5. Summary Takeaways

- The five fields represent: **Minute (0-59)**, **Hour (0-23)**, **Day (1-31)**, **Month (1-12)**, **Day-of-Week (0-7)**.
- Use `*/N` for interval steps (e.g., `*/10 * * * *` for every 10 minutes).
- System-wide crontabs (`/etc/crontab`, `/etc/cron.d/`) require an extra user field before the command.
- Use `@reboot` to trigger initialization scripts automatically upon system startup.
