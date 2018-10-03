# PyEXASOL best practices

Exasol is distributed columnar MPP database. It easily runs on dozens of nodes and hundreds of CPU cores. It is capable of processing billions of rows in a few seconds. Every SQL query is running in parallel across the whole cluster and uses all available resources. There is a big difference between Exasol (OLAP) and row-based OLTP databases (e.g. MySQL, PostgreSQL).

The single best piece of advice is to **limit usage of Python** only to **generation of SQL queries** and **retrieval of small result sets**.

If you want to use Python for actual data processing, consider [Exasol eUDF Scripting framework](https://www.exasol.com/portal/display/SOL/UDFs+and+In-Database+Analytics). More details are available in [Exasol User Manual](https://www.exasol.com/support/secure/attachment/56160/EXASOL_User_Manual-6.0.5-en.pdf). It allows you to run multiple instances of Python scripts directly into Exasol cluster and take full advantage of parallelism and data locality.

If you absolutely have to move big amounts of data to and from Exasol (e.g. for Jupiter Notebook), please keep reading.

## Consider setting `compression=True`

```python
C = pyexasol.connect(... , compression=True)
```

Network is the main bottleneck in majority of cases. `Compression` flag enables transparent zlib compression for all client-server communication, including common fetching and fast [HTTP transport](/docs/HTTP_TRANSPORT.md). It may improve overall performance by factor 4-8x.

Enable compression for local laptops connecting to Exasol over wireless network. Disable `compression` if you transfer data within the same data centre over fast network.

## Use HTTP transport for big volumes of data

```python
pd = C.export_to_pandas('SELECT * FROM table')
C.export_to_file('my_file.csv', 'SELECT * FROM table')

C.import_from_pandas(pd, 'table')
C.import_from_file('my_file.csv', 'table')
```

It is okay to use common fetching for small data sets up to 1M of records.

For anything bigger than that you should always consider [HTTP transport](/docs/HTTP_TRANSPORT.md) (`export_*` and `import_*` functions). Internally it uses native `IMPORT` and `EXPORT` commands and transfers data using common CSV format. Most of Big Data tool and libraries are capable of parsing CSV using fast C implementation.

The best way to achieve good performance in Python is to avoid creation of Python objects altogether. You may stream raw data directly from Exasol to external tools using pipes opened in binary mode.

## Prefer iterator syntax to fetch result sets rather than `fetch*` functions

Iterator is very easy to use, and it has low memory footprint.

```python
stmt = C.execute('SELECT * FROM table')

for row in stmt:
    print(row)
```

Please note that `fetchall()`, `fetchmany()` and `fetchcol()` may run out of memory in case of very big data sets. Those functions are also harder to use because you have to check for `None` and `[]` results.

## Always specify full connection string for Exasol cluster

Unlike standard WebSocket Python driver, PyEXASOL supports full connection strings and node redundancy. For example, connection string `myexasol1..5:8563` will be unpacked as:

```
myexasol1:8563
myexasol2:8563
myexasol3:8563
myexasol4:8563
myexasol5:8563
```

PyEXASOL tries to connect to random node from this list. If it fails, it tries to connect to another random node. The main benefits of this approach are:

- Multiple connections are evenly distributed across the whole cluster;
- If one or more nodes are down, but the cluster is still operational due to redundancy, users will be able to connect using PyEXASOL without any problems or random error messages;

## Never use INSERT statement to insert raw values in SQL, always use IMPORT instead

```python
def data_generator():
    for id in range(10):
        yield (id, 'abc')

C.import_from_iterable(data_generator, 'table')
```

Exasol keeps audit logs for all executed SQL statements. Passing data directly with SQL text bloats audit log, may expose sensitive information and has high overhead of serialization-deserialization.

You should always use IMPORT even for relatively small data sets. Please consider [`import_from_iterable`](/docs/REFERENCE.md#import_from_iterable) function.

## Consider faster JSON-parsing libraries

PyEXASOL defaults to standard [`json`](https://docs.python.org/3/library/json.html) library for best compatibility. It is sufficient for majority of use-cases. However, if you are unhappy with HTTP transport and you wish to load large amounts of data using standard fetching, we highly recommend trying faster JSON libraries.

#### json_lib=[`rapidjson`](https://github.com/python-rapidjson/python-rapidjson)
```
pip install pyexasol[rapidjson]
```
Rapidjson provides significant performance boost and it is well maintained by creators. PyEXASOL defaults to `number_mode=NM_NATIVE`. Exasol server wraps big decimals with quotes and returns as strings, so it should be a safe option.

#### json_lib=[`ujson`](https://github.com/esnme/ultrajson)
```
pip install pyexasol[ujson]
```
Ujson provides the best performance in our internal tests, but it is abandoned by creators.

You may try any other json library. All you need to do is to overload `_init_json()` method in `ExaConnection`.
