# PostgreSQL Optimize Query

#### Create sample data for test

      create table orders (order_no serial primary key, order_date date);
    
      create table items(item_no serial not null, order_no integer, product_name varchar, descr varchar, created_ts timestamp,
      constraint fk_items foreign key (order_no) references orders(order_no) 
      match simple on update cascade on delete cascade);
      
      with order_rws as ( insert into orders(order_no, order_date) select generate_series(1, 1000000) t, now()
      returning order_no)
      insert into items (item_no, order_no, product_name, descr, created_ts) select generate_series(1, 4) item_no,order_no, 'product',
      repeat('the description of the product',10), now() from order_rws;

      -- Check without indexing and get almost 200
      
      select * from orders ord join items itm on ord.order_no = itm.order_no where ord.order_no = 1000000
      
      -- Check with indexing
      
      create index idx_ord on items (order_no);
      
      -- Check with indexing and get almost 50
      
       select * from orders ord join items itm on ord.order_no = itm.order_no where ord.order_no = 1000000

       -- check the total data size of table
       \di+;
       \dt+;


#### PostgreSQL table grows to a significant size, and optimizing for performance becomes crucial. Here are best practices and strategies you can use to optimize large tables in PostgreSQL

## 1. Optimize Indexing Strategy
- Choose Selective Indexes
  - While it might be tempting to index every column used in queries, it's essential to be selective. Indexes should primarily be on columns that are frequently used in the WHERE, JOIN, GROUP BY, or ORDER BY clauses.
- Multicolumn Indexes
  - For queries filtering on multiple columns, a multicolumn index can be more effective. You can combine username, email, or any combination of columns based on frequent query patterns.
    -CREATE INDEX idx_users_multicol ON users (username, email);
- Partial Indexes:
  -  If certain values are queried more frequently (e.g., status = 'active'), consider using a partial index:
    - CREATE INDEX idx_users_active ON users (username) WHERE status = 'active';
- Include Indexes for Covering Queries:
  - As discussed earlier, using INCLUDE can speed up read operations by avoiding table lookups.
- Note: Be cautious with too many indexes, as they can slow down INSERT, UPDATE, and DELETE operations.

## 2. Partitioning the Table
- For very large tables, consider table partitioning. Partitioning breaks the table into smaller, more manageable pieces, each of which can be processed more efficiently. In PostgreSQL, range partitioning or list partitioning is commonly used.
- Example: Partitioning the users table by a date column (e.g., created_at), if users sign up over time.
    CREATE TABLE users (
      user_id SERIAL PRIMARY KEY,
      username VARCHAR(100),
      email VARCHAR(100),
      phone VARCHAR(20),
      created_at TIMESTAMP NOT NULL
  ) PARTITION BY RANGE (created_at);
  
  -- Create partitions for each year
  CREATE TABLE users_2023 PARTITION OF users FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
  CREATE TABLE users_2024 PARTITION OF users FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
- With partitioning, queries on a specific time range will only scan relevant partitions, speeding up the execution.

## 3. Table Maintenance (VACUUM and ANALYZE)
- VACUUM:
  - Over time, PostgreSQL tables accumulate dead rows due to updates and deletes. Running VACUUM reclaims this space and keeps your tables efficient.
    
- VACUUM FULL users;
 - Alternatively, use VACUUM ANALYZE to avoid locking the table but still optimize performance:
-  VACUUM ANALYZE users;
- ANALYZE:
  - Updates the query planner's statistics, which helps PostgreSQL make better decisions about how to execute queries.
  - ANALYZE users;

## 4. Use Appropriate Data Types

- Smaller Data Types:
 - Ensure you are using the most efficient data types. For example:
   - Use VARCHAR(50) instead of TEXT when possible.
   - Use INT instead of BIGINT if the range of values is small.
   - Use NUMERIC(13, 2) for decimal numbers if precision requirements allow for it.
