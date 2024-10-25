## Enhancing Visibility with the JavaScript SDK

Running the query specified [here](../README.md) with JavaScript and Dynatrace SDK further enhances visibility and provide insights into the current status of batch job executions.  

### Logic breakdown to process BatchJob Data

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

With the above logic, we can create a dynamic dashboard that provides insights into batch jobs, as illustrated in the GIF file at ![Batch Job Insights](../img/BatchJobStatus.gif).  

## Impact of the BatchJobs on application landscape  
By utilizing Dynatrace Lakehouse and context awareness, we can assess the impact of various runs on the application landscape, which includes application hosts, db and message queues.

In the [BatchServer](../img/BatchServer.jpg) dashboard, the server sends a business event at the start and completion of all job runs in an execution which helps to determine the start and end of batch execution.    

```javascript
        try {
            sendBizEvent("STARTED", parentId);

            List<Map<String, Object>> accounts = fetchAccounts();

            String fetchedFileName = saveAccounts(accounts, "fetched_accounts");

            /* Process the account information in a batch of 1000 jobs */
            processAccountsInParallel(accounts);

            if (random.nextDouble() < 0.3) {
                throw new RuntimeException("Simulated random job failure");
            }

            logJobStatus(mainProcessId, "ended normally");
            sendBizEvent("COMPLETED", parentId);
        } catch (Exception e) {
            e.printStackTrace();
        }
```

The above code sends bizevent with the parentId to indicate start and completion of the 1000 jobs. Alternatively, unique `parentId` can be utilized to identify runs of different jobs and assess their impact on the landscape. 







