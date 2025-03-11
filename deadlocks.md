# **Deadlock Example**


## **📌 How Deadlocks Occur (Money Transfer Scenario)**
Imagine **two transactions transferring money between two accounts**, but in **reverse order**, leading to a deadlock:

### **🧑‍💻 Transaction A (Wants to transfers $100 from Account 1 to Account 2)**
```sql
START TRANSACTION;
UPDATE Accounts SET balance = balance - 100 WHERE account_id = 1;  -- Locks account_id = 1
-- Some processing time (Transaction A does NOT commit yet)
```

### **🧑‍💻 Transaction B (Wants to transfer $200 from Account 2 to Account 1 at the same time)**
```sql
START TRANSACTION;
UPDATE Accounts SET balance = balance - 200 WHERE account_id = 2;  -- Locks account_id = 2
-- Some processing time (Transaction B does NOT commit yet)
```

### **🧑‍💻 Transaction A (Continues, but is now blocked)**
```sql
UPDATE Accounts SET balance = balance + 100 WHERE account_id = 2;  -- 🚨 BLOCKED! Transaction B has this lock.
```

### **🧑‍💻 Transaction B (Continues, but is also blocked)**
```sql
UPDATE Accounts SET balance = balance + 200 WHERE account_id = 1;  -- 🚨 BLOCKED! Transaction A has this lock.
```

🚨 **Deadlock occurs!** MySQL will automatically detect and resolve the deadlock by rolling back **one of the transactions**.

### **Test the scenario** in MySQL Workbench by opening two connections to represent two sessions (transaction A and B respectively).

---

## **🛠️How to Prevent Deadlocks?**

1️⃣ **Access tables in a consistent order** → Always update `account_id = 1` first, then `account_id = 2`.

2️⃣ **Use `LOCK TABLES`** → Prevents concurrent access to multiple rows.

3️⃣ **Use `FOR UPDATE`** → Ensures rows are locked before modifying them.

4️⃣ **Set a timeout** → Prevents waiting indefinitely (`SET innodb_lock_wait_timeout = 5;`).

---
## **📌 Deadlocks Exercise using MySQL and Java**
Try out the following examples on your own computer. You can use another databaser server and possibly another programming language of your own choice. The example os based on MySQL and Java JDBC.

## **📝 SQL Script: Create Database and Table**
```sql
CREATE DATABASE IF NOT EXISTS bank;
USE bank;

CREATE TABLE IF NOT EXISTS Accounts (
    account_id INT PRIMARY KEY,
    balance DECIMAL(10,2) NOT NULL
);

INSERT INTO Accounts (account_id, balance) VALUES (1, 500), (2, 500)
ON DUPLICATE KEY UPDATE balance = balance;
```

---

## **🖥️ Basic JDBC Example with Deadlock Risk**
This Java code **does not yet handle deadlocks**, so it may fail if a deadlock occurs.

You should create locks in the database before running the application to experiment with deadlocks. 
You can do this by starting a transaction in MySQL Workbench without committing it like in the money transfer example above. This should provoke a deadlock situation when you run the Java program.

```java
import java.sql.*;

public class DeadlockExample {
    private static final String DB_URL = "jdbc:mysql://localhost:3306/bank";
    private static final String USER = "root";
    private static final String PASSWORD = "password";

    public static void main(String[] args) {
        try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASSWORD)) {
            conn.setAutoCommit(false);
            
            try (PreparedStatement stmt = conn.prepareStatement(
                    "UPDATE Accounts SET balance = balance - 100 WHERE account_id = 1")) {
                stmt.executeUpdate();
            }
            
            Thread.sleep(1000); // Simulating delay
            
            try (PreparedStatement stmt = conn.prepareStatement(
                    "UPDATE Accounts SET balance = balance + 100 WHERE account_id = 2")) {
                stmt.executeUpdate();
            }
            
            conn.commit();
            System.out.println("Transaction committed successfully.");
        } catch (SQLException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

---

## **🖥️ Adding a Configurable Retry Mechanism**
Here, we **retry the transaction if a deadlock occurs**, up to `MAX_RETRIES` times.
Run the application like above.

```java
import java.sql.*;

public class DeadlockHandlingExample {
    private static final int MAX_RETRIES = 3;

