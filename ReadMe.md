Building node.js app with neo4j
===

# JavaScrip Driver

## Objectives

- Install the Neo4j JavaScript Driver;
- Build a connection String;
- Create an instance of the Driver;
- Verify that the Driver has successfuly connected to Neo4j;

## About the driver

The **driver** object allows to execute Cypher statements agains a Neo4j database. It is topology independent and can be run against a Neo4j Cluster or a Single DBMS.

In this app we use [*java script driver*](https://neo4j.com/developer/javascript). *Only one instance of the driver* should be created in the application per one Neo4j cluster or DBMS, which can be than shared accross the application.

### Installation

```bash
npm install --save neo4j-driver
```
For the more information about *neo4j driver* see the [API documentation](https://neo4j.com/docs/api/javascript-driver/current/).

### Creating a driver instance

```JavaScript
// import the dependency
import neo4j from 'neo4j-driver';

// create a new driver instance
const driver = neo4j.driver(
    url: process.env.NEO4J_URI,
    authToken: neo4j.auth.basic(
        process.env.NEO4J_USERNAME,
        process.env.NEO4J_PASSWORD
    )
)
```

The `driver` function accepts *three arguments*:

- `url` connection string;
- `authToken` the authentication credentials, as defined by `neo4j.auth` function:
    - **basic**: `function(username: string, password: string, realm: ?string)`
    - **kerbros**: `function(base64EncodedTicket: string)`;
    - **bearer**: `function(base64EncodedTicket: string)`;
    - **custom**: `function(principal: string, credentials: string, realm: string, scheme: string, parameters: ?object)`;
- `cinfig`: additional configuration of the driver;

The snippet of the code represents *unencrypted connection* to Neo4j server. The driver than attempts to authenticate against the server with vlaues from the `.env` file.

### Verifying Connectivity

The *driver* instance provides *method* for the *connectivity verification*:

```JavaScript
await driver.verifyConnectivity();
```

The code above returns a *promise* which *resolves* if the connection is verified, and *rejects with error* (`Neo.ClientError.Security.Unauthorized`) if a connection could not be made.

### Connection string format

![connection string](https://cdn.graphacademy.neo4j.com/assets/img/courses/shared/connection-string.png)

# Interacting with database

## Outlook

- Open the *session* and *execut* a unit of work;
- Run *read* and *write* queries through the *driver*;
- Consume the *results* returned from Neo4j;
- *Handle* potential *errors* therown by the *driver*;

## Sessions and transactions

### Sessions

The *session* is opened through the *driver*. A *session* and a *database connection* are not the same.

A *driver* connected to the *database* opens up *multiple TCP connections* that can be *borrowed* by the *session*. Meaning that *query* may be sent over *multiple connections* and the reusults may be received by the *driver* over *multiple connections*.

*Sessions* should be considered a client-side abstraction for grouping units of work, handling *underlying connections*.

#### Openning the session

```JavaScript
const session = driver.session();
```

Additional arguments might be given to the *session*:

- A mode of the session, either `READ` or `WRITE`;
- A database name. If not supplied the default database will be used;

```JavaScript
import neo4j from 'neo4j-driver';

const session = driver.session({
    defaultAccessMode: driver.session.WRITE,
    database: 'people'
});
```

The *default database* is configured in the `dbms.default_database` in `neo4j.conf` (default value is `neo4j`).

The *default access mode* is `WRITE`, but it can be overwritten by calling `readTransaction()` or `writeTransaction()` methods within the section.

### Transactions

A *transaction* is a *unit of work* run against the database. A single *session* can contain *multiple transactions*.

There are *three types* of transactions:

- Auto-commit Transactions;
- Read Transactions;
- Write Transactions;

#### Auto-commit transactions

An *Auto-commit transaction* is immediately executed and acknowledged. It can be run by calling `run()` method on the *session object*, passing a *Cypher statement*, and optionally an object containing a set of parameters.

```JavaScript
const session = driver.session();

const res = await session.run(query, params);
```

Upon an error, driver will not retry a querry with `session.run()`. For this reason it shouldn't be used in production.

#### Read transaction

`session.readTransaction()` is meant to *read data* from the database:

```JavaScript
const res = await session.readTransaction(tx => {
    return tx.run(
        `MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
         WHERE m.title = $title
         RETURN p.name AS name
         LIMIT 10;
        `,
        { title: 'Arthur' }
    )
});
```

#### Write transaction

Execute `writeTransaction()` if you need to write the data into the database.

```JavaScript
const res = session.writeTransaction(tx => {
    return tx.run(
        `CREATE(p:Person { name: $name });
        `,
        { name: 'Michael' }
    )
});
```

In case of error, both *read* and *write* transactions, if there is a connectivity error for example, will retry the unit of work.

#### Creating transactions manually

Transaction can be *explicitly created* using `session.beginTransaction()` method. In contrast to `readTransaction()` and `writeTransaction()` transaction must be *commited or rolled back manually* with `commit()` or `rollback()` function. Once finished, the session must be closed to release any database connections held by the session.

```JavaScript
const session = driver.session({
    defaultAccessMode: session.WRITE
});

const tx = session.beginTransaction();

try {
    tx.run(query, params);
    tx.commit();
} catch (e) {
    tx.rollback();
};

await session.close();

```

### Working example

```JavaScript
async function createPerson (name) {
    // create a session for 'people' database
    const session = driver.session({
        // run in a WRITE mode
        defaultAccessMode: session.WRITE,
        // Run queries against people database
        database: 'people'
    });

    // Create a node with a write transaction
    const res = await session.writeTransaction(tx => {
        return tx.run(
            'CREATE (p:Person { name: $name }) RETURN p', 
            { name }
        );
    });

    // Get the p value from the first record
    const p = res.records[0].get('p');

    // Close the session
    await session.close();

    // Return the properties of the node
    return p.properties;
}
```

