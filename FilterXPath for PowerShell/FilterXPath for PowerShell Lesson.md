# Basic Syntax
### Basic Syntax for PowerShell
So, the ***basic Syntax of the powershell commands*** goes as follows:\
`Get-WinEvent [ -Path | -LogName ] -FilterXPath "*[XPath Query]"`

Let's break it down.
- `Get-WinEvent`: Gets events from event logs and event tracing log files on local and remote computers.
- `-Path`: Specifies the path to an .evtx file you want to analyze. Ex: `-Path C:\Users\Example\Example-ex`.
- `-LogName`: Specifies the internal log you want to analyze. Ex: `-LogName Microsoft-Windows/Application`.
  - Use `-Path` OR `-LogName` depending on what you're analyzing. Do not use both.
- `-FilterXPath`: Specifies an XPath query. Ex: `-FilterXPath "*[System/EventID=1337]`.

Thats the barebones for building the PowerShell command.

### Basic Syntax for XPath Queries
Now we will take a look at the ***basic syntax of the XPath queries*** themselves.\
The basic syntax for an XPath query looks like this:
`"*[.../.../...]"`.\
Every XPath query starts with `"*"`. We put `"*"` at the beginning becasuse it means "match any node" at the current level, without caring about the node's name.

Lets put the example into the context of the XML you'd find in an event log.\
Below is a simplified version of what you'd see in an event log if you go to the details tab and switch to XML view:
```xml  
<Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
  <System>
    <Provider Name="Service Control Manager" />
    <EventID>7040</EventID>
    <Level>4</Level>
    <TimeCreated SystemTime="2024-04-01T12:00:00.0000000Z" />
    <Computer>Server01</Computer>
  </System>
  <EventData>
    <Data Name="param1">Windows Service Name</Data>
    <Data Name="param2">manual</Data>
  </EventData>
</Event>
```
Now, for writing the XPath query, you need to order it the same way you would navigate directories in the CLI.\
To get to  EventID, we have to go through Event, to Systems, to EventID.
```pgsql
Event
 ├── System
 │    ├── Provider (Name = "Service Control Manager")
 │    ├── EventID (7040)
 │    ├── TimeCreated
 │    └── Computer (SERVER01.domain.local)
 └── EventData
      ├── Data (Name="param1") = "Spooler"
      ├── Data (Name="param2") = "auto"
      └── Data (Name="param3") = "manual"
```

So, if I wanted to create a query to filter for ***every event with an Event ID of 7040***, it would look like this:\
`"*[System/EventID=7040]"`.
 - `Event` and `*` can be used interchangeably. However you'll see `*` way more commonly used.

Putting it all into a full PowerShell command would look like this:
```PowerShell
Get-WinEvent -LogName Microsoft-Windows/Application -FilterXPath "*[System/EventID=7040]"
```
Not so bad right?
I think thats good enough for the basic syntax. Lets move on to more advanced stuff.

# Advanced Syntax
### Advanced Syntax for XPath Queries
Alright, now we're getting to the stuff that might be confusing.
Lets use a slightly more complex XML example for this section:
```xml
<Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
  <System>
    <Provider Name="Microsoft-Windows-Security-Auditing" Guid="{54124585-2635-4994-a5ba-3e3b9958c30d}" /> 
    <EventID>4798</EventID> 
    <Version>0</Version> 
    <Level>0</Level> 
    <Task>13824</Task> 
    <Opcode>0</Opcode> 
    <Keywords>0x8020000000000000</Keywords> 
    <TimeCreated SystemTime="2023-05-11T00:20:06.1963204Z" /> 
    <EventRecordID>1990615</EventRecordID> 
    <Correlation ActivityID="{6580bbd3-b48c-000e-efbb-80658cb4db01}" /> 
    <Execution ProcessID="952" ThreadID="2108" /> 
    <Channel>Security</Channel> 
    <Computer>SRVR-DSKTP01</Computer> 
    <Security /> 
  </System>
  <EventData>
    <Data Name="TargetUserName">WDAGUtilityAccount</Data> 
    <Data Name="TargetDomainName">SRVR-DSKTP01</Data> 
    <Data Name="TargetSid">S-1-4-62-1529298597-1620866952-269215817-504</Data> 
    <Data Name="SubjectUserSid">S-1-4-13</Data> 
    <Data Name="SubjectUserName">SRVR-DSKTP01$</Data> 
    <Data Name="SubjectDomainName">WORKGROUP</Data> 
    <Data Name="SubjectLogonId">0x3b6</Data> 
    <Data Name="CallerProcessId">0x1058</Data> 
    <Data Name="CallerProcessName">C:\Program Files\Malwarebytes\Anti-Malware\MBAMService.exe</Data> 
  </EventData>
</Event>
```
Lets take the PowerShell command at the end of the Basic Syntax section and work off of it.
```PowerShell
Get-WinEvent -LogName Microsoft-Windows/Application -FilterXPath "*[System/EventID=7040]"
```
How would we change the PowerShell command to filter for logs that contain an EventID of 4798 ***and*** a username of SRVR-DSKTP01?

Well first, we already know how to filter based on EventID, so lets just change it to the ID we're looking for: `"*[System/EventID=7040]"` becomes `"*[System/EventID=4798]"`.\
Next, we can use the parameters `and` and `or` to further hone in our search: `"*[System/EventID=4798] and *[????????????]"`.
- Take note of the double quotes. We put double quotes around the entire query, not each part of the query.
 - Using the `and` parameter means the filter will show every log that has an EventID of 4798 ***and*** a username of SRVR-DSKTP01.
 - Using the `or` parameter means the filter will show every log that has ***either*** an EventID of 4798 ***or*** a username of SRVR-DSKTP01.

So now, how would we write the syntax for filtering logs with the username SRVR-DSKTP01?\
Looking at the example XML. We can see SRVR-DSKTP01 under `<Data Name="SubjectUserName">SRVR-DSKTP01$</Data>` within `<EventData>`.\
Using what we know of the syntax so far, we can create the beginning of the XPath query `"*[EventData/Data]"`. However, theres multiple parameters under `/Data`, so how do we specify the username?\
We use the syntax `/Data[@Name='']`.

If we add that in, the query becomes `"*[EventData/Data[@Name='']]"`.\
Now, we can see that SRVR-DSKTP01 is the value for `<Data Name="SubjectUserName">`. Using this, the syntax becomes `"*[EventData/Data[@Name='SubjectUserName']]"`

Ok, we're almost there.
All we have to do now is specify what username were looking for with `='SRVR-DSKTP01'`.\
Add that last bit in, the XPath query would look like this: `"*[EventData/Data[@Name='SubjectUserName'] ='SRVR-DSKTP01']"`.

Now that we have that, lets combine it all together into the full PowerShell command:
```PowerShell
Get-WinEvent -LogName Microsoft-Windows/Application -FilterXPath "*[System/EventID=4798] and *[EventData/Data[@Name='SubjectUserName'] ='SRVR-DSKTP01']"
```
Once again, take note of the double quotes. Its an easy mistake to make when you're building out the query.

That seems like enough info on writing more advanced queries. You can add as many `and` or `or` parameters as you want to further hone your search. The syntax structure stays the same.

Now can get into more advanced PowerShell usage.
### "Advanced" PowerShell Usage with XPath Queries
When I say "advanced", it's not so much advanced as it is just utilizing more PowerShell cmdlets alongside the XPath queries.
We'll start with a list of useful cmdlets that enhance our log analysis abilities:
- `PLACEHOLDER`
