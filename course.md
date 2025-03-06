# 📌 **Course Session: Transactions & Concurrency in MySQL and Java**

## **🎯 Learning Objectives**
By the end of this session, students should be able to:
- Understand the **ACID** properties of transactions.
- Identify **concurrency issues** (lost updates, dirty reads, non-repeatable reads, phantom reads).
- Understand **serializability** and its role in transaction correctness.
- Use **isolation levels** to manage concurrency in MySQL.
- Implement **transaction management** in Java using **JDBC** and **JPA**.
- Handle **deadlocks** and implement proper **locking mechanisms**.

---

## **📖 1️⃣ Lecture Material (Slides)**

### **🔹 Transactions Overview**
- **Definition**: A transaction is a unit of work that is executed atomically.
- **ACID Properties:**
  - **Atomicity**: All or nothing.
  - **Consistency**: Database remains valid.
  - **Isolation**: Transactions do not interfere.
  - **Durability**: Changes persist after commit.

### **🔹 Serializability: Ensuring Correct Execution Order**
- **Serializability** ensures that the **final result of executing multiple concurrent transactions is the same as if they were executed in some serial order**.
- This is the **strongest level of isolation**, but it **may reduce concurrency and performance**.
- **Techniques to Achieve Serializability:**
  - **Two-Phase Locking (2PL)** – Transactions acquire all locks before releasing any.
  - **Timestamp Ordering** – Transactions execute based on timestamps.
  - **Validation-Based Concurrency Control** – Transactions validate before committing.
- **Downside:** Full serializability is expensive and can reduce parallel execution.

### **🔹 Implementing Serializability Techniques in MySQL and Java (JDBC)**
#### **1️⃣ Two-Phase Locking (2PL)**
**Concept:** A transaction must acquire all locks before releasing any locks. This ensures strict execution order.

**MySQL Example:**
```sql
START TRANSACTION;
SELECT * FROM Players WHERE player_id = 1 FOR UPDATE; -- Acquire lock
UPDATE Players SET rank = rank + 10 WHERE player_id = 1;
COMMIT; -- Release lock
```

**Java (JDBC) Example:**
```java
Connection conn = DriverManager.getConnection(url, user, password);
conn.setAutoCommit(false);
try (PreparedStatement stmt = conn.prepareStatement("SELECT * FROM Players WHERE player_id = ? FOR UPDATE")) {
    stmt.setInt(1, 1);
    stmt.executeQuery();
    
    PreparedStatement updateStmt = conn.prepareStatement("UPDATE Players SET rank = rank + 10 WHERE player_id = ?");
    updateStmt.setInt(1, 1);
    updateStmt.executeUpdate();
    
    conn.commit(); // Locks are released here
} catch (SQLException e) {
    conn.rollback();
}
```

#### **2️⃣ Timestamp Ordering**
**Concept:** Transactions execute in order of their timestamps. If a later transaction attempts to modify data before an earlier one commits, it is aborted.

**MySQL does not support full timestamp ordering natively**, but you can simulate it by adding a timestamp column and checking before updates.

**MySQL Example:**
```sql
ALTER TABLE Players ADD COLUMN last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;

START TRANSACTION;
SELECT last_modified FROM Players WHERE player_id = 1;
UPDATE Players SET rank = rank + 10 WHERE player_id = 1 AND last_modified = (SELECT last_modified FROM Players WHERE player_id = 1);
COMMIT;
```

**Java (JDBC) Example:**
```java
Connection conn = DriverManager.getConnection(url, user, password);
conn.setAutoCommit(false);
try (PreparedStatement checkStmt = conn.prepareStatement("SELECT last_modified FROM Players WHERE player_id = ?")) {
    checkStmt.setInt(1, 1);
    ResultSet rs = checkStmt.executeQuery();
    if (rs.next()) {
        Timestamp lastModified = rs.getTimestamp("last_modified");
        
        PreparedStatement updateStmt = conn.prepareStatement(
            "UPDATE Players SET rank = rank + 10 WHERE player_id = ? AND last_modified = ?");
        updateStmt.setInt(1, 1);
        updateStmt.setTimestamp(2, lastModified);
        
        int rowsUpdated = updateStmt.executeUpdate();
        if (rowsUpdated == 0) {
            throw new SQLException("Transaction aborted due to outdated timestamp.");
        }
    }
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

#### **3️⃣ Validation-Based Concurrency Control (Optimistic Concurrency Control)**
**Concept:** Transactions work without locks but validate before committing to ensure no conflicting changes occurred.

**MySQL Example:**
```sql
START TRANSACTION;
SELECT rank FROM Players WHERE player_id = 1;
-- Perform validation check
UPDATE Players SET rank = rank + 10 WHERE player_id = 1 AND (SELECT COUNT(*) FROM Players WHERE player_id = 1) = 1;
COMMIT;
```

**Java (JDBC) Example:**
```java
Connection conn = DriverManager.getConnection(url, user, password);
conn.setAutoCommit(false);
try (PreparedStatement checkStmt = conn.prepareStatement("SELECT rank FROM Players WHERE player_id = ?")) {
    checkStmt.setInt(1, 1);
    ResultSet rs = checkStmt.executeQuery();
    if (rs.next()) {
        int rank = rs.getInt("rank");
        
        PreparedStatement updateStmt = conn.prepareStatement(
            "UPDATE Players SET rank = ? WHERE player_id = ?");
        updateStmt.setInt(1, rank + 10);
        updateStmt.setInt(2, 1);
        
        int rowsUpdated = updateStmt.executeUpdate();
        if (rowsUpdated == 0) {
            throw new SQLException("Transaction validation failed.");
        }
    }
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

---

Would you like me to expand on **lock escalation** or **deadlock resolution** strategies? 🚀
