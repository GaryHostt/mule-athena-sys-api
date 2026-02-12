# AWS Athena MuleSoft Integration API

A production-ready MuleSoft API for integrating with AWS Athena, providing synchronous and asynchronous query execution, CTAS pattern for large datasets, and support for both Iceberg (transactional) and Parquet (analytical) use cases.

## Table of Contents

- [Prerequisites](#prerequisites)
- [AWS Account Setup](#aws-account-setup)
- [Installation](#installation)
- [Configuration](#configuration)
- [API Endpoints](#api-endpoints)
- [Usage Examples](#usage-examples)
- [Best Practices](#best-practices)
- [Error Handling](#error-handling)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Software Requirements

- **Mule Runtime**: 4.10.0 or higher
- **JDK**: 17
- **Maven**: 3.6+ (for building the project)
- **Anypoint Studio**: 7.x+ (optional, for development)

### AWS Requirements

- AWS Account with appropriate permissions
- AWS Athena service enabled in your region
- S3 bucket for query results storage
- IAM credentials with required permissions

## AWS Account Setup

### 1. Create S3 Bucket for Query Results

Athena requires an S3 bucket to store query results. Create a bucket following these steps:

1. Log in to AWS Console
2. Navigate to S3 service
3. Click "Create bucket"
4. Choose a unique bucket name (e.g., `my-athena-results-bucket`)
5. Select your preferred region
6. Configure bucket settings (versioning, encryption recommended)
7. Create the bucket

**Important**: Note the bucket name and region - you'll need these for configuration.

### 2. Create IAM User and Policy

#### Option A: Create IAM User with Access Keys

1. Navigate to IAM Console → Users
2. Click "Create user"
3. Enter a username (e.g., `mulesoft-athena-user`)
4. **Do NOT** attach policies directly to the user yet
5. Create the user
6. Click on the user → "Security credentials" tab
7. Click "Create access key"
8. Choose "Application running outside AWS"
9. Download or copy the Access Key ID and Secret Access Key

#### Option B: Use IAM Role (Recommended for CloudHub 2.0)

For CloudHub 2.0 deployments, consider using IAM role-based authentication:
- Configure AWS Transit Gateway
- Or deploy on Runtime Fabric (RTF) on AWS

### 3. Create IAM Policy

Create a policy with the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "athena:StartQueryExecution",
                "athena:GetQueryExecution",
                "athena:GetQueryResults",
                "athena:StopQueryExecution",
                "athena:ListQueryExecutions"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-athena-results-bucket/*",
                "arn:aws:s3:::your-athena-results-bucket"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "glue:GetTable",
                "glue:GetDatabase",
                "glue:GetPartitions"
            ],
            "Resource": "*"
        }
    ]
}
```

**Replace** `your-athena-results-bucket` with your actual bucket name.

### 4. Attach Policy to User

1. Go to IAM Console → Users → Select your user
2. Click "Add permissions" → "Attach policies directly"
3. Select the policy you created
4. Click "Next" → "Add permissions"

### 5. Create Athena Workgroup (Recommended for Production)

**⚠️ Production Best Practice**: Workgroups are **highly recommended** for production deployments. They provide critical governance, cost control, and query result location enforcement.

**Benefits**:
- **Cost Control**: Set per-query data scanned limits (e.g., 10GB max)
- **Governance**: Enforce query result location (specific S3 bucket)
- **Cost Allocation**: Tag queries for billing tracking
- **Query History**: Centralized query logging and monitoring
- **Performance**: Query result caching configuration

**Setup Steps**:

1. Navigate to Athena Console
2. Go to "Workgroups" → "Create workgroup"
3. Enter workgroup name (e.g., `mulesoft-production`)
4. Configure settings:
   - **Query result location**: Your S3 bucket path
   - **Data scanned limits**: Set per-query limit (e.g., 10GB)
   - **Tags**: Add cost allocation tags
5. Create the workgroup

## Installation

### 1. Install Athena JDBC Driver

The Simba Athena JDBC Driver is not available in Maven Central. You must install it manually:

#### Download the Driver

1. Visit: https://docs.aws.amazon.com/athena/latest/ug/connect-with-jdbc.html
2. Download the JDBC driver version **2.0.38** (used in this project)
   - **Current Version**: This project uses v2.0.38, which is compatible with Mule Runtime 4.10+ and JDK 17
   - **v3.x Series**: Available but not tested with this implementation. If upgrading to v3.x:
     - Verify compatibility with Mule Runtime 4.10+
     - Test all endpoints thoroughly
     - Update `pom.xml` dependency version
     - Review AWS documentation for breaking changes
3. Extract the JAR file (e.g., `AthenaJDBC42-2.0.38.jar`)

#### Install to Local Maven Repository

```bash
mvn install:install-file \
  -Dfile=AthenaJDBC42-2.0.38.jar \
  -DgroupId=com.simba.athena \
  -DartifactId=athena-jdbc \
  -Dversion=2.0.38 \
  -Dpackaging=jar
```

**Note**: Replace the version number with the version you downloaded.

#### Alternative: Install to Private Maven Repository

For production environments, publish the driver to a private Maven repository (Nexus, Artifactory, or Anypoint Exchange):

```bash
mvn deploy:deploy-file \
  -Dfile=AthenaJDBC42-2.0.38.jar \
  -DgroupId=com.simba.athena \
  -DartifactId=athena-jdbc \
  -Dversion=2.0.38 \
  -Dpackaging=jar \
  -Durl=http://your-nexus-repo/repository/maven-releases/ \
  -DrepositoryId=nexus
```

### 2. Build the Project

```bash
mvn clean install
```

### 3. Configure Application Properties

Edit `src/main/resources/application.properties` and replace placeholder values:

```properties
# AWS Region
aws.region=us-east-1

# AWS Credentials
aws.access.key=YOUR_ACTUAL_ACCESS_KEY
aws.secret.key=YOUR_ACTUAL_SECRET_KEY

# S3 Output Location
aws.s3.output.location=s3://your-athena-results-bucket/results/

# Athena Workgroup (recommended for production)
# Create a workgroup in Athena Console and specify the name here
aws.athena.workgroup=mulesoft-production
```

### 4. Deploy to Mule Runtime

#### Local Deployment

```bash
mvn mule:deploy
```

#### CloudHub 2.0 Deployment

1. Package the application: `mvn clean package`
2. Deploy via Anypoint Platform or Maven plugin
3. Configure environment variables in CloudHub 2.0:
   - `AWS_REGION`
   - `AWS_ACCESS_KEY`
   - `AWS_SECRET_KEY`
   - `AWS_S3_OUTPUT_LOCATION`
   - `AWS_ATHENA_WORKGROUP` (recommended for production)

**Important for CloudHub 2.0**:
- Whitelist Static Source IPs of your Private Space in AWS IAM/VPC policies
- Ensure TLS 1.2+ support (TLS 1.0/1.1 deprecated as of January 2026)

## Configuration

### Application Properties

All configuration is managed through `src/main/resources/application.properties`:

| Property | Description | Default | Required |
|----------|-------------|---------|----------|
| `aws.region` | AWS region (e.g., us-east-1) | us-east-1 | Yes |
| `aws.access.key` | AWS Access Key ID | - | Yes |
| `aws.secret.key` | AWS Secret Access Key | - | Yes |
| `aws.s3.output.location` | S3 bucket for query results | - | Yes |
| `aws.athena.workgroup` | Athena workgroup name | (empty) | **Recommended** for production |
| `db.pool.maxSize` | Maximum connection pool size | 5 | No |
| `db.pool.connectionTimeout` | Connection wait timeout (ms) | 5000 | No |
| `db.query.timeout` | Query timeout in milliseconds | 300000 (5 min) | No |
| `db.ssl.enabled` | Enable SSL for JDBC connections | false | No |
| `db.ssl.trustStorePath` | Custom SSL truststore path | (empty) | No |
| `http.port` | HTTP listener port | 8081 | No |
| `api.base.path` | API base path | /api/athena | No |
| `athena.table.transactional` | Iceberg table name | iceberg_orders | No |
| `athena.table.analytical` | Parquet table name | parquet_sales_log | No |
| `pagination.idColumn` | Column name for cursor pagination | id | No |

### Environment Variables

You can override properties using environment variables:

```bash
export AWS_REGION=us-west-2
export AWS_ACCESS_KEY=AKIA...
export AWS_SECRET_KEY=...
export AWS_S3_OUTPUT_LOCATION=s3://my-bucket/results/
```

### Configuration Examples

#### Example 1: Query Timeout Configuration

**Scenario**: Queries typically take 2-3 minutes, but some may take up to 10 minutes.

```properties
# Set query timeout to 10 minutes (600000 ms)
db.query.timeout=600000
```

**For long-running queries** (> 10 minutes), use the async pattern instead:
```bash
# Use async endpoint for queries that may take > 10 minutes
curl -X POST http://localhost:8081/api/athena/query/async \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT * FROM very_large_table"}'
```

#### Example 2: Connection Pool Tuning

**Scenario**: High concurrency with queries averaging 45 seconds.

```properties
# Increase pool size for more concurrent queries
db.pool.maxSize=10

# Increase connection timeout to handle query completion time
# Formula: (Average Query Time / 2) + Buffer = (45000 / 2) + 5000 = 27500ms
db.pool.connectionTimeout=30000
```

**Calculation**:
- Average query time: 45 seconds (45000ms)
- Recommended timeout: 30000ms (30 seconds) - allows time for queries to complete
- Pool size: 10 connections (if Athena quota allows)

#### Example 3: CloudHub 2.0 Configuration

**Using Environment Variables** (Recommended):

```bash
# Set in CloudHub 2.0 environment variables
AWS_REGION=us-east-1
AWS_ACCESS_KEY=AKIA...
AWS_SECRET_KEY=...
AWS_S3_OUTPUT_LOCATION=s3://my-athena-results/results/
AWS_ATHENA_WORKGROUP=mulesoft-production
DB_SSL_ENABLED=true
DB_POOL_CONNECTION_TIMEOUT=10000
DB_QUERY_TIMEOUT=300000
```

**Using application.properties** (Override in deployment):

```properties
# CloudHub 2.0 specific settings
db.ssl.enabled=true
db.pool.connectionTimeout=10000
db.query.timeout=300000
aws.athena.workgroup=mulesoft-production
```

#### Example 4: Secrets Manager Integration

**Note**: MuleSoft doesn't have built-in Secrets Manager integration. Use one of these approaches:

**Option A: External Secrets Manager** (Recommended for production):
1. Use Anypoint Secrets Manager or HashiCorp Vault
2. Retrieve secrets at application startup
3. Set as environment variables or system properties

**Option B: AWS Secrets Manager via HTTP Connector**:
```xml
<!-- Retrieve secret at startup -->
<http:request method="POST" 
             url="https://secretsmanager.us-east-1.amazonaws.com/"
             config-ref="AWS_Secrets_Config">
    <!-- AWS Signature V4 authentication required -->
</http:request>
```

**Option C: Environment Variables** (Simplest):
- Set secrets as environment variables in CloudHub 2.0
- Properties file uses `${AWS_ACCESS_KEY:default}` syntax
- Secrets are injected at runtime

#### Example 5: Production Configuration

**Complete production setup**:

```properties
# AWS Configuration
aws.region=us-east-1
aws.access.key=${AWS_ACCESS_KEY}
aws.secret.key=${AWS_SECRET_KEY}
aws.s3.output.location=${AWS_S3_OUTPUT_LOCATION}
aws.athena.workgroup=${AWS_ATHENA_WORKGROUP:mulesoft-production}

# Connection Pool (tuned for production)
db.pool.initialSize=2
db.pool.minSize=2
db.pool.maxSize=8
db.pool.idleTimeout=60000
db.pool.maxWait=15000
db.pool.connectionTimeout=15000

# Query Configuration
db.query.timeout=600000

# SSL/TLS (required for CloudHub 2.0)
db.ssl.enabled=${DB_SSL_ENABLED:true}

# Pagination
pagination.defaultLimit=10000
pagination.maxLimit=50000
pagination.idColumn=id
```

## API Endpoints

### 1. Synchronous Query Execution

Execute a SQL query and receive results immediately.

**⚠️ Important**: This endpoint is suitable for small to medium datasets. For large result sets, use the CTAS pattern endpoint.

**Endpoint**: `POST /api/athena/query`

**Request Body**:
```json
{
    "query": "SELECT id, name, amount FROM sales_data WHERE year=2026 AND month=02 AND amount > 1000",
    "lastSeenId": 12345,
    "limit": 10000
}
```

**Query Parameters**:
- `query` (required): SQL query to execute
- `lastSeenId` (optional): For cursor-based pagination - returns rows where id > lastSeenId
- `limit` (optional): Maximum number of rows to return (default: 10000)

**Response**:
```json
{
    "status": "SUCCESS",
    "query": "SELECT id, name, amount FROM sales_data WHERE year=2026 AND month=02 AND amount > 1000 AND id > 12345 ORDER BY id LIMIT 10000",
    "rowCount": 10000,
    "results": [
        {
            "id": 12346,
            "name": "Product A",
            "amount": 1500.00
        }
    ],
    "lastSeenId": 22345,
    "hasMore": true
}
```

**Cursor-Based Pagination**:
- When `lastSeenId` is provided, the query automatically adds `WHERE id > :lastSeenId` (or `AND id > :lastSeenId` if WHERE already exists)
- Results are automatically ordered by the ID column
- Use `lastSeenId` from the response for the next page request
- This avoids OFFSET costs - Athena doesn't scan and discard rows before the offset point

### 2. Asynchronous Query Execution

Start a query execution and receive an execution ID immediately.

**Endpoint**: `POST /api/athena/query/async`

**Request Body**:
```json
{
    "query": "SELECT * FROM large_table WHERE year=2026"
}
```

**Response**:
```json
{
    "status": "QUEUED",
    "executionId": "abc123-def456-ghi789",
    "query": "SELECT * FROM large_table WHERE year=2026",
    "statusEndpoint": "/api/athena/query/abc123-def456-ghi789/status",
    "resultsEndpoint": "/api/athena/query/abc123-def456-ghi789/results"
}
```

### 3. Query Status

Check the status of an asynchronous query execution.

**Endpoint**: `GET /api/athena/query/{executionId}/status`

**Response**:
```json
{
    "executionId": "abc123-def456-ghi789",
    "status": "SUCCEEDED",
    "query": "SELECT * FROM large_table WHERE year=2026",
    "submissionDateTime": "2026-01-15T10:00:00Z",
    "completionDateTime": "2026-01-15T10:02:30Z",
    "dataScannedInBytes": 1073741824,
    "executionTimeInMillis": 150000
}
```

**Status Values**: `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED`

### 4. Query Results

Retrieve results from a completed query execution.

**Endpoint**: `GET /api/athena/query/{executionId}/results?limit=10000`

**Query Parameters**:
- `limit` (optional): Maximum number of rows to return (default: 10000)

**Response**:
```json
{
    "executionId": "abc123-def456-ghi789",
    "resultSet": {
        "columnInfo": [
            {
                "Name": "id",
                "Type": "bigint"
            }
        ],
        "rows": [
            {
                "rowNumber": 0,
                "data": [
                    {
                        "VarCharValue": "1"
                    }
                ]
            }
        ]
    },
    "nextToken": "token123"
}
```

### 5. CTAS Pattern (Large Datasets)

Execute a CREATE TABLE AS SELECT query to export large result sets to S3 in Parquet format.

**Endpoint**: `POST /api/athena/query/ctas`

**Request Body**:
```json
{
    "query": "SELECT id, name, amount FROM sales_data WHERE year=2026",
    "outputTable": "sales_2026_export"
}
```

**Response**:
```json
{
    "status": "SUCCESS",
    "tableName": "sales_2026_export",
    "s3Location": "s3://your-bucket/results/sales_2026_export/",
    "message": "CTAS query completed. Results are available in S3.",
    "downloadEndpoint": "/api/athena/query/ctas/sales_2026_export/download"
}
```

### 6. Transactional Operations (Iceberg)

Execute ACID transactions on Iceberg tables.

**Endpoint**: `POST /api/athena/transaction`

**Request Body**:
```json
{
    "tableName": "iceberg_orders",
    "operations": [
        "INSERT INTO iceberg_orders VALUES (1, 'Order A', 100.00)",
        "UPDATE iceberg_orders SET amount = 150.00 WHERE id = 1"
    ]
}
```

**Response**:
```json
{
    "status": "SUCCESS",
    "message": "Transaction committed successfully",
    "tableName": "iceberg_orders",
    "operationsCount": 2
}
```

**Important**: 
- Only works with Apache Iceberg tables
- Standard Athena tables (CSV/JSON/Parquet without Iceberg) do NOT support transaction syntax
- Uses `START TRANSACTION` and `COMMIT` syntax
- **Automatic ROLLBACK**: If a transaction fails, the API automatically executes `ROLLBACK` to clean up pending/locked Iceberg snapshots, preventing snapshot lock issues

### 7. Analytical Queries (Parquet)

Execute optimized analytical queries with partition filtering.

**Endpoint**: `POST /api/athena/analytics`

**Request Body**:
```json
{
    "tableName": "parquet_sales_log",
    "columns": "id, name, amount, date",
    "partitionFilter": "year=2026 AND month=02",
    "whereClause": "amount > 1000",
    "lastSeenId": 12345,
    "limit": 10000
}
```

**Query Parameters**:
- `tableName` (optional): Table name (defaults to configured analytical table)
- `columns` (optional): Comma-separated column list (default: "*")
- `partitionFilter` (recommended): Partition filter to reduce data scanned
- `whereClause` (optional): Additional WHERE conditions
- `lastSeenId` (optional): For cursor-based pagination - returns rows where id > lastSeenId
- `limit` (optional): Maximum number of rows (default: 10000)

**Response**:
```json
{
    "status": "SUCCESS",
    "query": "SELECT id, name, amount, date FROM parquet_sales_log WHERE year=2026 AND month=02 AND amount > 1000 AND id > 12345 ORDER BY id LIMIT 10000",
    "tableName": "parquet_sales_log",
    "rowCount": 5000,
    "results": [...],
    "lastSeenId": 22345,
    "hasMore": false,
    "optimization": {
        "partitionFilterUsed": true,
        "columnsSelected": true,
        "cursorPaginationUsed": true
    }
}
```

**Cursor-Based Pagination**:
- When `lastSeenId` is provided, the query automatically adds `AND id > :lastSeenId` to the WHERE clause
- Results are automatically ordered by the ID column (configured via `pagination.idColumn` property)
- Use `lastSeenId` from the response for the next page request
- Avoids OFFSET costs - Athena doesn't scan and discard rows before the offset point

## Result Set Size Limits

### JDBC Driver Limitations

**⚠️ Critical**: The Athena JDBC driver has a **5GB result set limit** when using synchronous queries.

- **Synchronous Query Endpoint** (`POST /api/athena/query`): 
  - Maximum result set size: **~5GB** (loaded into Mule worker heap)
  - Exceeding this limit will cause `OutOfMemoryError`
  - **Recommendation**: Use for datasets < 1GB to ensure safety margin

- **Asynchronous Query Endpoint** (`POST /api/athena/query/async`):
  - No JDBC driver limit (results stored in S3)
  - Suitable for any result set size
  - Use for datasets > 1GB or when result size is unknown

### When to Use CTAS Pattern

Use the **CTAS Pattern** (`POST /api/athena/query/ctas`) for:
- Result sets > 100MB
- Result sets approaching 1GB (safety margin)
- When you need to process results in batches
- When results will be accessed multiple times

**Benefits**:
- Results stored in S3 as Parquet (columnar format, compressed)
- No memory limitations
- Can download files directly or use presigned URLs
- Efficient for large datasets

### Monitoring Result Sizes

Monitor query result sizes to prevent issues:

1. **Check Query Statistics**: Use the status endpoint to get `dataScannedInBytes`
2. **Estimate Result Size**: Typically 10-50% of data scanned (depends on columns selected)
3. **Set Alarms**: Configure CloudWatch alarms for queries scanning >10GB
4. **Use Workgroups**: Set data scanned limits per query (e.g., 10GB max)

**Example**: If a query scans 50GB but only selects 3 columns, result size might be ~5-10GB. Use CTAS pattern.

## Usage Examples

### Example 1: Simple Synchronous Query

```bash
curl -X POST http://localhost:8081/api/athena/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT COUNT(*) as total FROM sales_data WHERE year=2026"
  }'
```

### Example 2: Asynchronous Query with Polling

```bash
# Start query
EXECUTION_ID=$(curl -X POST http://localhost:8081/api/athena/query/async \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT * FROM large_table"}' \
  | jq -r '.executionId')

# Poll for status
while true; do
  STATUS=$(curl -s http://localhost:8081/api/athena/query/$EXECUTION_ID/status | jq -r '.status')
  echo "Status: $STATUS"
  if [ "$STATUS" = "SUCCEEDED" ] || [ "$STATUS" = "FAILED" ]; then
    break
  fi
  sleep 5
done

# Get results
curl http://localhost:8081/api/athena/query/$EXECUTION_ID/results
```

### Example 3: CTAS for Large Dataset

```bash
curl -X POST http://localhost:8081/api/athena/query/ctas \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT * FROM very_large_table WHERE year=2026",
    "outputTable": "export_2026"
  }'
```

### Example 4: Cursor-Based Pagination

```bash
# First page (no lastSeenId)
RESPONSE=$(curl -X POST http://localhost:8081/api/athena/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT id, name, amount FROM sales_data WHERE year=2026",
    "limit": 10000
  }')

# Extract lastSeenId from response
LAST_ID=$(echo $RESPONSE | jq -r '.lastSeenId')

# Next page using cursor
curl -X POST http://localhost:8081/api/athena/query \
  -H "Content-Type: application/json" \
  -d "{
    \"query\": \"SELECT id, name, amount FROM sales_data WHERE year=2026\",
    \"lastSeenId\": $LAST_ID,
    \"limit\": 10000
  }"
```

## Best Practices

### Cost Optimization

1. **Always Use Partition Filters**
   - ❌ BAD: `SELECT * FROM sales_data WHERE amount > 1000`
   - ✅ GOOD: `SELECT id, amount FROM sales_data WHERE year=2026 AND month=02 AND amount > 1000`
   - Impact: Scanning 2GB vs. 100GB = 50x cost reduction

2. **Use Specific Column Selection**
   - Avoid `SELECT *` in production
   - Only select columns you need

3. **Prefer Parquet/ORC over CSV/JSON**
   - Columnar formats reduce data scanned by 50-90%
   - Lower storage and query costs

4. **Use CTAS for Large Result Sets**
   - For datasets >100MB, use CTAS pattern
   - Export to S3 in Parquet format
   - Download and process via S3 Connector

### Performance Optimization

1. **Connection Pooling**
   - Default: max 5 connections per worker
   - Adjust based on your Athena quota (default: 25 concurrent queries)
   - Formula: `maxPoolSize = (Athena Quota / Number of Workers) - Buffer`
   
   **Connection Wait Strategy** (`db.pool.connectionTimeout`):
   - **Default**: 5000ms (5 seconds)
   - **Purpose**: When all connections in the pool are busy, new requests wait for an available connection instead of failing immediately
   - **Behavior**: 
     - If a connection becomes available within the timeout period, the request proceeds
     - If timeout expires, the request fails with "Could not obtain connection" error
   - **Tuning Guidance**:
     - **Short queries** (< 30 seconds): 5000ms is usually sufficient
     - **Medium queries** (30 seconds - 2 minutes): Increase to 10000-15000ms
     - **Long queries** (> 2 minutes): Consider using async pattern instead
     - **High concurrency**: Increase timeout to 10000-20000ms to handle bursts
   - **Formula**: `connectionTimeout = (Average Query Time / 2) + Buffer`
     - Example: Average query = 60s, use 30000ms (30s) + 5000ms buffer = 35000ms
   - **Trade-off**: Longer timeout = better request success rate but slower failure detection
   - **Monitoring**: Track "Could not obtain connection" errors - if frequent, increase timeout or pool size

2. **Query Timeouts**
   - Set appropriate timeouts based on SLA
   - Default: 5 minutes (300000 ms)
   - Long-running queries: Use async pattern

3. **Pagination**
   - **Use cursor-based pagination** for large datasets (built into the API)
   - ❌ BAD: `SELECT * FROM table LIMIT 10000 OFFSET 1000000` (scans and discards 1M rows)
   - ✅ GOOD: Use `lastSeenId` parameter - `WHERE id > :lastSeenId ORDER BY id LIMIT 10000`
   - The API automatically handles cursor-based pagination when `lastSeenId` is provided
   - Example: First request without `lastSeenId`, then use `lastSeenId` from response for next page

### Concurrent Query Management

- **Throttling**: Implement retry with exponential backoff
- **Queuing**: Use SQS for batch workloads
- **Monitoring**: Track concurrent query usage

### Data Format Selection

| Feature | Parquet (Standard) | Iceberg |
|---------|-------------------|---------|
| Use Case | Append-only / Read-heavy Analytics | Transactional / Concurrent Writes |
| ACID Support | No | Yes |
| Performance | Fastest | Fast (5-10% overhead) |
| Schema Evolution | Hard | Easy |
| Storage Cost | Lowest | Low (+1-2% metadata) |

**Recommendation**: Use Parquet for analytical workloads, Iceberg for transactional workloads requiring ACID guarantees.

## Error Handling

The API includes comprehensive error handling for common scenarios:

### Query Execution Errors

- **Timeout**: Query exceeds configured timeout
- **Syntax Error**: Invalid SQL syntax
- **Resource Error**: Insufficient resources

**Response**: HTTP 500 with error details

### Connectivity Errors

- **Network Issues**: Unable to reach AWS Athena
- **Authentication**: Invalid AWS credentials
- **Configuration**: Incorrect JDBC URL or driver

**Response**: HTTP 503 with connectivity error message

### Throttling Errors

- **TooManyRequestsException**: Rate limit exceeded
- **Automatic Retry**: Exponential backoff (3 retries, 5s intervals)

**Response**: Automatic retry, then HTTP 500 if all retries fail

### Iceberg Metadata Errors

- **ICEBERG_MISSING_METADATA**: Metadata corruption or missing
- **Action Required**: Contact data engineering team

**Response**: HTTP 500 with operational alert message

## Security Considerations

### Credential Management

1. **Development**: Use `application.properties` (not recommended for production)
2. **Production**: Use Anypoint Secrets Manager or Vault
3. **CloudHub 2.0**: Use environment variables or IAM roles

### Network Security

1. **SSL/TLS Configuration**: 
   - TLS 1.2+ required (TLS 1.0/1.1 deprecated as of January 2026)
   - **Enable SSL for JDBC connections**: Set `db.ssl.enabled=true` in `application.properties`
   - **CloudHub 2.0**: SSL is especially critical - always enable for production deployments
   - **RTF**: SSL works for both CloudHub 2.0 and RTF deployments
   - **Custom Truststore**: Optionally configure `db.ssl.trustStorePath` if using a custom truststore
   - The API automatically enforces SSL=1 in connection properties when enabled

2. **CloudHub 2.0 Static IPs**:
   - Whitelist Private Space static IPs in AWS IAM/VPC policies
   - Or use IAM role-based authentication via Transit Gateway

3. **VPC Configuration**:
   - For RTF on AWS, configure VPC endpoints for Athena and S3
   - Use private subnets for enhanced security

### IAM Best Practices

1. **Least Privilege**: Grant minimum required permissions
2. **Separate Credentials**: Use different IAM users/roles per environment
3. **Rotate Keys**: Regularly rotate access keys
4. **Monitor Access**: Enable CloudTrail for audit logging

## Monitoring & Observability

### CloudWatch Alarms

Set up CloudWatch alarms to monitor Athena query performance and costs:

#### 1. Query Execution Time Alarm

**Purpose**: Alert when queries exceed expected execution time

```json
{
  "AlarmName": "Athena-Query-Execution-Time-High",
  "MetricName": "QueryExecutionTime",
  "Namespace": "AWS/Athena",
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 1,
  "Threshold": 300000,
  "ComparisonOperator": "GreaterThanThreshold",
  "AlarmActions": ["arn:aws:sns:us-east-1:ACCOUNT:athena-alerts"]
}
```

**Recommended Thresholds**:
- Warning: > 5 minutes (300000 ms)
- Critical: > 10 minutes (600000 ms)

#### 2. Failed Queries Alarm

**Purpose**: Alert on query failures

```json
{
  "AlarmName": "Athena-Query-Failures",
  "MetricName": "QueryFailures",
  "Namespace": "AWS/Athena",
  "Statistic": "Sum",
  "Period": 300,
  "EvaluationPeriods": 1,
  "Threshold": 5,
  "ComparisonOperator": "GreaterThanThreshold"
}
```

#### 3. Throttling Exception Alarm

**Purpose**: Alert when hitting Athena concurrent query limits

```json
{
  "AlarmName": "Athena-Throttling-Exceptions",
  "MetricName": "ThrottlingException",
  "Namespace": "AWS/Athena",
  "Statistic": "Sum",
  "Period": 60,
  "EvaluationPeriods": 1,
  "Threshold": 1,
  "ComparisonOperator": "GreaterThanOrEqualToThreshold"
}
```

#### 4. Data Scanned Alarm

**Purpose**: Alert on queries scanning excessive data (cost control)

```json
{
  "AlarmName": "Athena-Data-Scanned-High",
  "MetricName": "DataScannedInBytes",
  "Namespace": "AWS/Athena",
  "Statistic": "Sum",
  "Period": 300,
  "EvaluationPeriods": 1,
  "Threshold": 10737418240,
  "ComparisonOperator": "GreaterThanThreshold"
}
```

**Threshold**: 10GB (10737418240 bytes) - adjust based on your cost tolerance

### CloudWatch Dashboard

Create a dashboard to visualize Athena metrics:

**Key Metrics to Monitor**:
- Query execution time (p50, p95, p99)
- Failed query count
- Data scanned per query
- Throttling exceptions
- Query cost (estimated from data scanned)

**Dashboard JSON Template**:

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Athena", "QueryExecutionTime", {"stat": "Average"}],
          [".", ".", {"stat": "p95"}],
          [".", ".", {"stat": "p99"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Query Execution Time"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Athena", "DataScannedInBytes", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Data Scanned (Bytes)"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Athena", "QueryFailures", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Failed Queries"
      }
    }
  ]
}
```

### Query Cost Tracking

**Estimate Query Costs**:
- Athena charges $5 per TB of data scanned
- Formula: `Cost = (DataScannedInBytes / 1099511627776) * 5`
- Example: 100GB scanned = $0.50 per query

**Cost Optimization**:
1. Use partition filters (reduce data scanned by 50-90%)
2. Select specific columns (not `SELECT *`)
3. Use Parquet/ORC format (columnar, compressed)
4. Set workgroup data scanned limits
5. Monitor and alert on high-cost queries

**Cost Allocation Tags**:
- Tag workgroups with cost allocation tags
- Track costs by application, environment, team
- Use AWS Cost Explorer to analyze Athena spending

### Application-Level Monitoring

Monitor Mule application metrics:

1. **Connection Pool Usage**: Track active/idle connections
2. **Query Timeout Rate**: Monitor queries exceeding timeout
3. **Error Rates**: Track DB:QUERY_EXECUTION and DB:CONNECTIVITY errors
4. **Response Times**: Monitor API endpoint latency

**Recommended Log Levels**:
- Production: INFO (log query execution, errors)
- Debugging: DEBUG (log SQL queries, connection pool status)
- Development: TRACE (detailed flow execution)

## Troubleshooting

### Common Issues

#### 1. "Driver not found" Error

**Problem**: Athena JDBC driver not installed in local Maven repository.

**Solution**: 
```bash
mvn install:install-file \
  -Dfile=AthenaJDBC42-2.0.38.jar \
  -DgroupId=com.simba.athena \
  -DartifactId=athena-jdbc \
  -Dversion=2.0.38 \
  -Dpackaging=jar
```

#### 2. "Connection timeout" Error

**Problem**: Unable to connect to AWS Athena.

**Solutions**:
- Verify AWS credentials in `application.properties`
- Check network connectivity to AWS
- Verify IAM permissions
- Ensure TLS 1.2+ support

#### 3. "Query timeout" Error

**Problem**: Query execution exceeds timeout threshold.

**Solutions**:
- Increase `db.query.timeout` in `application.properties`
- Use async query pattern for long-running queries
- Optimize query (add partition filters, limit columns)

#### 4. "ThrottlingException" Error

**Problem**: Exceeded Athena concurrent query limit.

**Solutions**:
- Reduce connection pool size
- Implement request queuing
- Use async pattern to free threads
- Contact AWS to increase quota

#### 5. "ICEBERG_MISSING_METADATA" Error

**Problem**: Iceberg table metadata corrupted or missing.

**Solutions**:
- Verify S3 metadata location is accessible
- Check for concurrent writes causing metadata conflicts
- Contact data engineering team for metadata repair

#### 6. "Transaction not available" Error

**Problem**: Attempting to use transactions on non-Iceberg tables.

**Solution**: Only use transactional flows with Iceberg tables. Use analytical flows for Parquet/CSV/JSON tables.

#### 7. "Iceberg Snapshot Lock" or "Pending Transaction" Error

**Problem**: Iceberg transaction failed but snapshot remains in locked/pending state.

**Solution**: The API automatically executes `ROLLBACK` in the error handler to clean up failed transactions. If you still see this error, check:
- Network connectivity during transaction execution
- Concurrent transaction conflicts
- S3 metadata location accessibility

### Debugging Tips

1. **Enable Debug Logging**: Set log level to DEBUG in `log4j2.xml`
2. **Check Query Logs**: Review Athena query history in AWS Console
3. **Monitor S3**: Verify query results are being written to S3
4. **Test Connectivity**: Use AWS CLI to test credentials:
   ```bash
   aws athena list-work-groups --region us-east-1
   ```

## Additional Resources

- [AWS Athena Documentation](https://docs.aws.amazon.com/athena/)
- [Athena JDBC Driver Guide](https://docs.aws.amazon.com/athena/latest/ug/connect-with-jdbc.html)
- [MuleSoft Database Connector](https://docs.mulesoft.com/db-connector/)
- [Apache Iceberg Documentation](https://iceberg.apache.org/)

## Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review AWS Athena CloudWatch logs
3. Contact your MuleSoft administrator or AWS support

## License

This project is licensed under the MIT License.
