### Driver types and Loading driver

- Type 1: JDBC-ODBC bridge driver
- Type 2: Java + Native code driver
- Type 3: All Java + Middleware translation driver
- Type 4: All Java driver

A call to `Class.forName("X")` causes the class named **X** to be dynamically loaded (at runtime) if it not already loaded and initialized (JVM executes all its **static block** after class loading).

Thus `Class.forName("com.mysql.jdbc.Driver")` load and init the MySQL driver via **static initializer block** that registers the class with `DriverManager.registerDriver()`.

From Java 6 there is no need to use `Class.forName()` because the **DriverManager** methods `getConnection` and `getDrivers` have been enhanced to support the Java Service Provider mechanism.


#### DriverManager

This class manages a list of database drivers. Does not support **connection pooling**.
To create the db connection the **DriverManager** class has to know which database driver you want to use. It does that by iterating over the array of drivers that have registered with it and calls the `acceptsURL(url)` method on each driver in the array, effectively asking the driver to tell it whether or not it can handle the JDBC URL. So the first driver that recognizes a certain subprotocol under JDBC will be used to establish a database Connection.


#### DataSource

A factory for connections to the physical data source that this **DataSource** object represents. An alternative to the **DriverManager** facility, a **DataSource** object is the preferred means of getting a connection.
- using a **DataSource** you only need to know the JNDI name. The AppServer cares about the details (url, username, password) and is not configured by the client application's vendor, but by an admin where the application is hosted.
- improves application performance as connections are not created/closed within a class, they are managed by the application server and can be fetched while at runtime.
- it provides a facility creating a pool of connections


### Connection request workflow

`App -> DataSource -> DriverManager -> Driver -> Connection -> SocketFactory -> Socket -> DB`
- Without connection pooling
	1. The application data layer asks the **DataSource** for a database connection
	2. The **DataSource** will use the underlying driver to open a physical connection
	3. A physical connection is created, and a *TCP socket* is opened
	4. The **DataSource** simply returns the physical connection to the application layer
	5. The application executes statements using the acquired database connection
	6. When the connection is no longer needed, the application closes the physical connection along with the underlying *TCP socket*

`App -> DataSource -> Connection Pool -> LogicalConnection -> Connection  -> DB`
- With connection pooling
	1. When a connection is being requested, the pool looks for unallocated connections
	2. If the pool finds a free one, it handles it to the client
	3. If there is no free connection, the pool tries to grow to its maximum allowed size
	4. If the pool already reached its maximum size, it will retry several times before giving up with a connection acquisition failure exception
	5. When the client closes the **logical connection**, the connection is released and returns to the pool without closing the underlying **physical connection**.

The **connection pool** doesn’t return the **physical connection** to the client, but instead it offers a proxy or a handle. When a connection is in use, the pool changes its state to allocated to prevent two concurrent threads from using the same database connection. The proxy intercepts the connection close method call, and it notifies the pool to change the connection state to unallocated.



### Connection

A connection (session) with a specific database. SQL statements are executed and results are returned within the context of a connection.

- `DriverManager.getConnection(String url)`
- `DriverManager.getConnection(String url, username, password)`
- `DriverManager.getConnection(String url, Properties info)`

A method `setTransactionIsolation(int)` allows to set transactional isolation level.

 - TRANSACTION_READ_UNCOMMITTED
 - TRANSACTION_READ_COMMITTED
 - TRANSACTION_REPEATABLE_READ
 - TRANSACTION_SERIALIZABLE


### How most relational databases handles a JDBC/SQL query:
- Parsing of SQL query
- Compilation of SQL Query
- Planning and optimization of data acquisition path
- Executing the optimized query and return the resulted data


### Statement

When we use **Statement**, it goes through all the four steps above but with **PreparedStatement** first three steps are executed when we create the prepared statement.

- `boolean execute(String SQL)` returns true if a **ResultSet** can be retrieved; otherwise, it returns false
- `int executeUpdate(String SQL)` returns the number of rows affected by the execution (INSERT/UPDATE/DELETE)
- `ResultSet executeQuery(String SQL)` returns a **ResultSet**

