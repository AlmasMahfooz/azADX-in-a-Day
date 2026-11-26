Because the official Microsoft Azure Blob Storage containers referenced in the lab instructions were decommissioned when the workshop was retired, the One-Click Blob/Container ingestion step fails. 
Uploading the files locally bypasses that broken link completely while keeping the exact same dataset and schema.

Ingest the records into logsRaw directly from the GitHub mirror <br>
**https://github.com/Azure/ADX-in-a-Day-Lab1/tree/main/Data**<br>
by using the kusto ingest command instead of the retired Azure Blob Storage link.<br>
```kusto
.ingest into table logsRaw (
    h@"https://raw.githubusercontent.com/Azure/ADX-in-a-Day-Lab1/main/Data/TelemetryLogs_File1.csv.gz"
) with (ignoreFirstRecord=true)
```
But this only brings 356 rows.

```kusto
logsRaw
| count
```
Copy any record with property value.

```
"Timestamp": 2014-03-08T13:00:01.6638235Z,
"Source": IMAGINEFIRST0,
"Node": Engine000000000899,
"Level": Information,
"Component": INGESTOR_EXECUTER,
"ClientRequestId": 5a848f70-9996-eb17-15ed-21b8eb94bf0e,
"Message": IngestionCompletionEvent: finished ingestion file path: "https://benchmarklogs3.blob.core.windows.net/benchmark/2014/IMAGINEFIRST0_1399_46.json.gz",
"Properties": {
	"size": 1567064757,
	"format": "json",
	"rowCount": 892405,
	"cpuTime": "07:06:59.4469410",
	"duration": "00:01:53.6312940"
}
```
Provide it to Copilot for GitHub and ask it to create few more records, with similar kind of data to ingest in the table, for some rows property field is empty.

```kusto
.set-or-append logsRaw <|
range n from 1 to 120 step 1
| extend
    Timestamp = datetime_add("second", n, datetime(2014-03-08 13:00:00.0000000)),
    Source = case(n % 4 == 0, "IMAGINEFIRST0",
                  n % 4 == 1, "IMAGINEFIRST1",
                  n % 4 == 2, "IMAGINEFIRST2",
                  "IMAGINEFIRST3"),
    Node = case(n % 3 == 0, "Engine000000000899",
                n % 3 == 1, "Engine000000000900",
                "Engine000000000901"),
    Level = "Information",
    Component = "INGESTOR_EXECUTER",
    ClientRequestId = tostring(new_guid())
| extend
    Message = strcat(
        "IngestionCompletionEvent: finished ingestion file path: ",
        "\"https://example.blob.core.windows.net/benchmark/2014/",
        Source, "_", tostring(n), "_46.json.gz\""
    ),
    Properties = iff(
        n % 6 == 0,
        dynamic({}),
        pack(
            "size", 1567064757 + n * 1000,
            "format", "json",
            "rowCount", 892405 + n * 17,
            "cpuTime", tostring(7h + n * 1m),
            "duration", tostring(1m + n * 1s)
        )
    )
| project Timestamp, Source, Node, Level, Component,
          ClientRequestId, Message, Properties
```

This appends the rows to the table; it doesn’t replace existing data. The file paths and metrics are generated examples, not actual ingestion results. This assumes logsRaw has matching columns and types, with Properties defined as dynamic.<br>
To confirm the rows were added:

```kusto
logsRaw
| summarize TotalRows = count()
```
