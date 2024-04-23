

## Resources 

### [Template Databases](https://gajus.com/blog/setting-up-postgre-sql-for-running-integration-tests#template-databases)

A [template databases](https://www.postgresql.org/docs/current/manage-ag-templatedbs.html) is a database that serves as a template for creating new databases. When you create a new database from a template database, the new database has the same schema as the template database. The steps to create a new database from a template database are 
as follows:

1. Create a template database (`ALTER DATABASE <database_name> is_template=true;`)
2. Create a new database from the template database (`CREATE DATABASE <new_database_name> TEMPLATE <template_database_name>;`)

The key advantage of using template databases is that you do not need to mess with managing multiple PostgreSQL instances. You can create copy databases and have each test run in isolation.

However, on its own, template databases are not fast enough for our use case. The time it takes to create a new database from a template database is still too high for running thousands of tests:

```
postgres=# CREATE DATABASE foo TEMPLATE contra;CREATE DATABASETime: 1999.758 ms (00:02.000)
```

This is where the [memory mounting](https://gajus.com/blog/setting-up-postgre-sql-for-running-integration-tests#mounting-a-memory-disk)comes in.

The other limitation of template databases to be aware of is that no other sessions can be connected to the source database while it is being copied. `CREATE DATABASE` will fail if any other connection exists when it starts; during the copy operation, new connections to the source database are prevented. It is an easy enough limitation to work around using a [mutex pattern](https://en.wikipedia.org/wiki/Lock_(computer_science)), but it is something to be aware of.

### [Mounting a memory disk](https://gajus.com/blog/setting-up-postgre-sql-for-running-integration-tests#mounting-a-memory-disk)

The final piece of the puzzle is mounting a memory disk. By mounting a memory disk, and creating the template database on the memory disk, we can significantly reduce the overhead of creating a new database.

I will talk about how to mount a memory disk in the next section, but first, let's see how much of a difference it makes.

```
postgres=# CREATE DATABASE bar TEMPLATE contra;CREATE DATABASETime: 87.168 ms
```

This is a significant improvement and makes the approach viable for our use case.

Needless to say, this approach is not without its drawbacks. The data is stored in memory, which means that it is not persistent. If the database crashes or the server restarts, the data is lost. However, for running tests, this is not a problem. The data is recreated from the template database each time a new database is created.

### [Using Docker container with a memory disk](https://gajus.com/blog/setting-up-postgre-sql-for-running-integration-tests#using-docker-container-with-a-memory-disk)

The approach we settled on was to use a Docker container with a memory disk. Here is how you can set it up:

```
$ docker run \  -p 5435:5432 \  --tmpfs /var/lib/pg/data \  -e PGDATA=/var/lib/pg/data \  -e POSTGRES_PASSWORD=postgres \  --name contra-database \  --rm \  postgres:14
```

In the above command, we are creating a Docker container with a memory disk mounted at `/var/lib/pg/data`. We are also setting the `PGDATA` environment variable to `/var/lib/pg/data` to ensure that PostgreSQL uses the memory disk for data storage. The end result is that the underlying data is stored in memory, which significantly reduces the overhead of creating a new database.

### [Managing test databases](https://gajus.com/blog/setting-up-postgre-sql-for-running-integration-tests#managing-test-databases)

The basic idea is to create a template database before running the tests and then create a new database from the template database for each test. Here is a simplified version of how you can manage test databases:

```
import {  createPool,  sql,  stringifyDsn,} from 'slonik'; type TestDatabase = {  destroy: () => Promise<void>;  getConnectionUri: () => string;  name: () => string;}; const createTestDatabasePooler = async (connectionUrl: string) => {  const pool = await createPool(connectionUrl, {    connectionTimeout: 5_000,    // This ensures that we don't attempt to create multiple databases in parallel.    maximumPoolSize: 1,  });   const createTestDatabase = async (    templateName: string,  ): Promise<TestDatabase> => {    const database = 'test_' + uid();     await pool.query(sql.typeAlias('void')`      CREATE DATABASE ${sql.identifier([database])}      TEMPLATE ${sql.identifier([templateName])}    `);     return {      destroy: async () => {        await pool.query(sql.typeAlias('void')`          DROP DATABASE ${sql.identifier([database])}        `);      },      getConnectionUri: () => {        return stringifyDsn({          ...parseDsn(connectionUrl),          databaseName: database,          password: 'unsafe_password',          username: 'contra_api',        });      },      name: () => {        return database;      },    };  };   return () => {    return createTestDatabase('contra_template');  };}; const getTestDatabase = await createTestDatabasePooler();
```

At this point, you can use `getTestDatabase` to create a new database for each test. The `destroy`method can be used to clean up the database after the test has run.