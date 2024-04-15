## ABOUT THE DATABASE
The PostgreSQL database, runs on the server and is setup to have the following:
- Secure connections with SSL
- Logging and monitoring
- Backup (and restore)
-  High Availability, Load Balancing, and Replication

### Initial setup (for whoever is managing the server)
- Login as ```postgres``` user
```
sudo -i -u postgres
psql -U postgres
```

- Create DB credentials
To ensure that the secrets required by the developers are created in the PostgreSQL database, you will need to perform the following steps:

    -   Create the Database:
```
CREATE DATABASE mydatabase;
```

    -   Create the User:
```
CREATE USER myuser WITH PASSWORD 'mypassword';
```

    -   Assign Privileges:
After creating the user, you need to give them the necessary privileges on the database they will be working with:
```
GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;
```

### A Peek into the database
**outside ```psql```**
- To list all databases within psql:
```
sudo -u postgres psql -c "\l"
```

- Viewing Users (Roles)
To list all users (also known as roles) in PostgreSQL within psql:
```
sudo -u postgres psql -c "\du"
```

**within ```psql```**
```\l``` to list all databases.
```\du``` to list all roles/users.

**connecting to the DB**
-   For Localhost: If you're running commands directly on the server, the database ```host``` is ```localhost```.
-   For Remote Connections: The ```host``` would be the ```IP address``` or ```hostname``` of the server where PostgreSQL is installed. You can find this by checking your application's database configuration file or by asking your system administrator.

**View Connection Details for Current Session**: If you're already connected to PostgreSQL via psql, you can view connection details for the current session with the command:
```
SELECT current_database(), current_user, inet_server_addr(), inet_server_port();
```
This SQL query returns the name of the current database, the name of the current user, the server IP address, and the server port number.

**Note**: 
If ```inet_server_addr()``` is returning an empty result, it's commonly because the connection to the PostgreSQL server is being made via a Unix domain socket rather than TCP/IP.
In such cases, the server is typically running on the same machine as the client, and you can use ```localhost``` or ```127.0.0.1``` as the host when configuring your database connection settings.

## WHAT SHOULD YOU, AS A DEV
In your .env file, set values for the following variable names
```
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
DATABASE_URL=
```
Show this file (.env) to the server admin, so they create these credentials for you on the database.

[Official Docs](https://www.postgresql.org/docs/current/admin.html)