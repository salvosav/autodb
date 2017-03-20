# Autodb

A very simple automated single table read-write Active Record Pattern implementation.

LIMITATIONS TO BE AWARE OF BEFORE YOU WOULD USE:

    This is not ORM. Just an active record pattern, it doesn't support joins on purpose.
    One AutoRecord instance = one row in one database's one table
    One AutoDb instance <-> One SQL database connection
    Static saveMore() and deleteMore() methods will only run on same AutoDb, same mysqli, same table rows, otherwise throwing AutoDbException. (Empty array won't throw)
    Also save(), row(), rowsArray(), newRow(), saveMore() and deleteMore() are final for a reason
    Recommended usage - AutoRecord(s) as class member (composition), AutoDb (one per connection instance) as singleton or in any container globally available
    One Database, one Table two calls for Primary key -> use the same AutoDb instance and you will never have a duplicated AutoRecord instance
    Redis is an optional dependency for caching table describes.
    If using more AutoDb (more tables or more connections) with Redis table definition caching, you WANT to set different $ident for AutoDb::init()
    If using Redis cache you WANT to purge 'autodbdefs.*' redis keys in your deploy script, or at least you REALLY WANT TO delete it right after running an ALTER TABLE
    Concurrent writing (INSERT IGNORE INTO, REPLACE INTO) is supported, but only via AutoRecord::saveMore(array $arrayOfRecords) which will set saved inserted/replaced references as dead. For limitations see "unit" tests / concurrentWriteTests()
    We do not support editing of primary keys, but you should do that manually and carefully anyway
    Supports only databases with AUTO_INCREMENT POSITIVE INTEGER primary keys
    Everything by AutoDb or AutoRecord will throw a new instance of AutoDbException in all circumstances
    For now, supports only MySQL - with mysqli (PDO planned to be supported very soon) - NEXT THING TO BE IMPLEMENTED
    For now, type checking is very basic, will improve

Usage example:

```php
    <?php
    // create a new AutoDb instance connecting to your database connection. One mysqli resource <-> One AutoDb
    $autoDb = AutoDb::init($mysqli, $optionalRedis); 
    // $mysqli is instanceof mysqli right after you connected
    // one $autoDb per connection is recommended, solve it with your own conatainer/singleton for best results
    
    //new record
    $rec = $autoDb->newRow('my_table'); // loads table definition once in runtime with describe 
    $rec->attr('some_column', 'newvalue');
    $rec->save();
    echo $rec->attr('my_table_id'); // gives you back the column value
    
    //existing record
    $rec = $autoDb->row('my_table', 'my_table_id', 23); // returns row with primary key my_table_id 23
    $rec->attr('some_column', 'changed_value');
    $rec->save();
    
    // for array of AutoRecord instances with one single query (and returns object cache version if exists)
    $arrayRecords = $autoDb->rowsArray($table, $where, $limit, $page); // limit default is -1 (unlimited results), page defaults to 1
    // BEWARE - 2nd parameter here is just an UNESCAPED string
    
    // You may also get a previous but unsaved value of an attr you already saved:
    echo $x->attr('username'); // 'changedPreviously'
    $x->attr('username', 'changedAgain');
    echo $x->attr('username'); // 'changedAgain'
    echo $x->dbAttr('username'); // 'changedPreviously' - from instance
    echo $x->dbAttrForce('username'); // 'changedPreviously' - from database
    $x->save(); // now all will be 'changedAgain'
    
    // During save() after INSERT/UPDATE there is no SELECT to populate the columns in the database. You can force it by calling $record->forceReloadAttributes(). 
    // (But primary key is up to date always after save(), even without calling $record->forceReloadAttributes() method)
    $row->attr('created_at', 'NOW()');
    $row->save();
    echo $row->attr('created_at');         // 'NOW()' :(
    echo $row->dbAttrForce('created_at');  // '2017-03-20 11:11:11'
    $row->forceReloadAttributes();         // the (not always required) extra query to sync column attributes
    echo $row->attr('created_at');         // '2017-03-20 11:11:11' :)
    
    // AutoDb also supports Read only, Write once and blocked tables:
    $autoDb->addBannedTable('sensitive_table');
    $autoDb->addWriteOnceTable('my_strict_log');
    $autoDb->addReadOnlyTable('categories');
    
    // You can insert more new rows with one single query optimally using 
    AutoRecord::saveMore($arrayOfSameTableAutorecords);
    // This line will also update one by one the lines already having a primary key
    
    // You can also delete in one single query multiple lines
    AutoRecord::deleteMore($arrayOfSameTableAutorecords);
```

CONCURRENT WRITE SUPPORT

```php
    <?php
    $row = $this->testAdb->newRow('unik');
    $row->attr('uniq_part_1', 'I_am_first_part_of_unique_key');
    $row->attr('uniq_part_2', 10);
    // $row->save(); // Would throw Exception if Unique key already exists (sometimes you want this though)
    
    // your options:
    AutoRecord::saveMore(array($row), 'INSERT IGNORE INTO'); // if there was a row using this unique key, that one wins
    AutoRecord::saveMore(array($row), 'REPLACE INTO'); // always the later write wins
    AutoRecord::saveMore(array($row), 'INSERT INTO', 'ON DUPLICATE KEY UPDATE request_count = request_count + 1'); // "manual"
    // YOU CANNOT USE $row AFTER ANY OF THESE, IT IS A DEAD REFERENCE, select again via rowsArray() by Unique keys, if you want to keep working with the row
    // REPLACE INTO also breaks previous record (by nature of mysql, that primary key value doesn't exist anymore)
    
    // after a concurrent write reload row by unique key, or you cannot work with it (dead reference):
    $row = MyAppContainer::getAutoDb()->rowsArray('unik', "uniq_part_1 = 'I_am_first_part_of_unique_key' AND uniq_part_2 = 10")[0]; // array[1] not set as unique
    
    // For more details and limitations on concurrent write see tests/AutoDbTest.php method concurrentWriteTests()
```

"UNIT" TESTS

```php
    <?php
    // to run "unit" tests (rather an Integration test) add to project root a file test_mysql_connection_credentials.php as stated in tests/bootstrap.php:
    
    // GITIGNORED FILE:
    
    define('MYSQL_HOST', 'localhost');
    define('MYSQL_USER', 'youruser');
    define('MYSQL_PASSWORD', 'yourpassword');
    
```
