---
slug: /en/guides/improving-query-performance/sparse-primary-indexes/sparse-primary-indexes-design
sidebar_label: ClickHouse Index Design 
sidebar_position: 2
description: todo
---
# ClickHouse Index Design



## An index design for massive data scales

In traditional relational database management systems the primary index would contain one entry per table row. For our data set this would result in the  primary index - often a <a href="https://en.wikipedia.org/wiki/B%2B_tree" target="_blank">B(+)-Tree</a> data structure - containing 8.87 million entries.

Such an index allows the fast location of specific rows, resulting in high efficiency for lookup queries and point updates. Searching an entry in a B(+)-Tree data structure has average time complexity of <font face = "monospace">O(log2 n)</font>. For a table of 8.87 million rows this means 23 steps are required to locate any index entry.

This capability comes at a cost: additional disk and memory overheads and higher insertion costs when adding new rows to to the table and entries to the index (and also sometimes rebalancing of the B-Tree).

Considering the challenges associated with B-Tree indexes, table engines in ClickHouse utilise a different approach. The ClickHouse [MergeTree Engine Family](/docs/en/engines/table-engines/mergetree-family/index.md) has been designed and  optimized to handle massive data volumes.

These tables are designed to receive millions of row inserts per second and store very large (100s of Petabytes) volumes of data.

