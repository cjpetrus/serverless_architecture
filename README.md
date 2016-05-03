# serverless_architecture
a repo for my notes on plexseo's serverless, event-driven, crawler architecture
![alt tag](https://github.com/cjpetrus/serverless_architecture/blob/master/plexseo_serverless_architecture.jpg?raw=true)

#DynamoDB2 Notes:

* database -> tables -> items -> attributes
  * db = a collection of tables
  * table = a collection of items
  * item = a collection of attributes
* use HTTP + JSON to talk with DynamoDB server
* use [Amazon Elastic MapReduce](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EMRforDynamoDB.html) to join/export/import/query tables.

## Attributes
* attribute value cannot be null, empty, or empty set
* data types:
  * string, number, binary
  * set of string, number, binary e.g. ['ruby', 'rails']
    * set = no duplicate value, unordered
   
## Items
* each item max size 64kb
* size = sum of lengths of attribute names + values (binary/utf8 length)
* 1 required attribute = primary key

### Operations
* getItem, putItem, updateAttributes(add,put,delete), deleteItem
* batchGetItems, batchWriteItems(put/delete)
* put - 1) create if not exists, 2) replace if exists (can turn off with `Exists`)
* **conditional write** -  put item.amount=10 if item.amount=8
* **atomic counter**, *not* idempotent! - increment/decrement counter
* **return values before write** - returns item and delete it
* when read, can choose between 1) strong consistency, 2) eventual consistency
  * Strong Read costs 2x read capacity unit of Eventual Read


## Tables
* primary key
  1. hash key
  2. hash key + range key
  
* calculate throughput - capacity unit
  * 1 read capacity unit
    * = 1 strong consistency read operation (max 4kb) / second
    * = 2 eventual consistency read operation (max 4b) / second
  * 1 write capacity unit = 1 write operation (max 1kb) / second

### Table design tips 

#### hash key 

* table hash key selection **does affect** actual throughput
  * why: dynamo distributes items across partitions based on hash key
  * good: random hash key
  * bad: hash key with few possible values (e.g. status code)
  * bad: hash key with certain popular values (e.g. 50% of items has the same device code)


#### time series data

* e.g. events, logs, messages
* access pattern: new data has more frequent access than old data.
* tips: split data by year/month/week into multiple tables e.g. events_2013jan
  * then adjust read/write throughput for each table based on access pattern
  

#### split tables according to access pattern

* The cost of updating a single attribute of an item is based on the full size of the item.
* e.g. split ProductCatalog into ProductCatalog (frequent read), ProductPricingAvailability (frequent write), ExtendedProductCatalog (seldom read)

#### large attribute value

* gzip and store as binary e.g long review message
* store in Amazon S3 e.g. images
* split a large item into multiple items stored in separate table
  * not recommended
  * need to handle cross-item transaction yourself

## Query & Scan
* a single `Scan` request can consume (1 MB page size / 4 KB item size) / 2 (eventually consistent reads) = 128 read operations.
*  A larger number of smaller `Scan` or `Query`     operations would allow your other critical requests to succeed without throttling.
* Some applications handle `Scan` load by rotating traffic hourly between two tables – one for critical traffic, and one for bookkeeping. Other applications can do this by performing every write on two tables: a "mission-critical" table, and a "shadow" table.


## Secondary Index
* 2 types: local, global
* for each table - max 5 local secondary key, 5 global secondary key
* cannot add/modify/delete secondary index to an existing table.
* when create several tables with secondary index, must create one by one, otherwise `LimitExceededException`.
* when `QUERY`, must specify the index name to use.

### local
* primary key = hash key + range key
* hash key = table's hash key, range key = any scalar attribute in table
* for each hash key, total indexed items <= 10GB
* query single partition only
* can choose: strong or eventual consistency read.
* when query index, consume read capacity units from the table.
* when write table, write to index too, charge extra write capacity units to table.
* when query index, can select any attributes (costs extra fetches to table if select attributes not included in index)

#### local index - design tips

* take advantage of **sparse index**
  * an item only appears in index, when the range-key exists
  * e.g. to search active users, add attribute IsActive="Y" to active user, delete attribute "IsActive" from inactive user. Query on index has fewer reads, lower costs.

### global
* primary key = hash key OR (hash key + range key)
* free to select any hash key, range key
* no max size restrictions
* query across all partitions
* support eventual consistency read only.
* has its own read/write throughput capacity (charged separately)
* when query index, can only select attributes included in the index
* the key values in a global secondary index do not need to be unique

#### global index - design tips

* select key for better distribution across partitions
* take advantage of sparse index
* tips: for quick lookup - create global index with same primary key as table, but sub-set of attributes.
* tips: global index as read replica - to deal with high traffic, clones table to a global index, then use it for high volume reads; original table continues to handle writes.

## Best Practices

See [http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices.html](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BestPractices.html)

## Limitations

See [http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html)
