# Openflow Watermarking Skill - Examples

## Example 1: Timestamp-Based Watermark

**User Request:**
> I need to set up incremental ingestion from a PostgreSQL orders table to Snowflake. The table has an `updated_at` timestamp column.

**Skill Response:**
This is a timestamp-based watermark scenario. I'll create a flow using `QueryDatabaseTable`.

**Questions to gather:**
1. PostgreSQL connection details (host, port, database)
2. Table: `public.orders`
3. Watermark column: `updated_at` (TIMESTAMP)
4. Snowflake destination details

**Generated Flow:** Uses `timestamp-watermark.json` template

---

## Example 2: ID-Based Watermark (Append-Only)

**User Request:**
> We have an event log table in MySQL that only receives inserts. Each row has an auto-increment `event_id`. I want to stream new events to Snowflake.

**Skill Response:**
This is an ID-based watermark for an append-only table. I'll use `QueryDatabaseTable` with the `event_id` as the maximum-value column.

**Questions to gather:**
1. MySQL connection details
2. Table: `events.event_log`
3. Watermark column: `event_id` (BIGINT AUTO_INCREMENT)
4. Initial load strategy: Start at current max or fetch all existing?

**Generated Flow:** Uses `id-watermark.json` template

---

## Example 3: Composite Watermark (Partitioned Table)

**User Request:**
> Our sales data is partitioned by `sale_date` and has a `transaction_seq` within each day. I need to handle late-arriving data where older partitions might get new records.

**Skill Response:**
This requires a composite watermark with hierarchical columns. I'll use `GenerateTableFetch` with multiple maximum-value columns.

**Configuration:**
- Maximum-value Columns: `sale_date, transaction_seq`
- The processor tracks the max `sale_date`, and within that date, the max `transaction_seq`
- Partition size: 10000 rows for parallel processing

**Generated Flow:** Uses `composite-watermark.json` template

---

## Example 4: Full Table Snapshot

**User Request:**
> I have a dimension table without any timestamp or sequence column. It gets full updates nightly and I need to sync it to Snowflake.

**Skill Response:**
Without a reliable watermark column, we'll use `FetchTableSnapshot` which fetches the entire table using primary key-based batching.

**Questions to gather:**
1. Primary key column(s) for batching
2. Batch size (rows per fetch)
3. Schedule frequency

**Generated Flow:** Uses `full-snapshot.json` template

---

## Example 5: Mixed Strategy Discussion

**User Request:**
> I'm not sure which approach to use. My table has both a `created_at` and `updated_at` column, plus an auto-increment ID.

**Skill Response:**
Let me help you decide:

1. **If rows are updated in place** → Use `updated_at` timestamp watermark
   - Captures both new inserts and updates
   - Best for SCD Type 1 scenarios

2. **If rows are insert-only** → Use auto-increment ID watermark
   - More efficient (no index needed on timestamp)
   - Best for event/log tables

3. **If you need both** → Use composite watermark with `updated_at` as primary
   - Handles updates AND provides ordering within same timestamp

**Recommendation:** For most OLTP tables with updates, timestamp-based watermarking on `updated_at` is the safest choice.

---

## Common Follow-up Questions

### Q: How do I handle schema changes?
A: The flow will fail on schema mismatch. Options:
1. Use `UpdateSnowflakeTable` processor before `PutSnowpipeStreaming`
2. Enable schema evolution in Snowflake target table
3. Use a staging table with VARIANT column for flexibility

### Q: What about initial backfill?
A: Set `initial-load-strategy` to "Start at Beginning" for the first run, then switch to "Start at Current Maximum Values" after backfill completes.

### Q: How do I handle deletes?
A: These processors don't capture deletes. For delete tracking:
1. Use soft deletes with a `deleted_at` column (included in watermark)
2. Switch to CDC processors (`CaptureChangePostgreSQL`, etc.)
3. Implement periodic full snapshot comparison

### Q: What batch size should I use?
A: Start with these defaults and tune:
- Small tables (<1M rows): 10,000 rows/batch
- Medium tables (1M-100M rows): 50,000 rows/batch
- Large tables (>100M rows): 100,000 rows/batch with partitioning