Data is quickly written to a table [part by part](/docs/en/engines/table-engines/mergetree-family/mergetree.md/#mergetree-data-storage), with rules applied for merging the parts in the background.

In ClickHouse each part has its own primary index. When parts are merged, then the merged part’s primary indexes are also merged.

At the very large scale that ClickHouse is designed for, it is paramount to be very disk and memory efficient. Therefore, instead of indexing every row, the primary index for a part has one index entry (known as a ‘mark’) per group of rows (called ‘granule’).

This sparse indexing is possible because ClickHouse is storing the rows for a part on disk ordered by the primary key column(s).

Instead of directly locating single rows (like a B-Tree based index), the sparse primary index allows it to quickly (via a binary search over index entries) identify groups of rows that could possibly match the query.

The located groups of potentially matching rows (granules) are then in parallel streamed into the ClickHouse engine in order to find the matches.

This index design allows for the primary index to be small (it can, and must, completely fit into the main memory), whilst still significantly speeding up query execution times: especially for range queries that are typical in data analytics use cases.

The following illustrates in detail how ClickHouse is building and using its sparse primary index.
Later on in the article we will discuss some best practices for choosing, removing, and ordering the table columns that are used to build the index (primary key columns).

## A table with a primary key



Create a table that has a compound primary key with key columns UserID and URL:

```sql
CREATE TABLE hits_UserID_URL
(
    `UserID` UInt32,
    `URL` String,
    `EventTime` DateTime
)
ENGINE = MergeTree
// highlight-next-line
PRIMARY KEY (UserID, URL)
ORDER BY (UserID, URL, EventTime)
SETTINGS index_granularity = 8192, index_granularity_bytes = 0;
```

[//]: # (<details open>)
<details>
    <summary><font color="black">
    DDL Statement Details
    </font></summary>
    <p><font color="black">

In order to simplify the discussions later on in this guide, as well as  make the diagrams and results reproducible, the DDL statement
<ul>
<li>specifies a compound sorting key for the table via an <font face = "monospace">ORDER BY</font> clause</li>
<br/>
<li>explicitly controls how many index entries the primary index will have through the settings:</li>
<br/>
<ul>
<li><font face = "monospace">index_granularity</font>: explicitly set to its default value of 8192. This means that for each group of 8192 rows, the primary index will have one index entry, e.g. if the table contains 16384 rows then the index will have two index entries.
</li>
<br/>
<li><font face = "monospace">index_granularity_bytes</font>: set to 0 in order to disable <a href="https://clickhouse.com/docs/en/whats-new/changelog/2019/#experimental-features-1" target="_blank"><font color="blue">adaptive index granularity</font></a>. Adaptive index granularity means that ClickHouse automatically creates one index entry for a group of n rows
<ul>
<li>if either n is less than 8192 but the size of the combined row data for that n rows is larger than or equal 10 MB (the default value for index_granularity_bytes) or</li>
<li>if the combined row data size for n rows is less than 10 MB but n is 8192.</li>
</ul>
</li>
</ul>
</ul>
</font></p>
</details>


The primary key in the DDL statement above causes the creation of primary index based on the two specified key columns.

<br/>
Next insert the data:

```sql
INSERT INTO hits_UserID_URL SELECT
   intHash32(c11::UInt64) AS UserID,
   c15 AS URL,
   c5 AS EventTime
FROM url('https://datasets.clickhouse.com/hits/tsv/hits_v1.tsv.xz')
WHERE URL != '';
```
The response looks like:
```response
0 rows in set. Elapsed: 149.432 sec. Processed 8.87 million rows, 18.40 GB (59.38 thousand rows/s., 123.16 MB/s.)
```


<br/>
And optimize the table:

```sql
OPTIMIZE TABLE hits_UserID_URL FINAL;
```

<br/>
We can use the following query to obtain metadata about our table:

```sql
SELECT
    part_type,
    path,
    formatReadableQuantity(rows) AS rows,
    formatReadableSize(data_uncompressed_bytes) AS data_uncompressed_bytes,
    formatReadableSize(data_compressed_bytes) AS data_compressed_bytes,
    formatReadableSize(primary_key_bytes_in_memory) AS primary_key_bytes_in_memory,
    marks,
    formatReadableSize(bytes_on_disk) AS bytes_on_disk
FROM system.parts
WHERE (table = 'hits_UserID_URL') AND (active = 1)
FORMAT Vertical;
```

The response is:

```response
part_type:                   Wide
path:                        ./store/d9f/d9f36a1a-d2e6-46d4-8fb5-ffe9ad0d5aed/all_1_9_2/
rows:                        8.87 million
data_uncompressed_bytes:     733.28 MiB
data_compressed_bytes:       206.94 MiB
primary_key_bytes_in_memory: 96.93 KiB
marks:                       1083
bytes_on_disk:               207.07 MiB


1 rows in set. Elapsed: 0.003 sec.
```

The output of the ClickHouse client shows:

- The table’s data is stored in [wide format](/docs/en/engines/table-engines/mergetree-family/mergetree.md/#mergetree-data-storage) in a specific directory on disk meaning that there will be one data file (and one mark file) per table column inside that directory.
- The table has 8.87 million rows.
- The uncompressed data size of all rows together is 733.28 MB.
- The compressed on disk data size of all rows together is 206.94 MB.
- The table has a primary index with 1083 entries (called ‘marks’) and the size of the index is 96.93 KB.
- In total the table’s data and mark files and primary index file together take 207.07 MB on disk.

## Data is stored on disk ordered by primary key column(s)

Our table that we created above has
- a compound [primary key](/docs/en/engines/table-engines/mergetree-family/mergetree.md/#primary-keys-and-indexes-in-queries) <font face = "monospace">(UserID, URL)</font> and
- a compound [sorting key](/docs/en/engines/table-engines/mergetree-family/mergetree.md/#choosing-a-primary-key-that-differs-from-the-sorting-key) <font face = "monospace">(UserID, URL, EventTime)</font>.

:::note
- If we would have specified only the sorting key, then the primary key would be implicitly defined to be equal to the sorting key.

- In order to be memory efficient we explicitly specified a primary key that only contains columns that our queries are filtering on. The primary index that is based on the primary key is completely loaded into the main memory.

- In order to have consistency in the guide’s diagrams and in order to maximise compression ratio we defined a separate sorting key that includes all of our table's columns (if in a column similar data is placed close to each other, for example via sorting, then that data will be compressed better).

- The primary key needs to be a prefix of the sorting key if both are specified.
:::


The inserted rows are stored on disk in lexicographical order (ascending) by the primary key columns (and the additional EventTime column from the sorting key).

:::note
ClickHouse allows inserting multiple rows with identical primary key column values. In this case (see row 1 and row 2 in the diagram below), the final order is determined by the specified sorting key and therefore the value of the <font face = "monospace">EventTime</font> column.
:::



ClickHouse is a <a href="https://clickhouse.com/docs/en/introduction/distinctive-features/#true-column-oriented-dbms
" target="_blank">column-oriented database management system</a>. As show in the diagram below
- for the on disk representation, there is a single data file (*.bin) per table column where all the values for that column are stored in a <a href="https://clickhouse.com/docs/en/introduction/distinctive-features/#data-compression" target="_blank">compressed</a> format, and
- the 8.87 million rows are stored on disk in lexicographic ascending order by the primary key columns (and the additional sort key columns) i.e. in this case
  - first by <font face = "monospace">UserID</font>,
  - then by <font face = "monospace">URL</font>,
  - and lastly by <font face = "monospace">EventTime</font>:

<img src={require('./images/sparse-primary-indexes-01.png').default} class="image"/>
UserID.bin, URL.bin, and EventTime.bin are the data files on disk where the values of the <font face = "monospace">UserID</font>, <font face = "monospace">URL</font>, and <font face = "monospace">EventTime</font> columns are stored.

<br/>
<br/>


:::note
- As the primary key defines the lexicographical order of the rows on disk, a table can only have one primary key.

- We are numbering rows starting with 0 in order to be aligned with the ClickHouse internal row numbering scheme that is also used for logging messages.
:::



## Data is organized into granules for parallel data processing

For data processing purposes, a table's column values are logically divided into granules.
A granule is the smallest indivisible data set that is streamed into ClickHouse for data processing.
This means that instead of reading individual rows, ClickHouse is always reading (in a streaming fashion and in parallel) a whole group (granule) of rows.
:::note
Column values are not physically stored inside granules: granules are just a logical organization of the column values for query processing.
:::

The following diagram shows how the (column values of) 8.87 million rows of our table
are organized into 1083 granules, as a result of the table's DDL statement containing the setting <font face = "monospace">index_granularity</font> (set to its default value of 8192).

<img src={require('./images/sparse-primary-indexes-02.png').default} class="image"/>

The first (based on physical order on disk) 8192 rows (their column values) logically belong to granule 0, then the next 8192 rows (their column values) belong to granule 1 and so on.

:::note
- The last granule (granule 1082) "contains" less than 8192 rows.

- We marked some column values from our primary key columns (<font face = "monospace">UserID</font>, <font face = "monospace">URL</font>) in orange.

  These orange marked column values are the minimum value of each primary key column in each granule. The exception here is the last granule (granule 1082 in the diagram above) where we mark the maximum values.

  As we will see below, these orange marked column values will be the entries in the table's primary index.

- We are numbering granules starting with 0 in order to be aligned with the ClickHouse internal numbering scheme that is also used for logging messages.
:::



## The primary index has one entry per granule

The primary index is created based on the granules shown in the diagram above. This index is an uncompressed flat array file (primary.idx), containing so-called numerical index marks starting at 0.

The diagram below shows that the index stores the minimum primary key column values (the values marked in orange in the diagram above) for each granule.
For example
- the first index entry (‘mark 0’ in the diagram below) is storing the minimum values for the primary key columns of granule 0 from the diagram above,
- the second index entry (‘mark 1’ in the diagram below) is storing the minimum values for the primary key columns of granule 1 from the diagram above, and so on.



<img src={require('./images/sparse-primary-indexes-03a.png').default} class="image"/>

In total the index has 1083 entries for our table with 8.87 million rows and 1083 granules:

<img src={require('./images/sparse-primary-indexes-03b.png').default} class="image"/>

:::note
- The last index entry (‘mark 1082’ in the diagram below) is storing the maximum values for the primary key columns of granule 1082 from the diagram above.

- Index entries (index marks) are not based on specific rows from our table but on granules. E.g. for index entry ‘mark 0’ in the diagram above there is no row in our table where <font face = "monospace">UserID</font> is <font face = "monospace">240.923</font> and <font face = "monospace">URL</font> is <font face = "monospace">"goal://metry=10000467796a411..."</font>, instead, there is a granule 0 for the table where within that granule the minimum UserID vale is <font face = "monospace">240.923</font> and the minimum URL value is <font face = "monospace">"goal://metry=10000467796a411..."</font> and these two values are from separate rows.

- The primary index file is completely loaded into the main memory. If the file is larger than the available free memory space then ClickHouse will raise an error.
:::


The primary key entries are called index marks because each index entry is marking the start of a specific data range. Specifically for the example table:
- UserID index marks:<br/>
  The stored <font face = "monospace">UserID</font> values in the primary index are sorted in ascending order.<br/>
  ‘mark 1’ in the diagram above thus indicates that the <font face = "monospace">UserID</font> values of all table rows in granule 1, and in all following granules, are guaranteed to be greater than or equal to 4.073.710.

 [As we will see later](#the-primary-index-is-used-for-selecting-granules), this global order enables ClickHouse to <a href="https://github.com/ClickHouse/ClickHouse/blob/22.3/src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp#L1452" target="_blank">use a binary search algorithm</a> over the index marks for the first key column when a query is filtering on the first column of the primary key.

- URL index marks:<br/>
  The quite similar cardinality of the primary key columns <font face = "monospace">UserID</font> and <font face = "monospace">URL</font> means that the index marks for all key columns after the first column in general only indicate a data range per granule.<br/>
 For example, ‘mark 0’ for the <font face = "monospace">URL</font> column in the diagram above is indicating that the URL values of all table rows in granule 0 are guaranteed to be larger or equal to <font face = "monospace">goal://metry=10000467796a411...</font>. However, this same guarantee cannot also be given for the <font face = "monospace">URL</font> values of all table rows in granule 1  because ‘mark 1‘ for the <font face = "monospace">UserID</font> column has a different UserID value than ‘mark 0‘.

  We will discuss the consequences of this on query execution performance in more detail later.

## The primary index is used for selecting granules

We can now execute our queries with support from the primary index.


The following calculates the top 10 most clicked urls for the UserID 749927693.

```sql
SELECT URL, count(URL) AS Count
FROM hits_UserID_URL
WHERE UserID = 749927693
GROUP BY URL
ORDER BY Count DESC
LIMIT 10;
```

The response is:


```response
┌─URL────────────────────────────┬─Count─┐
│ http://auto.ru/chatay-barana.. │   170 │
│ http://auto.ru/chatay-id=371...│    52 │
│ http://public_search           │    45 │
│ http://kovrik-medvedevushku-...│    36 │
│ http://forumal                 │    33 │
│ http://korablitz.ru/L_1OFFER...│    14 │
│ http://auto.ru/chatay-id=371...│    14 │
│ http://auto.ru/chatay-john-D...│    13 │
│ http://auto.ru/chatay-john-D...│    10 │
│ http://wot/html?page/23600_m...│     9 │
└────────────────────────────────┴───────┘

10 rows in set. Elapsed: 0.005 sec.
// highlight-next-line
Processed 8.19 thousand rows,
740.18 KB (1.53 million rows/s., 138.59 MB/s.)
```

The output for the ClickHouse client is now showing that instead of doing a full table scan, only 8.19 thousand rows were streamed into ClickHouse.


If <a href="https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings/#server_configuration_parameters-logger" target="_blank">trace logging</a> is enabled then the ClickHouse server log file shows that ClickHouse was running a <a href="https://github.com/ClickHouse/ClickHouse/blob/22.3/src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp#L1452" target="_blank">binary search</a> over the 1083 UserID index marks, in order to identify granules that possibly can contain rows with a UserID column value of <font face = "monospace">749927693</font>. This requires 19 steps with an average time complexity of <font face = "monospace">O(log2 n)</font>:
```response
...Executor): Key condition: (column 0 in [749927693, 749927693])
// highlight-next-line
...Executor): Running binary search on index range for part all_1_9_2 (1083 marks)
...Executor): Found (LEFT) boundary mark: 176
...Executor): Found (RIGHT) boundary mark: 177
...Executor): Found continuous range in 19 steps
...Executor): Selected 1/1 parts by partition key, 1 parts by primary key,
// highlight-next-line
              1/1083 marks by primary key, 1 marks to read from 1 ranges
...Reading ...approx. 8192 rows starting from 1441792
```


We can see in the trace log above, that one mark out of the 1083 existing marks satisfied the query.

<details>
    <summary><font color="black">
    Trace Log Details
    </font></summary>
    <p><font color="black">

Mark 176 was identified (the 'found left boundary mark' is inclusive, the 'found right boundary mark' is exclusive), and therefore all 8192 rows from granule 176 (which starts at row 1.441.792 - we will see that later on in this guide) are then streamed into ClickHouse in order to find the actual rows with a UserID column value of <font face = "monospace">749927693</font>.
</font></p>
</details>

We can also reproduce this by using the <a href="https://clickhouse.com/docs/en/sql-reference/statements/explain/" target="_blank">EXPLAIN clause</a> in our example query:
```sql
EXPLAIN indexes = 1
SELECT URL, count(URL) AS Count
FROM hits_UserID_URL
WHERE UserID = 749927693
GROUP BY URL
ORDER BY Count DESC
LIMIT 10;
```

The response looks like:

```response
┌─explain───────────────────────────────────────────────────────────────────────────────┐
│ Expression (Projection)                                                               │
│   Limit (preliminary LIMIT (without OFFSET))                                          │
│     Sorting (Sorting for ORDER BY)                                                    │
│       Expression (Before ORDER BY)                                                    │
│         Aggregating                                                                   │
│           Expression (Before GROUP BY)                                                │
│             Filter (WHERE)                                                            │
│               SettingQuotaAndLimits (Set limits and quota after reading from storage) │
│                 ReadFromMergeTree                                                     │
│                 Indexes:                                                              │
│                   PrimaryKey                                                          │
│                     Keys:                                                             │
│                       UserID                                                          │
│                     Condition: (UserID in [749927693, 749927693])                     │
│                     Parts: 1/1                                                        │
// highlight-next-line
│                     Granules: 1/1083                                                  │
└───────────────────────────────────────────────────────────────────────────────────────┘

16 rows in set. Elapsed: 0.003 sec.
```
The client output is showing that one out of the 1083 granules was selected as possibly containing rows with a UserID column value of 749927693.


:::note Conclusion
When a query is filtering on a column that is part of a compound key and is the first key column, then ClickHouse is running the binary search algorithm over the key column's index marks.
:::

<br/>


As discussed above, ClickHouse is using its sparse primary index for quickly (via binary search) selecting granules that could possibly contain rows that match a query.


This is the **first stage (granule selection)** of ClickHouse query execution.

In the **second stage (data reading)**, ClickHouse is locating the selected granules in order to stream all their rows into the ClickHouse engine in order to find the rows that are actually matching the query.

We discuss that second stage in more detail in the following section.



## Mark files are used for locating granules

The following diagram illustrates a part of the primary index file for our table.

<img src={require('./images/sparse-primary-indexes-04.png').default} class="image"/>

As discussed above, via a binary search over the index’s 1083 UserID marks, mark 176 were identified. Its corresponding granule 176 can therefore possibly contain rows with a UserID column value of 749.927.693.

<details>
    <summary><font color="black">
    Granule Selection Details
    </font></summary>
    <p><font color="black">

The diagram above shows that mark 176 is the first index entry where both the minimum UserID value of the associated granule 176 is smaller than 749.927.693, and the minimum UserID value of granule 177 for the next mark (mark 177) is greater than this value. Therefore only the corresponding granule 176 for mark 176 can possibly contain rows with a UserID column value of 749.927.693.
</font></p>
</details>

In order to confirm (or not) that some row(s) in granule 176 contain a UserID column value of 749.927.693, all 8192 rows belonging to this granule need to be streamed into ClickHouse.

To achieve this, ClickHouse needs to know the physical location of granule 176.

In ClickHouse the physical locations of all granules for our table are stored in mark files. Similar to data files, there is one mark file per table column.

The following diagram shows the three mark files UserID.mrk, URL.mrk, and EventTime.mrk that store the physical locations of the granules for the table’s UserID, URL, and EventTime columns.
<img src={require('./images/sparse-primary-indexes-05.png').default} class="image"/>

We have discussed how the primary index is a flat uncompressed array file (primary.idx), containing index marks that are numbered starting at 0.

Similarily, a mark file is also a flat uncompressed array file (*.mrk) containing marks that are numbered starting at 0.

Once ClickHouse has identified and selected the index mark for a granule that can possibly contain matching rows for a query, a positional array lookup can be performed in the mark files in order to obtain the physical locations of the granule.

Each mark file entry for a specific column is storing two locations in the form of offsets:

- The first offset ('block_offset' in the diagram above) is locating the <a href="https://clickhouse.com/docs/en/development/architecture/#block" target="_blank">block</a> in the <a href="https://clickhouse.com/docs/en/introduction/distinctive-features/#data-compression" target="_blank">compressed</a> column data file that contains the compressed version of the selected granule. This compressed block potentially contains a few compressed granules. The located compressed file block is uncompressed into the main memory on read.

- The second offset ('granule_offset' in the diagram above) from the mark-file provides the location of the  granule within the uncompressed block data.

All the 8192 rows belonging to the located uncompressed granule are then streamed into ClickHouse for further processing.


:::note Why Mark Files

Why does the primary index not directly contain the physical locations of the granules that are corresponding to index marks?

Because at that very large scale that ClickHouse is designed for, it is important to be very disk and memory efficient.

The primary index file needs to fit into the main memory.

For our example query ClickHouse used the primary index and selected a single granule that can possibly contain rows matching our query. Only for that one granule does ClickHouse then need the physical locations in order to stream the corresponding rows for further processing.

Furthermore, this offset information is only needed for the UserID and URL columns.

Offset information is not needed for columns that are not used in the query e.g. the EventTime.

For our sample query, ClickHouse needs only the two physical location offsets for granule 176 in the UserID data file (UserID.bin) and the two physical location offsets for granule 176 in the URL data file (URL.bin).

The indirection provided by mark files avoids storing, directly within the primary index, entries for the physical locations of all 1083 granules for all three columns: thus avoiding having unnecessary (potentially unused) data in main memory.
:::

The following diagram and the text below illustrates how for our example query ClickHouse locates granule 176 in the UserID.bin data file.

<img src={require('./images/sparse-primary-indexes-06.png').default} class="image"/>

We discussed earlier in this guide that ClickHouse selected the primary index mark 176 and therefore granule 176 as possibly containing matching rows for our query.

ClickHouse now uses the selected mark number (176) from the index for a positional array lookup in the UserID.mrk mark file in order to get the two offsets for locating granule 176.

As shown, the first offset is locating the compressed file block within the UserID.bin data file that in turn contains the compressed version of granule 176.

Once the located file block is uncompressed into the main memory, the second offset from the mark file can be used to locate granule 176 within the uncompressed data.

ClickHouse needs to locate (and stream all values from) granule 176 from both the UserID.bin data file and the URL.bin data file in order to execute our example query (top 10 most clicked urls for the internet user with the UserID 749.927.693).

The diagram above shows how ClickHouse is locating the granule for the UserID.bin data file.

In parallel, ClickHouse is doing the same for granule 176 for the URL.bin data file. The two respective granules are aligned and streamed into the ClickHouse engine for further processing i.e. aggregating and counting the URL values per group for all rows where the UserID is 749.927.693, before finally outputting the 10 largest URL groups in descending count order.