- Compression with TOAST:
  -  PostgreSQL automatically compresses large TEXT or BYTEA columns using TOAST. However, avoid storing excessively large data (e.g., images, documents) directly in the table. Instead, store paths or references and use external storage solutions.

## 5. Query Optimization

- Batch Queries:
  - Instead of making multiple small queries, batch updates and inserts to reduce overhead.
      INSERT INTO users (username, email, phone)
      VALUES ('user1', 'user1@example.com', '123456'),
             ('user2', 'user2@example.com', '654321');
 - **Avoid SELECT ***: Only retrieve the columns you need in queries. This reduces the amount of data PostgreSQL has to fetch, process, and return:
 - SELECT username, email FROM users WHERE status = 'active';
- Limit the Results:
  -  For certain queries (e.g., searches), use LIMIT to restrict the number of rows returned:
  -  SELECT username, email FROM users WHERE status = 'active' LIMIT 100;

## 6. Archiving Old Data

- If your table contains a lot of historical data that is infrequently accessed, consider archiving old data to another table or a separate database. This reduces the size of the working table and improves performance.

- For example, if you have old users who haven’t logged in for years, move their data to an archive table:
   CREATE TABLE archived_users AS SELECT * FROM users WHERE last_login < '2020-01-01';
   DELETE FROM users WHERE last_login < '2020-01-01';

## 7. Optimize Hardware and Configuration
- Memory Settings:
  - Ensure PostgreSQL has enough memory allocated. Adjust parameters like shared_buffers, work_mem, and maintenance_work_mem in your postgresql.conf file based on the available hardware.
- SSD Storage:
  - Use SSDs for faster disk I/O, especially with large datasets.
- Parallel Query Execution:
  -  PostgreSQL can run parallel queries for large data sets. Make sure parallel execution is enabled (with settings like max_parallel_workers_per_gather).

## 8. Connection Pooling
- For high traffic, use connection pooling to handle multiple client requests more efficiently. A tool like pgBouncer can help manage connections by reducing the overhead of opening and closing connections frequently.

## 9. Denormalization
- In some cases, a denormalized table can be more efficient for read-heavy workloads. This involves storing redundant data to avoid expensive JOIN operations. For example, if you frequently join the users table with another table, consider adding some of that information directly to the user's table. However, be cautious as this can increase data redundancy and complicate updates.

## 10. Indexing Foreign keys
- 

## Performance Comparison (with and without optimization)
- Let's consider a query that fetches a user profile by username from a table of 1 million rows.
- Without optimization:  (no index, no partitioning, unoptimized queries):
  - The query may take several seconds, as PostgreSQL will perform a full table scan.
  - Estimated time: 2-5 seconds depending on the hardware and data structure.
- With optimization
  - The same query may be completed in milliseconds since PostgreSQL will use the index and avoid scanning the entire table.
  - Estimated time: < 100 milliseconds.

## Conclusion
- For a large table (like 10 GB or more) with growing data, you can optimize the performance by:
- sing efficient indexing strategies like multicolumn and covering indexes.
- Implementing table partitioning for large datasets.
- Running VACUUM and ANALYZE to maintain table health.
- Ensuring queries are optimized with selective column retrieval and using batch queries.
- Archiving old data and denormalizing where appropriate.

These techniques combined will help ensure that your queries remain fast, even as your data continues to grow.


### Cluster Command

- The cluster command works by making a whole new copy of the table, sorted by the index you asked for. once built, the original copy of the data is dropped
- cluster test_table using idx_random;
- 
### fillfactor

-  the page size is fixed at 8 KB (8192 bytes). Each page can store multiple rows of data, and the maximum number of rows or the amount of data that can fit on a page depends on several factors
-  
        -- Create a new table with fillfactor set to 80%
      CREATE TABLE my_table_with_fillfactor (
          id SERIAL PRIMARY KEY,
          data TEXT
      ) WITH (fillfactor = 80);
   
-
### 