Using `close()` we can close a **Connection** object to save database resources method. If you close the **Connection** object first, it will close the **Statement** object as well. However, you should always explicitly close the **Statement** object to ensure proper cleanup.

To obtain the insert ID from *INSERT* statement we can use `Statement#getGeneratedKeys()` but first we need to create the statement using `Statement.RETURN_GENERATED_KEYS` to notify the JDBC driver to return the keys.

- `setFetchSize(int)` gives the JDBC driver a hint as to the number of rows that should be fetched from the database. Limit a number that will be returned in each **database roundtrip**.
- `setMaxRows(int)` sets the limit for the maximum number of rows that any **ResultSet** object generated by this **Statement** object can contain to the given number.


#### PreparedStatement

Advantages of **PreparedStatement** over **Statement**:
- Pre-compiled (once), so faster for repeated execution of dynamic SQL (where parameters change)
- DB-side caching of the SQL statement leads to overall faster execution and the ability to reuse the same SQL statement in batches
- Protect against SQL injection, by escaping text for all the parameter values provided
- Helps to write object oriented code with setter methods whereas with **Statement** we have to use String concatenation to create the query
- One of the limitation of **PreparedStatement** is that we can’t use it for SQL queries with IN clause because **PreparedStatement** doesn’t allow us to bind multiple values for single placeholder (?).

The **PreparedStatement** object only uses the **IN** parameter. The `setXXX()` methods bind values to the parameters, where XXX represents the Java data type of the value you wish to bind to the input parameter. If you forget to supply the values, you will receive an **SQLException**.

A **PreparedStatement** object has the ability to use input and output streams to supply parameter data. This enables you to place entire files into database columns that can hold large values, such as **CLOB** and **BLOB**.
- `setAsciiStream()` used to supply large ASCII values
- `setCharacterStream()` used to supply large UNICODE values
- `setBinaryStream()` used to supply large binary values

```java
File file = new File("XML_Data.xml");

pstmt = conn.prepareStatement("INSERT INTO XML_Data VALUES (?,?)");
pstmt.setInt(1, 100);
pstmt.setAsciiStream(2, new FileInputStream(file), (int) file.length());
```


#### CallableStatement

Using **CallableStatement** objects is much like using **PreparedStatement** objects. You must bind values to all the parameters before executing the statement, or you will receive an **SQLException**.

The CallableStatement object can use all the three - **IN**, **OUT**, **INOUT**.
When you use **OUT** and **INOUT** parameters, you must employ an additional **CallableStatement** method, `registerOutParameter()`.

```java
CallableStatement cstmt = conn.prepareCall("{call getSalary(?, ?)}"); // set id and get salary
cstmt.setInt(1, 2);
cstmt.registerOutParameter(2, java.sql.Types.INTEGER);
cstmt.execute();
int salary = cstmt.getInt(2);
```


#### JDBC SQL Escape Syntax

The general SQL escape syntax format is as follows: `{keyword 'parameters'}`.
- **call** keyword is used to call the stored procedures
`{call my_procedure(?)}` and function `{? = call my_function(?)}`
- **fn** keyword represents scalar functions used in a DBMS
`{fn length('Hello World')}` - returns 11
- **d**, **t**, **ts** keywords help identify date, time, and timestamp literals. This escape syntax tells the driver to render the date or time in the target database's format.
`stmt.executeUpdate("INSERT INTO PEOPLE VALUES (100, 'Zara', {d '2001-12-16'})");`
- **escape** keyword identifies the escape character used in LIKE clauses
`stmt.execute("SELECT symbol FROM Symbols WHERE symbol LIKE '\\%' {escape '\\'}");`



### ResultSet

A **ResultSet** object maintains a **cursor** that points to the current row in the result set. It also contains dozens of methods for getting the data of the current row. There are two types of these methods: `getXXX(String columnName)` and `getXXX(int columnIndex)`.

A default **ResultSet** object is not updatable and has a cursor that moves forward only.

In **MySQL**, by default, **ResultSets** are completely retrieved and stored in memory. But you can tell the driver to stream the results back one row at a time using `stmt.setFetchSize(Integer.MIN_VALUE);`

