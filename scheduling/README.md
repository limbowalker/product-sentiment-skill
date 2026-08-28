# Scheduling the sentiment report (macOS)

opencode skills are invoked, not autonomous — the skill can't schedule itself.
Use `launchd` to run opencode headless on a schedule with a fixed prompt.

## 1. Headless run command

opencode can run a single prompt non-interactively. The prompt just needs to
trigger this skill:

```bash
opencode run "Run the product sentiment report using the product-sentiment-skill skill. Follow its full run procedure and write both output artifacts."
```

Verify the exact headless flag for your opencode version with `opencode --help`
(commonly `opencode run "<prompt>"`). Test it once by hand before scheduling.

## 2. launchd job

Edit `com.user.sentiment-report.plist` in this folder:

- Set the correct absolute path to the `opencode` binary (find it with
  `which opencode`).
- Adjust the schedule (`StartCalendarInterval`). The example runs daily at 09:00.
- Set `WorkingDirectory` so relative skill paths resolve.

Install it:

```bash
cp com.user.sentiment-report.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.user.sentiment-report.plist
```

Check / unload:

```bash
launchctl list | grep sentiment-report
launchctl unload ~/Library/LaunchAgents/com.user.sentiment-report.plist
```

Logs go to `scheduling/run.out.log` and `scheduling/run.err.log`.

## Alternative: cron

```cron
0 9 * * * cd ~/path/to/product-sentiment-skill && /absolute/path/to/opencode run "Run the product sentiment report using the product-sentiment-skill skill." >> scheduling/run.out.log 2>&1
```
