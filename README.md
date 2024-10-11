# marketData
NFO Data Analysis


All the data is available here

https://drive.google.com/drive/folders/1oFJro3-siJNnnihCGXPIl6ZkmhGBwSDM?usp=drive_link

https://drive.google.com/drive/u/0/folders/1QnH9XNjbDGSw-eDLZUgx-fT0bg2f3LgN

Please raise the request and I will  provide the access




Study Resources
https://www.youtube.com/watch?v=pPqazMTzNOM&t=1159s


Sample Data

Ticker	SYMBOL	EXPIRY	STRIKE	TYPE	SEGMENT	Date	Time	Open	High	Low	Close	Volume	Open Interest
AARTIIND25JAN24500PE.NFO	AARTIIND	25-Jan-24	500	PE	NFO	01-01-2024	10:31:59	0.5	0.5	0.5	0.5	2000	59000
HCLTECH25JAN241200PE.NFO	HCLTECH	25-Jan-24	1200	PE	NFO	01-01-2024	11:02:59	0.6	0.6	0.6	0.6	700	29400
BANKNIFTY03JAN2437500PE.NFO	BANKNIFTY	03-Jan-24	37500	PE	NFO	01-01-2024	09:32:59	1.1	1.15	1.1	1.1	225	99990
![image](https://github.com/user-attachments/assets/944c2b1a-024b-4feb-98b7-4f7681cf17ab)





To optimize this type of data in PostgreSQL, particularly given that it's financial data with time series and categorical fields, you can focus on **indexing**, **partitioning**, and **compression** strategies. Here’s a step-by-step guide for the best optimization practices:

### 1. **Table Design**
   We will start by designing the table with appropriate data types based on the columns in your dataset:

```sql
CREATE TABLE financial_options_data (
    ticker VARCHAR(255),
    symbol VARCHAR(100),
    expiry DATE,
    strike NUMERIC(10, 2),
    opt_type VARCHAR(3),
    segment VARCHAR(10),
    trade_date DATE,
    trade_time TIME,
    open NUMERIC(12, 4),
    high NUMERIC(12, 4),
    low NUMERIC(12, 4),
    close NUMERIC(12, 4),
    volume BIGINT,
    open_interest BIGINT
);
```
- `VARCHAR` is used for text fields like `ticker`, `symbol`, and `segment`.
- `NUMERIC(10, 2)` for strike price and financial values that need precision.
- `DATE` for dates (`expiry`, `trade_date`).
- `TIME` for `trade_time`.
- `BIGINT` for `volume` and `open_interest` (for large integer values).

### 2. **Indexes for Faster Querying**
   Given the type of data, some columns are likely to be queried frequently, such as `symbol`, `expiry`, `strike`, `trade_date`, and `opt_type`. Creating indexes on these columns will greatly improve the performance of select queries.

```sql
CREATE INDEX idx_symbol ON financial_options_data (symbol);
CREATE INDEX idx_expiry ON financial_options_data (expiry);
CREATE INDEX idx_strike ON financial_options_data (strike);
CREATE INDEX idx_trade_date_symbol ON financial_options_data (trade_date, symbol);
CREATE INDEX idx_opttype ON financial_options_data (opt_type);
```

- **Multi-column indexes**: For queries that involve both `trade_date` and `symbol`, a multi-column index (`trade_date, symbol`) will boost performance.

### 3. **Partitioning by Date**
   Since you're dealing with a potentially massive time-series dataset, partitioning the table by date (or expiry date) will help manage the large data size and improve performance for querying specific time periods.

#### Partitioning Example:

```sql
CREATE TABLE financial_options_data_partitioned (
    ticker VARCHAR(255),
    symbol VARCHAR(100),
    expiry DATE,
    strike NUMERIC(10, 2),
    opt_type VARCHAR(3),
    segment VARCHAR(10),
    trade_date DATE,
    trade_time TIME,
    open NUMERIC(12, 4),
    high NUMERIC(12, 4),
    low NUMERIC(12, 4),
    close NUMERIC(12, 4),
    volume BIGINT,
    open_interest BIGINT
) PARTITION BY RANGE (trade_date);

-- Create partitions by year
CREATE TABLE financial_options_data_2024 PARTITION OF financial_options_data_partitioned
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

- **Partitioning by year** or even by month is ideal if queries are frequently constrained by date ranges (e.g., filtering on `trade_date`).
- This reduces the amount of data PostgreSQL needs to scan when executing queries.

### 4. **Compression (Optional with PostgreSQL Extensions)**
   If you're using **PostgreSQL 13+** or are open to extensions like **TimescaleDB**, you can use native compression features to optimize storage.

   Example with **TimescaleDB**:
   - Convert your table into a **hypertable**, then enable compression.

   ```sql
   SELECT create_hypertable('financial_options_data', 'trade_date');

   -- Enable compression
   ALTER TABLE financial_options_data SET (timescaledb.compress, true);

   -- Compress historical data (e.g., older than 30 days)
   SELECT compress_chunk(i.chunk_name)
   FROM show_chunks('financial_options_data') i
   WHERE i.range_end < now() - interval '30 days';
   ```

### 5. **Batch Insertion**
   If you're loading large volumes of data (e.g., millions of rows), use `COPY` instead of multiple `INSERT` statements:

```sql
COPY financial_options_data (ticker, symbol, expiry, strike, opt_type, segment, trade_date, trade_time, open, high, low, close, volume, open_interest)
FROM '/path/to/your/data.csv' DELIMITER ',' CSV HEADER;
```

This is much faster for bulk data loading.

### 6. **Query Optimization**
   When querying this dataset, make sure to:
   - Use indexes in your `WHERE` clauses (e.g., filtering on `symbol`, `expiry`, or `trade_date`).
   - Avoid selecting all columns unless absolutely necessary, e.g., `SELECT *`.
   - Utilize partitioning to target specific date ranges, reducing the scan size.

### 7. **Vacuum and Analyze**
   After large data loads, ensure to run `VACUUM` and `ANALYZE` to update table statistics and reclaim storage space:

```sql
VACUUM ANALYZE financial_options_data;
```

### Example Query for Optimized Table:
```sql
SELECT symbol, strike, expiry, open, close, volume
FROM financial_options_data
WHERE symbol = 'AARTIIND' AND trade_date = '2024-01-01';
```

This query will benefit from indexes and partitions, ensuring quick execution for large datasets.

Let me know if you need further optimizations or specific query tuning! 💡
