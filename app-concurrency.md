## **📖 How to Handle to Concurrency Issues in Application Code**
Concurrency control is essential to prevent data inconsistencies when multiple transactions access the same data. Often a **business transaction** executes across a series of **system transactions**. Once outside the confines of a single system transaction, one can't depend on the database manager alone to ensure that the business transaction will leave the record data in a consistent state. Data integrity is at risk once two sessions begin to work on the same records and lost updates are quite possible. Also, with one session editing data that another is reading an inconsistent read becomes likely.

[Optimistic Currency Control](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html), also known as optimistic locking, solves this problem by validating that the changes about to be committed by one session don't conflict with the changes of another session. A successful pre-commit validation is, in a sense, obtaining a lock indicating it's okay to go ahead with the changes to the record data. So long as the validation and the updates occur within a single system transaction the business transaction will display consistency.

![OptimisticSketch](https://github.com/user-attachments/assets/b849cedf-c897-4540-9813-d8f9518e421e)


[Pessimistic Locking](https://martinfowler.com/eaaCatalog/pessimisticOfflineLock.html), on the other hand, assumes that the chance of session conflict is high and therefore limits the system's concurrency to prevent conflicts between concurrent business transactions. Pessimistic Locking allows only one business transaction at a time to access data. It forces a business transaction to acquire a lock on a piece of data before it starts to use it, so that once you begin a business transaction you can be pretty sure you'll complete it without being bounced by concurrency control. Optimistic Lock assumes that the chance of conflict is low. The expectation that session conflict isn't likely allows multiple users to work with the same data at the same time. 

![PessimisticSketch](https://github.com/user-attachments/assets/21c43d7b-743f-4e00-8c74-f60c6ed00bb0)

### **How Optimistic Concurrency Control Works**

1️⃣ A transaction reads a record **along with a version number**.  
2️⃣ The transaction **performs modifications** but **does not lock the row**.  
3️⃣ Before updating, the transaction checks if the **version number has changed**.  
4️⃣ **If the version is unchanged** → Commit the update.  
5️⃣ **If the version has changed** (another transaction modified the row) → Retry or fail with an error.  

### **How Pessimistic Locking Works**

1️⃣ A transaction locks a record **before making modifications**.  
2️⃣ Other transactions attempting to modify the locked record **must wait**.  
3️⃣ The lock is held until the transaction **commits or rolls back**.  

### **Comparing Optimistic Concurrency Control vs. Pessimistic Locking**

| Feature | Optimistic Concurrency Control | Pessimistic Locking |
|---------|------------------------------------|--------------------|
| **Locking Mechanism** | No locks; checks conflicts at commit | Locks rows to prevent conflicts |
| **Performance** | High for read-heavy workloads | Can cause bottlenecks with many updates |
| **Conflict Handling** | Detects conflicts at update time | Prevents conflicts before they happen |
| **Best Use Case** | When conflicts are rare (e.g., user profile updates) | When conflicts are frequent (e.g., banking transactions) |
| **Downside** | May need retries if conflicts occur | Locks can slow down concurrent access |

✅ **Optimistic Concurrency Control is suitable when conflicts are rare**, while **Pessimistic Locking is better for frequent write conflicts**.

---

## **Code Examples: Implementing Optimistic Concurrency Control and Pessimistic Locking**

The examples use a `products` table and they are based on MySQL and Java JDBC.

### **📌 Part 1: Create a Table with a Version Column**


#### **🔹 Step 1: Create a `products` Table**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL,
    version INT NOT NULL DEFAULT 1
);
```

#### **🔹 Step 2: Insert Sample Data**
```sql
INSERT INTO products (name, price, stock_quantity, version) VALUES
('Laptop', 1200.00, 10, 1),
('Mouse', 25.50, 50, 1);
```
✅ **The `version` column starts at `1` and increments with each update.**


### **📌 Part 2: Implement Optimistic Concurrency in Java (JDBC)**
The program must:

1️⃣ Read a product’s details **including the version number**.  
2️⃣ Attempt to update the price **only if the version number matches**.  
3️⃣ **Handle conflicts** (if another transaction has modified the row).  

#### **🔹 Java Code:**
```java
import java.sql.*;

public class OptimisticConcurrencyExample {
    private static final String DB_URL = "jdbc:mysql://localhost:3306/testdb";
    private static final String USER = "root";
    private static final String PASSWORD = "password";

    public static void main(String[] args) {
        int productId = 1;
        double newPrice = 1300.00;
        
        try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASSWORD)) {
            conn.setAutoCommit(false); // Start transaction
            
            // Step 1: Read the product details with version
            String selectSQL = "SELECT price, version FROM products WHERE product_id = ?";
            int currentVersion;
            
            try (PreparedStatement selectStmt = conn.prepareStatement(selectSQL)) {
                selectStmt.setInt(1, productId);
                ResultSet rs = selectStmt.executeQuery();
                
                if (!rs.next()) {
                    System.out.println("Product not found.");
                    return;
                }
                
                currentVersion = rs.getInt("version");
            }
            
            // Step 2: Attempt to update with OCC
            String updateSQL = "UPDATE products SET price = ?, version = version + 1 WHERE product_id = ? AND version = ?";
            try (PreparedStatement updateStmt = conn.prepareStatement(updateSQL)) {
                updateStmt.setDouble(1, newPrice);
                updateStmt.setInt(2, productId);
                updateStmt.setInt(3, currentVersion);
                
                int rowsUpdated = updateStmt.executeUpdate();
                
                if (rowsUpdated == 0) {
                    System.out.println("Optimistic locking failed: The product was modified by another transaction.");
                    conn.rollback(); // Undo changes
                } else {
                    conn.commit(); // Commit transaction
                    System.out.println("Product updated successfully!");
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```


### **📌 Part 3: Implement Pessimistic Locking in Java (JDBC)**

The program uses:

1️⃣ SELECT FOR UPDATE SQL statement for database locking

2️⃣ The lock is held until the transaction **commits or rolls back**.   

#### **🔹 Java Code:**
```java
try (Connection conn = DriverManager.getConnection(DB_URL, USER, PASSWORD)) {
    conn.setAutoCommit(false);
    
    // Lock the row before updating
    String lockSQL = "SELECT price FROM products WHERE product_id = ? FOR UPDATE";
    try (PreparedStatement lockStmt = conn.prepareStatement(lockSQL)) {
        lockStmt.setInt(1, productId);
        lockStmt.executeQuery();
    }
    
    // Now perform the update
    String updateSQL = "UPDATE products SET price = ? WHERE product_id = ?";
    try (PreparedStatement updateStmt = conn.prepareStatement(updateSQL)) {
        updateStmt.setDouble(1, newPrice);
        updateStmt.setInt(2, productId);
        updateStmt.executeUpdate();
    }
    
    conn.commit();
    System.out.println("Product updated successfully!");
} catch (SQLException e) {
    e.printStackTrace();
}
```