# Batch Jobs Overview  

Batch jobs are automated tasks designed to process data or perform operations without user interaction. They are typically:  

- Short-lived processes  
- Scheduled to run at specific times or triggered by events  
- Used for tasks like data processing, report generation, or system maintenance  

### Batch Job Syntax
The execution of a batch job is typically represented in logs with specific syntax that conveys the job name, process ID, status, and timestamp. Here is an example of how a typical batch job log entry may look:  
```
20240923/123311.421 U02000003 Job 'JOBS.ET_PULL_CC_STATUS' with RunID '2057540001340019' started.  
20240923/123312.421 U02000003 Job 'JOBS.ET_PULL_CC_STATUS' with RunID '2057540001340019' exited with error code 1.  
```

### Breaking it down  
- **`U02000003`**: The unique identifier (UUID) of the job, which is used to track its execution.
- **`JOBS.ET_PULL_CC_STATUS`**: The name of the batch job, specifying the task it is performing (in this case, pulling credit card status data).
- **`2057540001340019`**: The `RunID`, which is the unique process ID associated with this specific execution of the batch job.
- **`started`**, **`exited with error code 1`** or **`successfully completed.`**: These phrases indicate the status of the job, whether it has started or finished. An error code of `1` suggests the job exited with an issue, while `successfull` would indicate successful completion.

## Utilizing Dynatrace Lakehouse (Grail) for Batch Job Insights

Once logs are ingested into Dynatrace using methods such as `OneAgent`, `FluentD`, `oTel`, or other approaches, they can be parsed with DQL to extract valuable insights from batch job executions.  

**For example**, the following DQL query breaks down logs into key attributes, helping to monitor batch job activity. The results can also be visualized on dashboards with filters for further analysis:  
```javascript
fetch logs
| filter matchesPhrase(content, "JOBS") AND matchesPhrase(content, "RunID")
| parse content, "
LD 'JOBS.' WORD:Job
LD 'RunID ' STRING:RunId
LD:status"
| fields timestamp, Job, status, content, RunId
| filterOut status == "."
| fieldsAdd start_time=if(contains(content,"started."),timestamp)
| fieldsAdd end_time=if((contains(content,"ended normally")) or contains(content,"exited with error code 1."),timestamp)
| fields timestamp, Job, status, RunId, start_time, end_time
```

## Enhancing Visibility with the JavaScript SDK

Running the above query with the JavaScript SDK will further enhance visibility and provide insights into the current status of batch job executions.

### Logic to Process Batch Job Data

1. **Identify the unique property of each job and initialize its structure.**

   ```javascript
   const batch = {};

   /* Reiterate through each record and populate the data structure */
   for (const record of recordSet) {
     const runId = record['RunId'];
     if (!batch[runId]) {
       batch[runId] = {
         Job: record["Job"],
         run_id: runId,
         Status: "",
         JobStarted: null,
         JobEnded: null,
         Duration: "NA"
       };
     }
   }
   ```

2. **Process each job's start time, end time, and status from the DQL parsed output.**
```javascript
if (record["start_time"]) batch[runId].JobStarted = utcToLocal(record["start_time"]);
if (record["end_time"]) batch[runId].JobEnded = utcToLocal(record["end_time"]);

if (!statusLocked[runId]) {
  let status = record["status"]?.trim() || "";
  
  if (status.toLowerCase().includes("ended with return code")) {
    batch[runId].Status = "Failed";
    statusLocked[runId] = true;
  } else if (status == "started.") {
    batch[runId].Status = "Running";
  } else if (status == "ended normally.") {
    batch[runId].Status = "Completed without errors";
    statusLocked[runId] = true;
  } else {
    batch[runId].Status = status;
  }
}
```

3. **Update job status based on specific conditions (running, failed, completed).**
```javascript
/* Leverage pre-populated data to identify duration for the completed jobs */
for (const runId in batch) {
  const job = batch[runId];
  
  if (job.JobStarted && job.JobEnded) {
    const startTime = new Date(job.JobStarted);
    const endTime = new Date(job.JobEnded);
    const duration = endTime - startTime;
    
    job.Duration = `${duration / 1000} seconds`;
  }
}
```

With the above logic, we can create a dynamic dashboard that provides insights into batch jobs, as illustrated in the GIF file at ![Batch Job Insights](https://link_to_your_gif.gif).  

## Beyond dashboard and into Orchestration  