SQL's use of **NULL** values and Java's use of **null** are different concepts. Use wrapper classes for primitive data types, and use the **ResultSet**'s `wasNull()` method to test whether the wrapper class variable that received the value returned by the `getXXX()` method should be set to **null**.

```java
int id = rs.getInt(1);
if (rs.wasNull()) id = 0;
```

To create statements with desired **ResultSet**:
- `createStatement(int RSType, int RSConcurrency, int RSHoldability);`
- `prepareStatement(String SQL, int RSType, int RSConcurrency, int RSHoldability);`
- `prepareCall(String sql, int RSType, int RSConcurrency, int RSHoldability);`


- **Type**
	- ***TYPE_FORWARD_ONLY*** can only be navigated forward
	- ***TYPE_SCROLL_INSENSITIVE*** can be navigated both forward/backwards and jump to a relative/absolute position. The **ResultSet** is insensitive to changes in the underlying data source.
	- ***TYPE_SCROLL_SENSITIVE*** the same but the **ResultSet** is sensitive to changes in the underlying data source. If a record in the **ResultSet** is changed in the database by another thread, it will be reflected in already opened **ResulsSet**.
	- Navigation Methods
		- `absolute, afterLast, beforeFirst, first, last, next, previous, relative`
		- `getRow, getType, isAfterLast, isBeforeFirst, isFirst`
		- `refreshRow` refreshes the column values of that row with the latest values from the database
- **Concurrency**
	- ***CONCUR_READ_ONLY*** a ResultSet can only be read
	- ***CONCUR_UPDATABLE*** a ResultSet can be both read and updated
- **Holdability**
	- ***CLOSE_CURSORS_OVER_COMMIT*** indicating that open **ResultSet** will be closed when the current transaction is committed `connection.commit()`
	- ***HOLD_CURSORS_OVER_COMMIT*** indicating that open **ResultSet** will remain open when the current transaction is committed `connection.commit()`



#### Manipulating ResultSet

If a **ResultSet** is *updatable* we can:

- update the columns of each its row by using the many `updateXXX()` methods and then `updateRow()` to apply changes
- insert a new row by calling `moveToInsertRow()` then set the row column values and finally call `insertRow()`
- delete a row by `deleteRow()`
- refresh a data by `refreshRow()`


A **ResultSet** object is automatically closed when the **Statement** object that generated it is closed, re-executed, or used to retrieve the next result from a sequence of multiple results.



### Batch

You can batch both SQL inserts, updates and deletes. It does not make sense to batch select statements. Also we should disable **auto-commit**.

- `addBatch()` method of **Statement**, **PreparedStatement**, and **CallableStatement** is used to add individual statements to the batch
- `executeBatch()` is used to start the execution of all the statements grouped together
- `clearBatch()` method removes all the statements you added before

There are two ways to execute batch updates:

- Using a Statement
	```java
    statement = connection.createStatement();
    statement.addBatch("update people set firstname='John' where id=123");
    statement.addBatch("update people set firstname='Eric' where id=456");
    int[] recordsAffected = statement.executeBatch();
    ```
- Using a PreparedStatement
	```java
	preparedStatement = connection.prepareStatement(sql);
    preparedStatement.setString(1, "Gary");
    preparedStatement.addBatch();
    preparedStatement.setString(1, "Stan");
    preparedStatement.addBatch();
    int[] affectedRecords = preparedStatement.executeBatch();
    ```



### Transaction

JDBC Connection has **auto-commit** mode by default, thus every SQL statement is committed to the database upon its completion.

Reason to disable auto commit:
- To increase performance
- To maintain the integrity of business processes (a group of SQL queries should be commmited or rollbacked as one)
- To use distributed transactions

`connection.setAutoCommit(false);`
`connection.commit();`
`connection.rollback();`

We can use a savepoint to define a **logical rollback** point within a transaction. If an error occurs past a savepoint, you can use the `rollback(String savepointName)` method to undo either all the changes or only the changes made after the savepoint.

- `Savepoint setSavepoint(String name)` defines a new savepoint
- `void releaseSavepoint(Savepoint name)` deletes a savepoint

In this case, none of the below INSERT statement would success and everything would be rolled back.

