# Real-Time-Data-Streaming-Analytics-for-Fraud-Detection
![Untitled Diagram drawio (8)](https://github.com/user-attachments/assets/66285321-d0c1-44cb-bf2e-2b7e0e3aa000)

## Project Overview
This project implements a real-time data streaming analytics solution to detect fraudulent transactions using Azure technologies. The solution uses Azure Event Hub for real-time ingestion, Azure Stream Analytics for processing, Azure Blob Storage for staging, and Azure Cosmos DB for storing clean and structured data for further analysis.

## Architecture Components
- **Azure Event Hub**: Ingests real-time streaming data.
- **Azure Stream Analytics**: Processes incoming data using SQL-like queries to identify high-value and rapid transactions.
- **Azure Blob Storage**: Temporary storage for cleaned output.
- **Azure Cosmos DB**: Stores structured transaction data for analysis.

## Step-by-Step Implementation

### 1. Event Hub Setup
- Created a namespace: `frauddetectionnamespace`
- Created an Event Hub: `fraud-stream`

### 2. Storage Account Setup
- Created Storage Account: `fraudstoragebootcamp4`
- Created two containers:
  - `fraud-output` (for cleaned output)
  - `cosmosoutput` (for Cosmos DB structured data)

### 3. Cosmos DB Setup
- Created account: `fraudcosmosbossman`
- Created database: `frauddb`
- Created container: `uniqueTransactions` with `/transaction_id` as Partition Key

### 4. Stream Analytics Job
- Created job: `fraud-detection-job`
- Inputs:
  - `transactionsinput` from Event Hub `fraud-stream`
- Outputs:
  - `cleanedoutput` to Blob Storage container `fraud-output`
  - `cosmosoutput` to Cosmos DB container `uniqueTransactions`

### 5. Query Used in Stream Analytics
```sql
WITH CleanData AS (
    SELECT *
    FROM transactionsinput
    WHERE transaction_id IS NOT NULL
      AND user_id IS NOT NULL
      AND amount IS NOT NULL
      AND ip_address IS NOT NULL
      AND TRY_CAST(amount AS float) IS NOT NULL
),

HighAmountFraud AS (
    SELECT 
        transaction_id,
        user_id,
        amount,
        timestamp,
        currency,
        location,
        device,
        ip_address,
        'HighAmountFraud' AS fraud_type
    FROM CleanData
    WHERE amount > 5000
),

RapidTransactionFraud AS (
    SELECT 
        NULL AS transaction_id,
        user_id,
        NULL AS amount,
        System.Timestamp AS timestamp,
        NULL AS currency,
        NULL AS location,
        NULL AS device,
        ip_address,
        'RapidTransactionFraud' AS fraud_type
    FROM CleanData
    GROUP BY
        user_id,
        ip_address,
        TumblingWindow(minute, 10)
    HAVING COUNT(*) >= 3
),

CombinedFrauds AS (
    SELECT * FROM HighAmountFraud
    UNION ALL
    SELECT * FROM RapidTransactionFraud
),

UniqueTransactions AS (
    SELECT DISTINCT
        transaction_id,
        user_id,
        amount,
        timestamp,
        currency,
        location,
        device,
        ip_address
    FROM CleanData
)

-- Output to Blob Storage
SELECT * INTO cleanedoutput FROM CombinedFrauds;

-- Output to Cosmos DB
SELECT * INTO cosmosoutput FROM UniqueTransactions;
```

## Testing
- Sample messages were sent to Event Hub from Azure Portal's Data Explorer.
- Data was processed and visible in Stream Analytics Test Preview.
- Processed data appeared successfully in both Blob Storage and Cosmos DB.

## Result
- Cosmos DB `uniqueTransactions` container was populated with clean, structured transaction data.
- Blob Storage `fraud-output` container received fraud detection results.

## Difficulties Faced
1. **Cosmos DB output was not receiving data initially**:
   - Output alias mismatch (`cosmosoutput` vs `output`).
   - Missing or incorrect partition key caused container creation errors.
   - Manual recreation of container was needed with correct `/transaction_id` partition key.

2. **Stream Analytics job not compiling**:
   - Used `System.Timestamp` without windowing.
   - Forgot aliasing subqueries.
   - Wrong usage of `GROUP BY` and `MAX()` in the same query.

3. **Test data not triggering expected results**:
   - Sent events before starting job.
   - Event schema issues (bad JSON format, extra data, etc.).

4. **Manual backup plan**:
   - When Cosmos DB still didn’t show results after fixing outputs, fallback was used by manually uploading test results to Cosmos DB via Data Explorer.

## Conclusion
This project demonstrates a robust real-time fraud detection pipeline using Azure’s serverless tools. The use of SQL-like queries in Stream Analytics simplifies stream processing, and Cosmos DB enables low-latency querying of structured results.


