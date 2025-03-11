# **Experiment with Concurrency in MySQL Workbench**

---

## **📌 Steps to Follow**

### **1️⃣ Open Two Sessions in MySQL Workbench**
1. Open **MySQL Workbench**.
2. Open **two separate connections** to the same database to simulate multiple users.
3. Each connection represents a **different session**.

---

### **2️⃣ Setup: Create the Players Table (If Not Exists)**
Run this in **either session** to ensure the table exists:
```sql
CREATE TABLE IF NOT EXISTS Players (
    player_id INT PRIMARY KEY,
    username VARCHAR(50),
    ranking INT
);

INSERT INTO Players (player_id, username, ranking) VALUES (1, 'Alice', 1000)
ON DUPLICATE KEY UPDATE ranking = 1000;
```

✅ **Now the table is ready for the simulation.**

---

### **3️⃣ Session 1 (Transaction A) - Start a Transaction & Read Data**
Run the following in **Session 1**:
```sql
START TRANSACTION;
SELECT * FROM Players WHERE player_id = 1; -- Reads from snapshot
```

🔍 **Expected Behavior:** 
- This will read the data **as it exists at the start of the transaction**.
- Even if another transaction updates the row, this session will **continue seeing the old value** until it commits (due to snapshot isolation).

---

### **4️⃣ Session 2 (Transaction B) - Update the Data**
Switch to **Session 2** and run:
```sql
START TRANSACTION;
UPDATE Players SET ranking = ranking + 10 WHERE player_id = 1;
```

🔍 **Expected Behavior:**
- This **modifies the row** but the change is **not yet visible** to other transactions until it is committed.
- Session 1 **still sees the old rank value**.

---

### **5️⃣ Session 1 (Transaction A) - Read Data Again**
Switch back to **Session 1** and run:
```sql
SELECT * FROM Players WHERE player_id = 1; -- Still sees old value
```

🔍 **Expected Behavior:** 
- Even though **Session 2 updated the row**, Session 1 **still sees the old value** because it is reading from its transaction snapshot.

---

### **6️⃣ Commit the Change in Session 2**
Switch to **Session 2** and commit the update:
```sql
COMMIT;
```

🔍 **Now, the update is permanently stored and visible to new transactions.**

---

### **7️⃣ Session 1 (Transaction A) - Read Data Again After Commit in Session 2**
Back in **Session 1**, run:
```sql
SELECT * FROM Players WHERE player_id = 1;
```
🔍 **Expected Behavior:** 
- **Session 1 STILL sees the old value!** 
- This is because MySQL’s **default `REPEATABLE READ` isolation level** ensures that a transaction reads from a consistent snapshot.

---

### **8️⃣ Commit Transaction in Session 1 & Read Again**
Now commit the transaction in **Session 1**:
```sql
COMMIT;
SELECT * FROM Players WHERE player_id = 1;
```

🔍 **Now, Session 1 finally sees the updated value!**

---

## **🔎 Key Takeaways**
| **Scenario** | **Session 1 (Transaction A)** | **Session 2 (Transaction B)** |
|-------------|------------------------------|------------------------------|
| **Start transaction** | Reads snapshot | Starts transaction |
| **Update data** | Sees old value | Updates a row |
| **Read data again** | Still sees old value | Sees updated value |
| **Commit in Session 2** | Still sees old value | Commits the update |
| **Read data after commit** | Still sees old value until commit | Sees updated value |
| **Commit in Session 1** | Now sees updated value | Already committed |

💡 **By default, MySQL uses `REPEATABLE READ` isolation level, meaning transactions continue to see the snapshot taken at the start until they commit.**

---

## **💡 Bonus: Test with `READ COMMITTED` Isolation Level**
Want to test a different isolation level? Switch **Session 1** to `READ COMMITTED` and see how it behaves:
```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM Players WHERE player_id = 1; -- Now sees the latest committed value
```
🔍 **With `READ COMMITTED`, Session 1 will see updates as soon as they are committed in Session 2!** Now you can get non-repeatable reads. What problems could that cause?
<details>
  <summary>Help</summary>
  <p>  
  A transaction may base a decision on outdated data. <br>
 Example: A banking app checks a user’s balance before allowing a withdrawal, but another transaction changes the balance in between, leading to incorrect results. 
<details>
  <summary>More help</summary>

  **Real-world scenario:**
  
  ```sql
  -- Transaction A: Check if the user has enough balance
  START TRANSACTION;
  SELECT balance FROM Accounts WHERE account_id = 1;  -- Returns $500

  -- Transaction B: Another transaction updates the balance
  UPDATE Accounts SET balance = 100 WHERE account_id = 1;
  COMMIT;

  -- Transaction A: Withdraw money (but still sees old balance)
  UPDATE Accounts SET balance = balance - 300 WHERE account_id = 1;
  COMMIT;
```

---

## **🎯 Conclusion**
- **Snapshot isolation in MySQL (`REPEATABLE READ`) ensures that a transaction sees a consistent view of the database, ignoring committed changes from other transactions.**
- **To see the latest committed data, switch to `READ COMMITTED` isolation level.**
- **Understanding isolation levels helps prevent concurrency issues like lost updates and inconsistent reads.**
