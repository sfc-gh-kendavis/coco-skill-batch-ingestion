# Openflow Batch Ingestion Watermarking Skill

Use this skill to help users build Openflow flows for batch data ingestion with watermarking strategies. The destination is always Snowflake.

## When to Activate

Activate when users mention:
- Openflow batch ingestion
- Watermarking for data pipelines
- Incremental data loading to Snowflake
- CDC or change tracking for batch jobs
- Building Openflow flows for database replication

## Decision Process

Follow this interactive decision tree to guide the user:

### Step 1: Determine Watermark Strategy

Ask the user which watermarking approach fits their source data:

| Strategy | When to Use | Processor |
|----------|-------------|-----------|
| **Timestamp-based** | Table has a reliable `updated_at`, `modified_date`, or `created_at` column that updates on every change | `QueryDatabaseTable` |
| **ID-based** | Table has an auto-incrementing primary key and is append-only (no updates) | `QueryDatabaseTable` |
| **Composite** | Need hierarchical watermarking (e.g., partition date + sequence number) or parallel fetching of large tables | `GenerateTableFetch` |
| **Full Snapshot** | No reliable watermark column exists; need complete table sync with PK-based batching | `FetchTableSnapshot` |

**Questions to ask:**
1. Does your source table have a timestamp column that updates when rows are modified?
2. Is the table append-only with an auto-incrementing ID?
3. Do you need to track multiple columns for incremental fetching?
4. Do you need a full table snapshot (initial load or no watermark available)?

### Step 2: Gather Source System Information

Ask about the source database:

1. **Database Type**: PostgreSQL, MySQL, SQL Server, Oracle, or Generic JDBC
2. **Connection Details**:
   - Hostname/IP
   - Port (default varies by DB type)
   - Database name
   - Schema name (if applicable)
3. **Authentication**:
   - Username
   - Password (recommend using Openflow Parameter Context or secrets manager)
4. **JDBC Driver**: Location of driver JAR if not built-in

### Step 3: Gather Source Table Details

Ask about the specific table:

1. **Table Name**: Fully qualified name (schema.table)
2. **Watermark Column(s)**: Name and data type of column(s) used for tracking
3. **Primary Key Column(s)**: Required for FetchTableSnapshot
4. **Columns to Fetch**: Specific columns or all (`*`)
5. **WHERE Clause**: Any filtering conditions to apply
6. **Fetch Size**: Number of rows per batch (default: 1000)

### Step 4: Gather Snowflake Destination Details

1. **Snowflake Account**: Account identifier
2. **Database**: Target database name
3. **Schema**: Target schema name
4. **Table**: Target table name
5. **Role**: Snowflake role to use
6. **Warehouse**: Warehouse for Snowpipe Streaming
7. **Authentication**: Key-pair authentication details

### Step 5: Generate Flow Definition

Based on the gathered information, generate a JSON Flow Definition using the appropriate template:

- **Timestamp watermark** → Use `templates/timestamp-watermark.json`
- **ID watermark** → Use `templates/id-watermark.json`
- **Composite watermark** → Use `templates/composite-watermark.json`
- **Full snapshot** → Use `templates/full-snapshot.json`

## Processor Configuration Details

### QueryDatabaseTable (Timestamp/ID Watermark)

Key properties:
- `Database Connection Pooling Service`: Reference to DBCP controller
- `Table Name`: Source table name
- `Maximum-value Columns`: The watermark column(s)
- `Columns to Return`: Columns to fetch (empty = all)
- `db-fetch-where-clause`: Optional filter condition
- `initial-load-strategy`: How to handle first run
  - `Start at Beginning`: Fetch all existing rows first
  - `Start at Current Maximum Values`: Skip existing, only fetch new

### GenerateTableFetch (Composite Watermark)

Key properties:
- `Database Connection Pooling Service`: Reference to DBCP controller
- `Table Name`: Source table name
- `Maximum-value Columns`: Comma-separated list in hierarchical order
- `gen-table-fetch-partition-size`: Rows per partition (e.g., 10000)
- Requires downstream `ExecuteSQL` processor to run generated queries

### FetchTableSnapshot (Full Snapshot)

Key properties:
- `Connection Pool`: Reference to DBCP controller
- `Schema Name`: Source schema
- `Table Name`: Source table
- `Max Batch Size`: Rows per batch
- `Record Writer`: Output format writer
- Requires input FlowFile with JSON schema including `columns` and `primaryKeys`

### PutSnowpipeStreaming (Snowflake Destination)

Key properties:
- `Snowflake Account Identifier`: Account URL
- `Snowflake User`: Username
- `Private Key`: RSA private key for auth
- `Database`: Target database
- `Schema`: Target schema
- `Table`: Target table
- `Role`: Snowflake role
- `Record Reader`: Input format reader

## Output Requirements

Generate a complete JSON Flow Definition that includes:

1. **Controller Services**:
   - `DBCPConnectionPool` or `HikariCPConnectionPool` for source
   - `SnowflakeConnectionService` for destination
   - `JsonTreeReader` for record reading
   - `JsonRecordSetWriter` for record writing

2. **Processors**:
   - Source fetch processor (based on strategy)
   - `ConvertRecord` if format conversion needed
   - `PutSnowpipeStreaming` for Snowflake ingestion

3. **Connections**:
   - Wire processors with appropriate relationships
   - Handle `success`, `failure`, and `retry` paths

4. **Process Group**:
   - Wrap in a named Process Group
   - Include Parameter Context for sensitive values

Save the generated flow to a `.json` file in the current working directory.

## Error Handling Recommendations

Include in the generated flow:
- Retry logic for transient failures
- Dead letter queue (funnel to failure) for permanent errors
- Logging via `LogAttribute` processor for debugging
- Back-pressure settings to prevent memory issues

## Validation Checklist

Before finalizing, verify:
- [ ] Watermark column exists and has appropriate data type
- [ ] Source credentials are parameterized (not hardcoded)
- [ ] Snowflake target table exists or will be auto-created
- [ ] Primary keys specified for FetchTableSnapshot
- [ ] Batch sizes appropriate for data volume