```java
try {
   Savepoint savepoint1 = conn.setSavepoint("Savepoint1");
   stmt.executeUpdate("INSERT INTO Employees VALUES (11, 'Rita')");
   //Submit a malformed SQL statement that breaks
   stmt.executeUpdate("INSERTED IN Employees VALUES (107, 'Sita')");

   conn.commit();
} catch(SQLException e){
   conn.rollback(savepoint1);
}
```


### MetaData

#### ResultSetMetaData

- `getColumnCount()`
- `getColumnName(index)`
- `getColumnTypeName(index)`

#### DatabaseMetaData

- `getDriverName(); getUserName(); getDatabaseProductName(); getTables(...)`
- `DatabaseMetaData.supportsResultSetType(int type)` checks whether the given RS type is supported or not
- `DatabaseMetaData.supportsResultSetConcurrency(int mode)` checks whether the given RS mode is supported or not
- `DatabaseMetaData.supportsResultSetHoldability(int h)` checks whether the given RS mode is supported or not
- `DatabaseMetaData.supportsBatchUpdates()` checks wheather a db supports batch update processing.


### RowSet

The interface that adds support to the JDBC API for the JavaBeans component model. Introduced since JDK 5.

It is the wrapper of **ResultSet**. It holds tabular data like **ResultSet** but it is easy and flexible to use. Also **RowSet** is *scrollable* and *updatable* by default.


The implementation classes of **RowSet**:

- Connected
	- **JdbcRowSet** basically acts as a wrapper around the **ResultSet** object with some additional functionality
- Disconnected
	- **CachedRowSet** extends **RowSet** and is a disconnected **RowSet** that acts as a container for database records and caches them in memory
	- **WebRowSet** providing all the features of **CachedRowSet** and can read and write XML document
	- **JoinRowSet** has all the capabilities of **WebRowSet** and **CachedRowSet**. Also can perform a *SQL JOIN* operation without connecting to a data source
	- **FilteredRowSet** has all the capabilities of **WebRowSet** as well as **CachedRowSet**. Also we can apply filtering criteria to fetch selected rows from the data source so that we can work with the relevant data


```java
JdbcRowSet rowSet = RowSetProvider.newFactory().createJdbcRowSet();
rowSet.setUrl("jdbc:mysql://localhost:3306/practice");
rowSet.setUsername("root");
rowSet.setPassword("root");

rowSet.setCommand("select * from emp400");
rowSet.execute();

while (rowSet.next()) {
	System.out.println("Id: " + rowSet.getString(1));
	System.out.println("Name: " + rowSet.getString(2));
}
```

To perform event handling with **JdbcRowSet**, you need to add the instance of **RowSetListener** in the `addRowSetListener` method:

- `cursorMoved(RowSetEvent event)`
- `rowChanged(RowSetEvent event)`
- `rowSetChanged(RowSetEvent event)`


### Exceptions

- `java.sql.SQLException` – this is the base exception class for JDBC exceptions.
- `java.sql.BatchUpdateException` – this exception is thrown when Batch operation fails, but it depends on the JDBC driver whether they throw this exception or the base SQLException.
- `java.sql.SQLWarning` – for warning messages in SQL operations.
- `java.sql.DataTruncation` – when a data values is unexpectedly truncated for reasons other than its having exceeded MaxFieldSize.



### Best Practices

1. Database resources are heavy, so make sure you close it as soon as you are done with it. **Connection**, **Statement**, **ResultSet** and all other JDBC objects have `close()` method defined to close them.
2. Always close the **ResultSet**, **Statement** and **Connection** explicitly in the code, because if you are working in connection pooling environment, the connection might be returned to the pool leaving open result sets and statement objects resulting in resource leak.
3. Close the resources in the **finally block** to make sure they are closed even in case of exception scenarios.
4. Use **batch processing** for bulk operations of similar kind.
5. Always use **PreparedStatement** over **Statement** to avoid SQL Injection and get pre-compilation and caching benefits of **PreparedStatemen**t.
6. If you are retrieving bulk data into result set, setting an optimal value for `fetchSize()` helps in getting good performance.
7. If you are creating database connections in a web application, try to use JDBC **DataSource** resources using JNDI context for re-using the connections.
8. Try to use disconnected **RowSet** when you need to work with **ResultSet** for a long time.