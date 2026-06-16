---
lab:
    title: 'Lab 7 - Build an Indexing Strategy for the E-Commerce Platform'
    module: 'Optimize query performance using indexes in Azure DocumentDB'
    description: 'Build an indexing strategy for an e-commerce platform in Azure DocumentDB. Create compound indexes using the ESR rule and verify performance improvements with the explain() command.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

# Build an Indexing Strategy for the E-Commerce Platform

In this exercise, you build an indexing strategy for the Cosmicworks e-commerce database. You analyze query patterns without indexes, create targeted indexes using the ESR (Equality, Sort, Range) rule, and use the `explain()` command to verify that your indexes eliminate collection scans. With 1,000 products, 4,000 orders, and 5,000 reviews, the `explain()` output shows meaningful differences between indexed and unindexed queries.

> [!NOTE]
> This exercise assumes you have the `cosmicworks` database loaded from a previous module's exercise. If you need to reload it, follow the download and import steps from the *Model data relationships* exercise.

## Prerequisites

Before you begin this exercise, ensure you have the following installed and configured in your environment:

- [Visual Studio Code](https://code.visualstudio.com/) installed
- [MongoDB Shell (mongosh)](https://www.mongodb.com/try/download/shell) installed
- [MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools) installed (provides the `mongoimport` command)
- An Azure DocumentDB cluster with your admin credentials

## Set up the working environment

Skip this section if you already uploaded your collections to the database.

The `Cosmicworks` dataset contains collections that you use in this exercise to build and test your indexing strategy.

### Create the work folder

First you need a work folder to download the dataset.

1. Open **Visual Studio Code** and open a new terminal (**Terminal > New Terminal**).

1. On your local machine, choose a working directory and use the `cd` command to navigate to it.

1. Create a working folder and download the dataset:

    ```bash
    mkdir cosmicworks
    cd cosmicworks
    ```

1. Download and extract the Cosmicworks dataset:

    macOS/Linux (bash):

    ```bash
    curl -L -o dataset.zip "https://github.com/MicrosoftLearning/mslearn-azure-documentdb/raw/main/Allfiles/Shared/cosmicworks_documentdb_dataset.zip"
    unzip dataset.zip
    ```

    Windows (PowerShell):

    ```powershell
    Invoke-WebRequest -Uri "https://github.com/MicrosoftLearning/mslearn-azure-documentdb/raw/main/Allfiles/Shared/cosmicworks_documentdb_dataset.zip" -OutFile dataset.zip
    Expand-Archive -Path dataset.zip -DestinationPath .
    ```

1. Verify the extracted files. You should see a `collections` folder containing JSON files:

    macOS/Linux (bash):

    ```bash
    ls collections/
    ```

    Windows (PowerShell):

    ```powershell
    dir collections\
    ```

    You should see files including `categories.json`, `products.json`, `customers.json`, `orders.json`, and `reviews.json`.

### Import the dataset

Now load the baseline Cosmicworks data into your Azure DocumentDB cluster.

1. In the Visual Studio Code terminal, set your connection URI as a variable so you don't have to retype it for each import. Replace the placeholders with your cluster details:

    macOS/Linux (bash):

    ```bash
    export URI="mongodb+srv://<your-admin-user>@<your-cluster-name>.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
    ```

    Windows (PowerShell):

    ```powershell
    $env:URI = "mongodb+srv://<your-admin-user>@<your-cluster-name>.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
    ```

1. Import each collection. Enter your password when prompted for each command:

    macOS/Linux (bash):

    ```bash
    mongoimport --uri "$URI" --db cosmicworks --collection categories --file collections/categories.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection customers --file collections/customers.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection inventory --file collections/inventory.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection orders --file collections/orders.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection ordersArchive --file collections/ordersArchive.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection products --file collections/products.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection reviews --file collections/reviews.json --jsonArray
    mongoimport --uri "$URI" --db cosmicworks --collection tags --file collections/tags.json --jsonArray
    ```

    Windows (PowerShell):

    ```powershell
    mongoimport --uri $env:URI --db cosmicworks --collection categories --file collections\categories.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection customers --file collections\customers.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection inventory --file collections\inventory.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection orders --file collections\orders.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection ordersArchive --file collections\ordersArchive.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection products --file collections\products.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection reviews --file collections\reviews.json --jsonArray
    mongoimport --uri $env:URI --db cosmicworks --collection tags --file collections\tags.json --jsonArray
    ```

    Each command should report the number of documents imported.

## Connect to your cluster

Once your Azure DocumentDB has the sample collections loaded, connect to the cluster and start working:

1. If you haven't already, open **Visual Studio Code** and open a terminal (**Terminal > New Terminal**).

1. In the Visual Studio Code terminal, connect to your cluster using mongosh. Replace the placeholders with your cluster details:

    macOS/Linux (bash):

    ```bash
    mongosh "mongodb+srv://<your-admin-user>@<your-cluster-name>.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh "mongodb+srv://<your-admin-user>@<your-cluster-name>.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
    ```

    Enter your password when prompted.

1. Switch to the cosmicworks database and confirm the data is loaded:

    ```javascript
    use cosmicworks
    ```

    ```javascript
    print("Products: " + db.products.countDocuments())
    print("Orders: " + db.orders.countDocuments())
    print("Reviews: " + db.reviews.countDocuments())
    ```

    You should see approximately 1,000 products, 4,000 orders, and 5,000 reviews.

## Analyze queries without indexes (baseline)

Before creating any indexes, run three common e-commerce queries with `explain("executionStats")` to establish baseline performance. Azure DocumentDB indexes only the `_id` field by default, so all other queries perform collection scans.

If a previous exercise left indexes on the products collection, drop them first so the baseline is clean:

```javascript
try { db.products.dropIndex("category.name_1_price_1") } catch(e) {}
try { db.products.dropIndex("tags_1") } catch(e) {}
try { db.products.dropIndex("sku_1") } catch(e) {}
```

### Query 1: Product search by category and price range

```javascript
db.products.find({
  "category.name": "Mountain Bikes",
  price: { $gte: 500, $lte: 2000 }
}).sort({ price: 1 }).explain("executionStats")
```

Check the output for these key fields:
- `stage`: should show `COLLSCAN` (no index available)
- `totalDocsExamined`: should be approximately 1,000 (scans every product)
- `nReturned`: should be much less than totalDocsExamined

Note the `executionTimeMillis` value.

### Query 2: Order lookup by customer

```javascript
db.orders.find({
  customerId: db.customers.findOne({ firstName: "Haladhar" })._id
}).sort({ orderDate: -1 }).explain("executionStats")
```

This scans all ~4,000 orders to find one customer's ~30-40 orders. Note the `totalDocsExamined` vs. `nReturned` values. Additionally, the `sort` stage is performed in memory after the collection scan, which can be inefficient.

### Query 3: Reviews filtered by product and rating

```javascript
var product = db.products.findOne({ isOutlier: true })

db.reviews.find({
  productId: product._id,
  rating: { $gte: 4 }
}).sort({ helpful: -1 }).explain("executionStats")
```

This scans all ~5,000 reviews. Again, note the `totalDocsExamined` vs. `nReturned` values. The sort stage is also performed in memory, which can be inefficient.

## Create compound indexes using the ESR rule

Now create indexes optimized for each query pattern. Apply the ESR (Equality, Sort, Range) rule to determine the optimal field order.

### Index for Query 1 (category = Equality, price = Sort + Range)

```javascript
// ESR: Equality (category.name) then Sort/Range (price)
db.products.createIndex({ "category.name": 1, price: 1 })
```

### Index for Query 2 (customerId = Equality, orderDate = Sort)

```javascript
// ESR: Equality (customerId) then Sort (orderDate descending)
db.orders.createIndex({ customerId: 1, orderDate: -1 })
```

### Index for Query 3 (productId = Equality, helpful = Sort, rating = Range)

```javascript
// ESR: Equality (productId) then Sort (helpful) then Range (rating)
db.reviews.createIndex({ productId: 1, helpful: -1, rating: 1 })
```

## Verify index usage with explain()

Rerun each query with `explain("executionStats")` and compare the results to the baseline.

### Query 1 with index

```javascript
db.products.find({
  "category.name": "Mountain Bikes",
  price: { $gte: 500, $lte: 2000 }
}).sort({ price: 1 }).explain("executionStats")
```

Verify that:
- The `stage` is `IXSCAN` instead of `COLLSCAN`.
- The `totalKeysExamined` is much closer to `nReturned` than the first time you ran the query (the baseline).
- The `executionTimeMillis` is lower than the baseline.
- The index scan rather than in memory now performs the sort, so no `SORT` stage appears in the `explain()` output.

### Query 2 with index

```javascript
db.orders.find({
  customerId: db.customers.findOne({ firstName: "Haladhar" })._id
}).sort({ orderDate: -1 }).explain("executionStats")
```

Verify that the query uses the `customerId_1_orderDate_-1` index and returns results sorted by date without scanning all 4,000 orders.

### Query 3 with index

```javascript
var product = db.products.findOne({ isOutlier: true })

db.reviews.find({
  productId: product._id,
  rating: { $gte: 4 }
}).sort({ helpful: -1 }).explain("executionStats")
```

Verify the query now uses an index scan instead of a collection scan.

## Calculate the efficiency ratio

The efficiency ratio tells you how well an index serves a query:

```javascript
var stats = db.orders.find({
  customerId: db.customers.findOne({ firstName: "Haladhar" })._id
}).sort({ orderDate: -1 }).explain("executionStats")

var returned = stats.executionStats.nReturned
var examined = stats.executionStats.totalKeysExamined

print("nReturned: " + returned)
print("totalKeysExamined: " + examined)
print("Efficiency ratio: " + (returned / examined).toFixed(2))
```

A ratio of **1.0** means every examined index entry resulted in a returned document. The index is perfectly selective for this query.

## Create a multikey index for tag queries

Products have a `tags` array. Create a multikey index to efficiently query by tag:

```javascript
db.products.createIndex({ tags: 1 })
```

Test the multikey index:

```javascript
// Find products with a specific tag
db.products.find({ tags: "mountain" }).explain("executionStats")
```

Verify the `stage` is `IXSCAN`. Then try the `$all` operator for multiple tags:

```javascript
db.products.find(
  { tags: { $all: ["mountain", "aluminum"] } },
  { name: 1, price: 1, tags: 1, _id: 0 }
).explain("executionStats")
```

Multikey indexes work with array operators like `$all`, `$in`, and `$elemMatch`. A single multikey index on `tags` supports all of these query patterns without needing separate indexes for each.

## Create a unique index for SKU lookups

The `sku` field is a natural key for product lookups, and each product must have a distinct SKU. A unique index enforces this constraint at the database level and also makes point lookups efficient:

```javascript
db.products.createIndex({ sku: 1 }, { unique: true })
```

Test the unique constraint by trying to insert a duplicate:

```javascript
try {
  db.products.insertOne({ sku: "BK-M100S-44", name: "Duplicate", price: 0 })
} catch(e) {
  print("Expected error: " + e.message)
}
```

The insert fails with a duplicate key error, which confirms the index enforces uniqueness.

Now run a lookup query to see the performance benefit:

```javascript
db.products.find(
  { sku: "BK-M100S-44" }
).explain("executionStats")
```

Check the top-level `executionStats` in the output:
- `nReturned` should be **1**.
- `totalDocsExamined` should be **1**.
- `totalKeysExamined` should be **1** or **2**. The index narrows directly to the matching entry, so the database examines very few keys to find a single document.

## Identify redundant indexes using prefix matching

The compound index `{ "category.name": 1, price: 1 }` already supports queries on `category.name` alone via prefix matching. Adding a separate single-field index on `category.name` would be redundant.

1. To demonstrate, create a redundant index:

    ```javascript
    db.products.createIndex({ "category.name": 1 })
    ```

1. Verify it's redundant. Both indexes serve the same query:

    ```javascript
    // This query can use EITHER index
    db.products.find({ "category.name": "Road Bikes" }).explain("executionStats")
    ```

1. Drop the redundant single-field index:

    ```javascript
    db.products.dropIndex("category.name_1")
    ```

1. Rerun the same query and confirm the compound index handles it:

    ```javascript
    db.products.find({ "category.name": "Road Bikes" }).explain("executionStats")
    ```

    The `indexName` should now show `category.name_1_price_1`. The compound index serves category-only queries through its prefix, so the single-field index was unnecessary.

## Check index usage statistics

After running the previous queries, check which indexes are being used:

```javascript
db.products.aggregate([
  { $indexStats: {} },
  { $project: { name: 1, "accesses.ops": 1 } },
  { $sort: { "accesses.ops": -1 } }
])
```

```javascript
db.orders.aggregate([
  { $indexStats: {} },
  { $project: { name: 1, "accesses.ops": 1 } },
  { $sort: { "accesses.ops": -1 } }
])
```

```javascript
db.reviews.aggregate([
  { $indexStats: {} },
  { $project: { name: 1, "accesses.ops": 1 } },
  { $sort: { "accesses.ops": -1 } }
])
```

Each index you created and tested should show `accesses.ops` greater than zero.

## Review the final index state

List all indexes across the three collections:

```javascript
print("=== Products Indexes ===")
db.products.getIndexes().forEach(function(idx) {
  print("  " + idx.name + ": " + JSON.stringify(idx.key))
})

print("\n=== Orders Indexes ===")
db.orders.getIndexes().forEach(function(idx) {
  print("  " + idx.name + ": " + JSON.stringify(idx.key))
})

print("\n=== Reviews Indexes ===")
db.reviews.getIndexes().forEach(function(idx) {
  print("  " + idx.name + ": " + JSON.stringify(idx.key))
})
```

You should see a focused set of indexes where each one serves a specific query pattern:
- **Products:** `_id`, `category.name + price` (product search), `tags` (tag filtering), `sku` (unique point lookups)
- **Orders:** `_id`, `customerId + orderDate` (customer order history)
- **Reviews:** `_id`, `productId + helpful + rating` (product reviews)

## Clean up

If you're finished with all modules in this learning path, drop the database:

```javascript
use cosmicworks
```

```javascript
db.dropDatabase()
```


> [!TIP]
> If you plan to continue with the other modules in this learning path, keep the `cosmicworks` database. Those exercises use the same dataset.

In this exercise, you built an indexing strategy from scratch: you established baseline performance with collection scans, created compound indexes using the ESR rule, verified improvements with `explain()`, and identified redundant indexes through prefix matching. You also practiced creating multikey indexes for array fields and unique indexes for data integrity. These techniques help you design indexes that match your application's query patterns while keeping write overhead low.

