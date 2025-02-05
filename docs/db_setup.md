# MySQL Database Deployment Manual

## 1. Overview

This document explains how to quickly deploy the `loxilb_db` database and related tables. If an existing MySQL server is available, use that server. If a new MySQL instance needs to be set up, follow the additional instructions provided. This database is essential for the userservice.

---

## 2. Preparing the MySQL Environment

### 2.1 Using an Existing MySQL Server

If a MySQL server is already running, verify the following information:

- MySQL server address (`hostname` or `IP`)
- Accessible account information (e.g., `root` user or a user with appropriate privileges)

### 2.2 Setting Up a New MySQL Server (Optional)

If MySQL is not installed, choose one of the following installation methods:

- **Linux**: `apt install mysql-server` or `yum install mysql-server`

After installation, start MySQL:

```bash
sudo systemctl start mysql  # Linux
```

By default, you can connect using the `root` user:

```bash
mysql -u root -p
```

### 2.3 Deploying MySQL Using Docker (Easy)

To deploy MySQL in a Docker environment, follow these steps:

1. Run the MySQL container:

```
docker run --name mysql-container -e MYSQL_ROOT_PASSWORD=secure_password -e MYSQL_DATABASE=loxilb_db -p 3306:3306 -d mysql:latest
```

1. Check if the container is running correctly:

```
docker ps
```

1. Access the MySQL container:

```
docker exec -it mysql-container mysql -u root -p
```

---

## 3. Creating the Database and Tables

After connecting to MySQL, execute the following SQL commands to create the database and tables.

### 3.1 Creating the Database

```sql
CREATE DATABASE loxilb_db;
```

### 3.2 Using the Database

```sql
USE loxilb_db;
```

### 3.3 Creating the `users` Table

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role VARCHAR(255) NOT NULL DEFAULT 'viewer'
);
```

### 3.4 Creating the `token` Table

```sql
CREATE TABLE token (
    id INT AUTO_INCREMENT PRIMARY KEY,
    token_value VARCHAR(512) NOT NULL,
    username VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role VARCHAR(255) NOT NULL
);
```

### 3.5 Complete SQL Script

```sql
-- Create the database
CREATE DATABASE loxilb_db;

-- Use the database
USE loxilb_db;

-- Create the users table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role VARCHAR(255) NOT NULL DEFAULT 'viewer'
);

-- Create the token table
CREATE TABLE token (
    id INT AUTO_INCREMENT PRIMARY KEY,
    token_value VARCHAR(512) NOT NULL,
    username VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role VARCHAR(255) NOT NULL
);
```

---

## 4. Verifying the Database

To verify that the tables have been created correctly, execute the following command:

```sql
SHOW TABLES;
```

To check the structure of each table:

```sql
DESC users;
DESC token;
```

---

## 5. Troubleshooting

### 5.1 MySQL Connection Errors

- If you encounter the error `ERROR 1045 (28000): Access denied for user`, check the user account privileges.
- Log in using `mysql -u root -p`, then create and grant privileges to the required account if necessary.

### 5.2 Database Creation Issues

- Run `SHOW DATABASES;` to check if the database exists.
- If necessary, drop the existing database and recreate it using `DROP DATABASE loxilb_db;` and `CREATE DATABASE loxilb_db;`.

---

## 6. Conclusion

The `loxilb_db` database and `users` and `token` tables are now successfully set up. If additional data input or application integration is required, execute the appropriate queries to manage them.
