---
description: Using Configuration Sources
---

# Using ormconfig.json

## Creating a new connection from the configuration file

Most of the times you want to store your connection options in a separate configuration file. It makes it convenient and easy to manage. TypeORM supports multiple configuration sources. You only need to create a `ormconfig.[format]` file in the root directory of your application \(near `package.json`\), put your configuration there and in your app call `createConnection()` without any configuration passed:

```typescript
import {createConnection} from "typeorm";

// createConnection method will automatically read connection options
// from your ormconfig file or environment variables
const connection = await createConnection();
```

Supported ormconfig file formats are: `.json`, `.js`, `.ts`, `.env`, `.yml` and `.xml`.

## Using `ormconfig.json`

Create `ormconfig.json` in the project root \(near `package.json`\). It should have the following content:

```javascript
{
   "type": "mysql",
   "host": "localhost",
   "port": 3306,
   "username": "test",
   "password": "test",
   "database": "test"
}
```

You can specify any other options from [ConnectionOptions](connection-options.md).

If you want to create multiple connections then simply create multiple connections in a single array:

```javascript
[{
   "name": "default",
   "type": "mysql",
   "host": "localhost",
   "port": 3306,
   "username": "test",
   "password": "test",
   "database": "test"
}, {
   "name": "second-connection",
   "type": "mysql",
   "host": "localhost",
   "port": 3306,
   "username": "test",
   "password": "test",
   "database": "test"
}]
```

## Using `ormconfig.js`

Create `ormconfig.js` in the project root \(near `package.json`\). It should have following content:

```javascript
module.exports = {
   "type": "mysql",
   "host": "localhost",
   "port": 3306,
   "username": "test",
   "password": "test",
   "database": "test"
}
```

Alternatively, you may use the ECMAScript module format if your environment supports it:

```javascript
export default {
   "type": "mysql",
   "host": "localhost",
   "port": 3306,
   "username": "test",
   "password": "test",
   "database": "test"
}
```

You can specify any other options from [ConnectionOptions](connection-options.md). If you want to create multiple connections then simply create multiple connections in a single array and return it.

## Using environment variables

Create `.env` or `ormconfig.env` in the project root \(near `package.json`\). It should have the following content:

```text
TYPEORM_CONNECTION = mysql
TYPEORM_HOST = localhost
TYPEORM_USERNAME = root
TYPEORM_PASSWORD = admin
TYPEORM_DATABASE = test
TYPEORM_PORT = 3000
TYPEORM_SYNCHRONIZE = true
TYPEORM_LOGGING = true
TYPEORM_ENTITIES = entity/*.js,modules/**/entity/*.js
```

List of available env variables you can set:

* TYPEORM\_CACHE
* TYPEORM\_CACHE\_ALWAYS\_ENABLED
* TYPEORM\_CACHE\_DURATION
* TYPEORM\_CACHE\_OPTIONS
* TYPEORM\_CONNECTION
* TYPEORM\_DATABASE
* TYPEORM\_DEBUG
* TYPEORM\_DRIVER\_EXTRA
* TYPEORM\_DROP\_SCHEMA
* TYPEORM\_ENTITIES
* TYPEORM\_ENTITIES\_DIR
* TYPEORM\_ENTITY\_PREFIX
* TYPEORM\_HOST
* TYPEORM\_LOGGER
* TYPEORM\_LOGGING
* TYPEORM\_MAX\_QUERY\_EXECUTION\_TIME
* TYPEORM\_MIGRATIONS
* TYPEORM\_MIGRATIONS\_DIR
* TYPEORM\_MIGRATIONS\_RUN
* TYPEORM\_MIGRATIONS\_TABLE\_NAME
* TYPEORM\_PASSWORD
* TYPEORM\_PORT
* TYPEORM\_SCHEMA
* TYPEORM\_SID
* TYPEORM\_SUBSCRIBERS
* TYPEORM\_SUBSCRIBERS\_DIR
* TYPEORM\_SYNCHRONIZE
* TYPEORM\_URL
* TYPEORM\_USERNAME
* TYPEORM\_UUID\_EXTENSION

`TYPEORM_CACHE` should be boolean or string of cache type

`ormconfig.env` should be used only during development. On production you can set all these values in real ENVIRONMENT VARIABLES.

You cannot define multiple connections using an `env` file or environment variables. If your app has multiple connections then use alternative configuration storage format.

If you need to pass a driver-specific option, e.g. `charset` for MySQL, you could use the `TYPEORM_DRIVER_EXTRA` variable in JSON format, e.g.

```text
TYPEORM_DRIVER_EXTRA='{"charset": "utf8mb4"}'
```

## Using `ormconfig.yml`

Create `ormconfig.yml` in the project root \(near `package.json`\). It should have the following content:

```yaml
default: # default connection
    host: "localhost"
    port: 3306
    username: "test"
    password: "test"
    database: "test"

second-connection: # other connection
    host: "localhost"
    port: 3306
    username: "test"
    password: "test"
    database: "test2"
```

You can use any connection options available.

## Using `ormconfig.xml`

Create `ormconfig.xml` in the project root \(near `package.json`\). It should have the following content:

```markup
<connections>
    <connection type="mysql" name="default">
        <host>localhost</host>
        <username>root</username>
        <password>admin</password>
        <database>test</database>
        <port>3000</port>
        <logging>true</logging>
    </connection>
    <connection type="mysql" name="second-connection">
        <host>localhost</host>
        <username>root</username>
        <password>admin</password>
        <database>test2</database>
        <port>3000</port>
        <logging>true</logging>
    </connection>
</connections>
```

You can use any connection options available.

## Which configuration file is used by Typeorm

Sometimes, you may want to use multiple configurations using different formats. When calling `getConnectionOptions()` or attempting to use `createConnection()` without the connection options, Typeorm will attempt to load the configurations, in this order:

1. From the environment variables. Typeorm will attempt to load the `.env` file using dotEnv if it exists. If the environment

   variables `TYPEORM_CONNECTION` or `TYPEORM_URL` are set, Typeorm will use this method.

2. From the `ormconfig.env`.
3. From the other `ormconfig.[format]` files, in this order: `[js, ts, json, yml, yaml, xml]`.

Note that Typeorm will use the first valid method found and will not load the others. For example, Typeorm will not load the `ormconfig.[format]` files if the configuration was found in the environment.

## Overriding options defined in ormconfig

Sometimes you want to override values defined in your ormconfig file, or you might want to append some TypeScript / JavaScript logic to your configuration.

In such cases you can load options from ormconfig and get `ConnectionOptions` built, then you can do whatever you want with those options, before passing them to `createConnection` function:

```typescript
// read connection options from ormconfig file (or ENV variables)
const connectionOptions = await getConnectionOptions();

// do something with connectionOptions,
// for example append a custom naming strategy or a custom logger
Object.assign(connectionOptions, { namingStrategy: new MyNamingStrategy() });

// create a connection using modified connection options
const connection = await createConnection(connectionOptions);
```