    public static void main(String[] args) {
        int attempt = 0;
        while (attempt < MAX_RETRIES) {
            try (Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/bank", "root", "password")) {
                conn.setAutoCommit(false);
                
                try (PreparedStatement stmt1 = conn.prepareStatement("UPDATE Accounts SET balance = balance - 100 WHERE account_id = 1")) {
                    stmt1.executeUpdate();
                }
                
                Thread.sleep(500);
                
                try (PreparedStatement stmt2 = conn.prepareStatement("UPDATE Accounts SET balance = balance + 100 WHERE account_id = 2")) {
                    stmt2.executeUpdate();
                }
                
                conn.commit();
                System.out.println("Transaction committed successfully.");
                return;
            } catch (SQLException e) {
                if (e.getSQLState().equals("40001")) { // Deadlock detected
                    System.out.println("Deadlock detected, retrying... Attempt " + (attempt + 1));
                    attempt++;
                } else {
                    e.printStackTrace();
                    break;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                break;
            }
        }
    }
}
```

---

## **🖥️ Adding Logging to Track Deadlocks**
This version **logs deadlock occurrences and failures** to a file (`deadlock_log.txt`) using a manual ```FileWriter```
Run the application like above and inspect the log afterwards.

```java
import java.io.FileWriter;
import java.io.IOException;
import java.sql.*;
import java.time.LocalDateTime;

public class DeadlockHandlingWithLogging {
    private static final String LOG_FILE = "deadlock_log.txt";

    private static void log(String message) {
        try (FileWriter writer = new FileWriter(LOG_FILE, true)) {
            writer.write(LocalDateTime.now() + " - " + message + "\n");
        } catch (IOException e) {
            System.err.println("Logging error: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        int attempt = 0;
        while (attempt < 3) {
            try (Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/bank", "root", "password")) {
                conn.setAutoCommit(false);
                
                log("Attempt " + (attempt + 1) + " started");
                
                try (PreparedStatement stmt1 = conn.prepareStatement("UPDATE Accounts SET balance = balance - 100 WHERE account_id = 1")) {
                    stmt1.executeUpdate();
                }
                
                Thread.sleep(500);
                
                try (PreparedStatement stmt2 = conn.prepareStatement("UPDATE Accounts SET balance = balance + 100 WHERE account_id = 2")) {
                    stmt2.executeUpdate();
                }
                
                conn.commit();
                log("Transaction committed successfully.");
                return;
            } catch (SQLException | InterruptedException e) {
                log("Error: " + e.getMessage());
                attempt++;
            }
        }
    }
}
```
---

## **🖥️ Adding More Professional Logging**
Instead of manual file writing, we could take a more professional approach and use a logging framework like **SLF4J with Logback or Log4j**. 

Rewrite manual logging code and use a logging framework instead. If you choose to use SLF4J & Logback, you can follow along instructions below.

**📜 Setting Up Logging in Java with SLF4J & Logback**

SLF4J (Simple Logging Facade for Java) with Logback allows:

✅ Log levels (INFO, WARN, ERROR, etc.)  
✅ Log rotation (new log files daily, keeping history)  
✅ Multi-thread safe logging  
✅ Support for different output formats (console, file, database, etc.)  

**📌 Step 1: Add Dependencies to `pom.xml`**

Include the required dependencies in your **Maven** project:
```xml
<dependencies>
    <!-- SLF4J API -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.5</version>
    </dependency>

    <!-- Logback (SLF4J Implementation) -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.4.0</version>
    </dependency>
</dependencies>
```

**📌 Step 2: Create `logback.xml` Configuration File**

Inside your **`src/main/resources/`** folder, create a **`logback.xml`** file with the following configuration:

**1️⃣ Basic Console Logging**

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```
🔹 **Logs messages to the console** in the format:
```
2024-03-04 14:30:12 [main] INFO  com.example.Main - Application started
```


 **2️⃣ File-Based Logging with Log Rotation**
 
If you need **log rotation (new log file daily, keeping history)**, use this `logback.xml`:
```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/app-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="FILE" />
    </root>
</configuration>
```
✅ This will:
- Save logs to `logs/app.log`
- Create a **new log file daily** (`app-YYYY-MM-DD.log`)
- **Keep logs for 30 days** (`maxHistory`)

**📌 Step 3: Use SLF4J in Java Code**

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LoggingExample {
    private static final Logger logger = LoggerFactory.getLogger(LoggingExample.class);

    public static void main(String[] args) {
        logger.info("Application started");
        logger.warn("This is a warning message");
        logger.error("This is an error message");
    }
}
```

**📌 Step 4: Change Log Level Dynamically**

Modify **`logback.xml`** to control log verbosity:
```xml
<root level="debug">
    <appender-ref ref="FILE" />
</root>
```
🔹 Change `"debug"` to `"info"`, `"warn"`, or `"error"` to adjust logging output.

---

🚀 **Now, deadlocks are detected, handled, and logged properly!**

