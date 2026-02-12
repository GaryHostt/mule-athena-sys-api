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

### 5. (Optional) Create Athena Workgroup

Workgroups provide governance and cost control:

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
2. Download the latest JDBC driver (recommended: v2.x series for Mule 4.10+)
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

# Athena Workgroup (optional)
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
   - `AWS_ATHENA_WORKGROUP` (optional)

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
| `aws.athena.workgroup` | Athena workgroup name | (empty) | No |
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

## API Endpoints

### 1. Synchronous Query Execution

Execute a SQL query and receive results immediately (for small datasets <5GB).

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
   - **Connection Wait Strategy**: Configure `db.pool.connectionTimeout` (default: 5000ms) to wait for available connections during high concurrency bursts, preventing immediate "Could not obtain connection" errors

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
