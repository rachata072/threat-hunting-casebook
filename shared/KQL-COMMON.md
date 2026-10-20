# Common KQL patterns

Reusable starters. Column names follow Sysmon and Windows Security conventions. Custom tables (`*_CL`) often rename or suffix columns, so run `getschema` first and adjust.

## Orient

```kql
// Schema
MyTable_CL | getschema

// Volume by host and event id
MyTable_CL
| summarize Events=count() by Computer, EventID
| order by Computer asc, Events desc

// Telemetry gaps per host (look for silent hours)
MyTable_CL
| summarize Events=count() by Computer, bin(EventTime, 1h)
| order by Computer asc, EventTime asc

// First and last event per host
MyTable_CL
| summarize First=min(EventTime), Last=max(EventTime), Events=count() by Computer
```

## Rare process lineage

```kql
// Parent-child pairs seen only a few times
MyTable_CL
| where EventID == 1
| summarize Count=count(), Hosts=dcount(Computer) by ParentImage, Image
| where Count <= 3
| order by Count asc
```

## Living-off-the-land binaries

```kql
let lolbins = dynamic(["certutil.exe","bitsadmin.exe","mshta.exe","rundll32.exe","regsvr32.exe","wmic.exe","schtasks.exe","sc.exe","vssadmin.exe","wevtutil.exe","bcdedit.exe","wbadmin.exe","powershell.exe","cmd.exe"]);
MyTable_CL
| where EventID == 1
| extend Exe = tolower(tostring(split(Image, "\\")[-1]))
| where Exe in (lolbins)
| project EventTime, Computer, User, Image, CommandLine, ParentImage
```

## Burst detection

```kql
// Many files created in a short window by one process
MyTable_CL
| where EventID == 11
| summarize Files=count(), Dirs=dcount(tostring(split(TargetFilename, "\\")[-2])) by Computer, Image, bin(EventTime, 1m)
| where Files > 200
```

## Beaconing

```kql
// Regular connection intervals from one process to one destination
MyTable_CL
| where EventID == 3
| sort by Computer asc, Image asc, DestinationIp asc, EventTime asc
| extend Delta = EventTime - prev(EventTime)
| where Computer == prev(Computer) and Image == prev(Image) and DestinationIp == prev(DestinationIp)
| summarize Conns=count(), AvgSec=avg(Delta / 1s), StdSec=stdev(Delta / 1s) by Computer, Image, DestinationIp
| where Conns > 10 and StdSec < AvgSec * 0.2
```

## Account spread

```kql
// Which hosts did an account log on to, and how
MyTable_CL
| where EventID == 4624
| summarize Logons=count(), Hosts=make_set(Computer) by TargetUserName, LogonType
| order by Logons desc
```

## Habits
- Filter by time and host first, then by event id.
- Project only the columns you need before screenshots.
- Save the query the moment it works.
