# Task
## Design a stored procedure that monitors the inventory levels of key ingredients like tea leaves, sugar, or fruit concentrates. Once these ingredients fall below a certain threshold, the procedure should automatically create a reorder entry in the purchase order system.

- Create a DB for the project
```
CREATE DATABASE ARIZONA
USE ARIZONA
```

### Creating a Few Tables to make it a real-scenario

- Inventory
(uses alter for the foriegn key because table was already created)
```
CREATE TABLE IngredientsInventory (
    IngredientID INT PRIMARY KEY,
    IngredientName VARCHAR(255),
    CurrentQuantity DECIMAL(10,2), --depends on the Ingredient
    ReorderThreshold DECIMAL(10,2),
    SupplierID INT REFERENCES Suppliers(SupplierID)
);

-- Adding Some Data
INSERT INTO IngredientInventory (IngredientID, IngredientName, CurrentQuantity, ReorderThreshold, SupplierID)
VALUES (1, 'Tea Leaves', 1200.00, 1000.00, 101),
(2, 'Bananas', 1200.00, 1000.00, 231),
(3, 'SUGAR', 1200.00, 1000.00, 102),
(4, 'Leaves', 1200.00, 1000.00, 101);
```


- Suppliers
```
CREATE TABLE Suppliers (
    SupplierID INT PRIMARY KEY,
    SupplierName VARCHAR(255),
    LeadTimeDays INT -- determine how early to order
);

-- some suppliers
INSERT INTO Suppliers (SupplierID, SupplierName, LeadTimeDays)
VALUES (101, 'GreenTea Suppliers', 7),
(231, 'Monkeys', 7),
(102, 'Sugar Depot', 7);
```

- PurchaseOrders:

```
CREATE TABLE PurchaseOrders (
    POID INT PRIMARY KEY,
    IngredientID INT FOREIGN KEY REFERENCES IngredientInventory(IngredientID),
    OrderQuantity DECIMAL(10,2),
    DateOrdered DATE,
    ExpectedDelivery DATE
);
```

- We can add also Logs and Notifications (to build email system)

```
CREATE TABLE Logs (
    LogID INT PRIMARY KEY IDENTITY(1,1),
    LogDate DATETIME,
    LogType VARCHAR(50),
    LogMessage TEXT
);

CREATE TABLE Notifications (
    NotificationID INT PRIMARY KEY IDENTITY(1,1),
    NotificationDate DATETIME,
    NotificationType VARCHAR(50),
    NotificationMessage TEXT, -- even though varchar is better but to show we know different types
    RecipientEmail VARCHAR(255) -- because we are having notifications emailed
);

```
- we need a reorder amount in gredients inventory to know how much do we need to order

```
ALTER TABLE IngredientInventory
ADD ReorderAmount DECIMAL(10,2)
```
- Update column values
```
UPDATE IngredientInventory
SET ReorderAmount = 2900
WHERE SupplierID = 231;

Select * From IngredientInventory
```

- Next the proceduer
```
CREATE PROCEDURE sp_ReorderIngredients
AS
BEGIN
    BEGIN TRY
        -- Start a transaction
        BEGIN TRANSACTION;

        -- Loop through each ingredient that's below the threshold
        DECLARE @IngredientID INT, @ReorderAmount DECIMAL(10,2), @LeadTimeDays INT, @Today DATE = GETDATE();

        DECLARE reorder_cursor CURSOR FOR
        SELECT i.IngredientID, i.ReorderAmount, s.LeadTimeDays 
        FROM IngredientInventory i
        JOIN Suppliers s ON i.SupplierID = s.SupplierID
        WHERE i.CurrentQuantity <= i.ReorderThreshold;

        OPEN reorder_cursor;

        FETCH NEXT FROM reorder_cursor INTO @IngredientID, @ReorderAmount, @LeadTimeDays;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            INSERT INTO PurchaseOrders (IngredientID, OrderQuantity, DateOrdered, ExpectedDelivery)
            VALUES (@IngredientID, @ReorderAmount, @Today, DATEADD(DAY, @LeadTimeDays, @Today));

            -- Add a log for successful reorder
            INSERT INTO Logs (LogDate, LogType, LogMessage)
            VALUES (@Today, 'SUCCESS', 'Reordered ' + CAST(@ReorderAmount AS VARCHAR(10)) + ' for IngredientID ' + CAST(@IngredientID AS VARCHAR(10)));

            -- Send a notification
            INSERT INTO Notifications (NotificationDate, NotificationType, NotificationMessage, RecipientEmail)
            VALUES (@Today, 'Reorder', 'Successfully reordered ' + CAST(@ReorderAmount AS VARCHAR(10)) + ' for IngredientID ' + CAST(@IngredientID AS VARCHAR(10)), 'purchasing@company.com');

            FETCH NEXT FROM reorder_cursor INTO @IngredientID, @ReorderAmount, @LeadTimeDays;
        END;

        CLOSE reorder_cursor;
        DEALLOCATE reorder_cursor;

        -- Commit the transaction
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        -- Rollback the transaction in case of error
        ROLLBACK TRANSACTION;

        -- Insert error details into the log
        INSERT INTO Logs (LogDate, LogType, LogMessage)
        VALUES (GETDATE(), 'ERROR', ERROR_MESSAGE());

        -- Send a notification for the error
        INSERT INTO Notifications (NotificationDate, NotificationType, NotificationMessage, RecipientEmail)
        VALUES (GETDATE(), 'ERROR', 'Failed to reorder. Error: ' + ERROR_MESSAGE(), 'it-support@company.com');
    END CATCH;
END;


```

```
Cannot insert the value NULL into column 'POID', table 'ARIZONA1.dbo.PurchaseOrders'; column does not allow nulls. INSERT fails.
```

Command worked but we got an error in logs and notification because i forgot to increment POID by itself in purchase order by using IDENTITY(,)


so i dropped table and recreated the purchase order with identity (not most effecient choice but fine for this example)

```
A cursor with the name 'reorder_cursor' already exists.
```
this error happened because of the first error when i looked it up it was because the cursor was not close because of the first error and we can avoid that by adding somthing called begin catch(i added the same statment below to my procedure)

this will close it

```
IF CURSOR_STATUS('global','reorder_cursor') >= -1 
BEGIN
    CLOSE reorder_cursor;
    DEALLOCATE reorder_cursor;
END
```




Didn't add the mail service for the sole purpose that i need to create profile and mail server which needs time to configure right (SSL and smtp server)