---
lab:
    title: 'Lab 6 - Identify and fix anti-patterns'
    module: 'Recognize and avoid schema design anti-patterns in Azure DocumentDB'
    description: 'Identify and fix schema design anti-patterns in an Azure DocumentDB e-commerce database, including unbounded arrays, overlapping indexes, over-normalization, and case-sensitivity issues.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

In this exercise, you identify and fix five schema design anti-patterns using deliberately flawed data from the Cosmicworks dataset. For each anti-pattern, you load the flawed data, examine the problem, apply a fix, and verify the result.

> [!NOTE]
> This exercise assumes you have an Azure DocumentDB cluster with public access enabled. If you need to set up a cluster, refer to the earlier module in this learning path: *Create and configure an Azure DocumentDB cluster*.

## Prerequisites

Before you begin this exercise, ensure you have the following installed and configured in your environment:

- [Visual Studio Code](https://code.visualstudio.com/) installed
- [MongoDB Shell (mongosh)](https://www.mongodb.com/try/download/shell) installed
- [MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools) installed (provides the `mongoimport` command)
- An Azure DocumentDB cluster with your admin credentials

## Set up the working environment

The Cosmicworks dataset contains the correct baseline data used throughout this exercise. The `anti-patterns/` folder contains deliberately flawed versions of that data, with one file per anti-pattern.

> [!NOTE]
> If you completed a previous exercise, you may be able to skip parts of this setup. If you still have the `anti-patterns/` folder on your local machine, skip the download step. If the `cosmicworks` database is already populated in your cluster, skip the import step. If both conditions are true, go directly to **Identify and fix unbounded arrays**. Make sure your connection URI is set as an environment variable before you continue.

### Create the work folder

First we need a work folder to download the dataset we're using.

1. Open **Visual Studio Code** and open a new terminal (**Terminal > New Terminal**).

1. On your local machine, choose a working directory.

1. Create a working folder and navigate into it:

  
    ```bash
    mkdir cosmicworks
    ```

    ```bash
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

1. Verify the extracted files. You should see both a `collections` folder and an `anti-patterns` folder:

    macOS/Linux (bash):

    ```bash
    ls collections/
    ls anti-patterns/
    ```

    Windows (PowerShell):

    ```powershell
    dir collections\
    dir anti-patterns\
    ```

    The `collections` folder should contain files including `categories.json`, `products.json`, `customers.json`, `orders.json`, and `reviews.json`. The `anti-patterns` folder should contain `products-unbounded-reviews.json`, `products-over-normalized.json`, `products-mixed-case.json`, `setup-unnecessary-indexes.js`, and a `sprawl-collections/` folder.

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

1. Import each collection into your cluster:

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

## Identify and fix unbounded arrays

The `products-unbounded-reviews.json` file contains 15 products with all their reviews embedded directly in an unbounded `reviews` array, including one outlier product with 100+ embedded reviews.

1. Import the flawed data into a separate database:

    macOS/Linux (bash):

    ```bash
    mongoimport --uri "$URI" --db antipatterns --collection products_bloated --file anti-patterns/products-unbounded-reviews.json --jsonArray
    ```

    Windows (PowerShell):

    ```powershell
    mongoimport --uri $env:URI --db antipatterns --collection products_bloated --file anti-patterns\products-unbounded-reviews.json --jsonArray
    ```

1. Connect with mongosh:

    macOS/Linux (bash):

    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

1. In mongosh, switch to the antipatterns database and find the product with the most embedded reviews:

    ```javascript
    use antipatterns
    ```

    ```javascript
    // Find the product with the most embedded reviews
    db.products_bloated.aggregate([
      { $project: { name: 1, reviewCount: { $size: "$reviews" } } },
      { $sort: { reviewCount: -1 } },
      { $limit: 3 }
    ])
    ```

    You should see three products with 100+ reviews embedded. This document is bloated and keeps growing with every new review.

1. Check the document size of the outlier:

    ```javascript
    var bloated = db.products_bloated.findOne(
      {},
      { reviews: 1 }
    )
    print("Embedded reviews: " + bloated.reviews.length)
    print("Approximate doc size: " + bsonsize(bloated) + " bytes")
    ```

    Notice the `reviewSummary` field is missing. Without precomputed stats, every page view would need to aggregate the embedded array.

1. Fix the anti-pattern by applying the subset pattern. Move reviews to a separate collection and keep only the top 3:

    ```javascript
    // Move all reviews to a separate collection
    db.products_bloated.find().forEach(function(product) {
      if (product.reviews && product.reviews.length > 0) {
        product.reviews.forEach(function(review) {
          review.productId = product._id;
          db.reviews_fixed.insertOne(review);
        });
      }
    })

    // Verify reviews were moved
    print("Reviews moved: " + db.reviews_fixed.countDocuments())
    ```

1. Update products to use the subset pattern:

    ```javascript
    db.products_bloated.find().forEach(function(product) {
      var topReviews = db.reviews_fixed.find(
        { productId: product._id }
      ).sort({ helpful: -1 }).limit(3).toArray();

      var stats = db.reviews_fixed.aggregate([
        { $match: { productId: product._id } },
        { $group: { _id: null, count: { $sum: 1 }, avg: { $avg: "$rating" } } }
      ]).toArray()[0];

      db.products_bloated.updateOne(
        { _id: product._id },
        {
          $unset: { reviews: "" },
          $set: {
            topReviews: topReviews,
            reviewSummary: {
              totalCount: stats ? stats.count : 0,
              averageRating: stats ? Math.round(stats.avg * 10) / 10 : 0
            }
          }
        }
      );
    })

    // Verify the fix
    var fixed = db.products_bloated.findOne()
    print("Embedded reviews removed: " + (fixed.reviews === undefined))
    print("Top reviews: " + (fixed.topReviews ? fixed.topReviews.length : 0))
    print("Review summary: " + JSON.stringify(fixed.reviewSummary))
    ```

  1. Exit mongosh to move to the next section.

     ```javascript
     exit
     ```

By the script applying the subset pattern, the product document now stays small and predictable. The full review history lives in its own collection where it can grow without affecting product query performance, while the `topReviews` array and `reviewSummary` give the product page everything it needs in a single read.

## Identify and fix collection sprawl

The `sprawl-collections/` folder contains 37 separate JSON files, one per product category. These collections simulate the anti-pattern of creating one collection per category instead of using a single collection with a discriminator field.

1. In your Visual Studio Code terminal (outside mongosh), count the sprawl files:

    macOS/Linux (bash):

    ```bash
    ls anti-patterns/sprawl-collections/ | wc -l
    ```

    Windows (PowerShell):

    ```powershell
    (dir anti-patterns\sprawl-collections\).Count
    ```

    You should see 37 files. Each would become a separate collection with its own indexes.

1. Import a few of them to see the problem:

    macOS/Linux (bash):

    ```bash
    mongoimport --uri "$URI" --db antipatterns --collection mountain_bikes --file anti-patterns/sprawl-collections/mountain_bikes.json --jsonArray
    mongoimport --uri "$URI" --db antipatterns --collection road_bikes --file anti-patterns/sprawl-collections/road_bikes.json --jsonArray
    mongoimport --uri "$URI" --db antipatterns --collection helmets --file anti-patterns/sprawl-collections/helmets.json --jsonArray
    ```

    Windows (PowerShell):

    ```powershell
    mongoimport --uri $env:URI --db antipatterns --collection mountain_bikes --file anti-patterns\sprawl-collections\mountain_bikes.json --jsonArray
    mongoimport --uri $env:URI --db antipatterns --collection road_bikes --file anti-patterns\sprawl-collections\road_bikes.json --jsonArray
    mongoimport --uri $env:URI --db antipatterns --collection helmets --file anti-patterns\sprawl-collections\helmets.json --jsonArray
    ```

1. Connect to DocumentDB with mongosh:

    macOS/Linux (bash):

    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

1. In mongosh, see the problem: you can't query across categories:

    ```javascript
    use antipatterns
    ```

    ```javascript
    // Each category is isolated
    print("Mountain bikes: " + db.mountain_bikes.countDocuments())
    print("Road bikes: " + db.road_bikes.countDocuments())
    print("Helmets: " + db.helmets.countDocuments())

    // To find the cheapest product across ALL categories,
    // you'd need to query each collection separately and merge in code!
    ```

1. The fix is already in the main dataset. Compare with the correct `products` collection in `cosmicworks`:

    ```javascript
    use cosmicworks
    ```

    ```javascript
    // Single collection with category.name as discriminator
    db.products.aggregate([
      { $group: { _id: "$category.name", count: { $sum: 1 } } },
      { $sort: { count: -1 } },
      { $limit: 5 }
    ])
    ```
    
    ```javascript
    // Cross-category query works instantly
    db.products.find().sort({ price: 1 }).limit(3).forEach(function(p) {
      print(p.name + " ($" + p.price + ") - " + p.category.name)
    })
    ```

    One collection, one index on `category.name`, and cross-category queries work naturally. No need to take care of merging results from multiple collections in application code.

## Identify and fix unnecessary indexes

The `setup-unnecessary-indexes.js` script creates 28 indexes on a products collection, far more than any query pattern needs.

1. Exit mongosh to run the unnecessary indexes script:

    ```javascript
    exit
    ```

1. Run the unnecessary indexes script. In your Visual Studio Code terminal (outside mongosh), run (this script might take a minute or two to run):

    macOS/Linux (bash):

    ```bash
    mongosh "$URI" --file anti-patterns/setup-unnecessary-indexes.js
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI --file anti-patterns\setup-unnecessary-indexes.js
    ```

1. Connect to DocumentDB with mongosh:

    macOS/Linux (bash):
r
    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

1. In mongosh, count the indexes:

    ```javascript
    use cosmicworks
    ```
    
    ```javascript
    print("Total indexes: " + db.products.getIndexes().length)
    db.products.getIndexes().forEach(function(idx) {
      print("  " + idx.name + ": " + JSON.stringify(idx.key))
    })
    ```

1. Check which indexes are actually used:

    ```javascript
    db.products.aggregate([{ $indexStats: {} }]).forEach(function(stat) {
      print(stat.name + " (" + JSON.stringify(stat.key) + ") - ops: " + stat.accesses.ops)
    })
    ```

    Most indexes show zero or near-zero operations.

1. Drop the unnecessary indexes (keep only the ones that serve real query patterns):

    ```javascript
    // These indexes don't support common query patterns
    db.products.dropIndex("description_1")
    db.products.dropIndex("createdAt_1")
    db.products.dropIndex("isActive_1")
    db.products.dropIndex("schemaVersion_1")
    db.products.dropIndex("isOutlier_1")
    db.products.dropIndex("parentCategory_1")
    db.products.dropIndex("inventory_1")
    db.products.dropIndex("price_1")
    db.products.dropIndex("name_1")
    db.products.dropIndex("name_1_price_1")
    db.products.dropIndex("category.name_1")
    db.products.dropIndex("productType_1")
    db.products.dropIndex("specs.frameSize_1")
    db.products.dropIndex("specs.frameMaterial_1")
    db.products.dropIndex("specs.color_1")
    db.products.dropIndex("specs.weight_kg_1")
    db.products.dropIndex("specs.gears_1")
    db.products.dropIndex("specs.brakeType_1")
    db.products.dropIndex("specs.suspensionType_1")
    db.products.dropIndex("specs.wheelSize_inches_1")
    db.products.dropIndex("approximateMetrics.viewCount_1")
    db.products.dropIndex("approximateMetrics.wishlistCount_1")
    db.products.dropIndex("approximateMetrics.cartAdditions_1")
    db.products.dropIndex("reviewSummary.averageRating_1")
    db.products.dropIndex("reviewSummary.totalCount_1")

    print("Remaining indexes: " + db.products.getIndexes().length)

    db.products.aggregate([{ $indexStats: {} }]).forEach(function(stat) {
      print(stat.name + " (" + JSON.stringify(stat.key) + ") - ops: " + stat.accesses.ops)
    })
    ```

    You should be left with a small set of useful indexes: `_id`, `category.name + price` (ESR for product searches), `tags` (multikey for tag filtering), and `sku` (unique lookups).

1. Exit mongosh:

    ```javascript
    exit
    ```

Keeping unnecessary indexes not only consumes storage and RAM, but also slows down writes. By dropping the unused indexes, you can improve write performance and reduce costs while still supporting all your query patterns efficiently.

## Identify and fix over-normalization

The `products-over-normalized.json` file has all 1,000 products but with category names and tag names replaced with just ID. This forces `$lookup` for every display query.

1. Import the over-normalized products:

    macOS/Linux (bash):

    ```bash
    mongoimport --uri "$URI" --db antipatterns --collection products_normalized --file anti-patterns/products-over-normalized.json --jsonArray
    ```

    Windows (PowerShell):

    ```powershell
    mongoimport --uri $env:URI --db antipatterns --collection products_normalized --file anti-patterns\products-over-normalized.json --jsonArray
    ```

1. Connect to DocumentDB with mongosh:

    macOS/Linux (bash):

    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

1. In mongosh, see the problem: display queries require lookups:

    ```javascript
    use antipatterns
    ```
    
    ```javascript
    // The product only has categoryId, not the category name
    db.products_normalized.findOne(
      {},
      { name: 1, categoryId: 1, tagIds: 1, _id: 0 }
    )
    ```

    To display "Mountain Bikes" on the product page, you'd need a `$lookup` to the categories collection.

1. Show how much more expensive the over-normalized query is:

    ```javascript
    // Over-normalized: requires $lookup to get category name
    var start = new Date()
    db.products_normalized.aggregate([
      { $limit: 100 },
      {
        $lookup: {
          from: "categories",
          localField: "categoryId",
          foreignField: "_id",
          as: "categoryInfo"
        }
      },
      { $unwind: "$categoryInfo" },
      { $project: { name: 1, category: "$categoryInfo.name" } }
    ]).toArray()
    var lookupTime = new Date() - start
    ```

      Compare with the correct model (uses cosmicworks.products)
    
    ```javascript
    use cosmicworks
    ```

    ```javascript
    var start2 = new Date()
    db.products.find(
      {},
      { name: 1, "category.name": 1 }
    ).limit(100).toArray()
    var embeddedTime = new Date() - start2

    print("Over-normalized (with $lookup): " + lookupTime + " ms")
    print("Denormalized (embedded): " + embeddedTime + " ms")
    ```

    The embedded version should be faster because it avoids the cross-collection join.

1. Exit mongosh:

    ```javascript
    exit
    ```

Denormalization is a common technique in DocumentDB to optimize read performance. By embedding frequently accessed data (like category names) directly in the product document, you can avoid expensive `$lookup` operations and speed up queries. The over-normalized model may save some storage space but at the cost of query performance and increased complexity.

## Identify and fix case-sensitivity issues

The `products-mixed-case.json` file contains 100 products with randomly mutated casing on names and category names, plus 10 standalone category documents with different casing. Those standalone entries simulate what happens when category records with inconsistent casing leak into a products collection.

1. Import the mixed-case products:

    macOS/Linux (bash):

    ```bash
    mongoimport --uri "$URI" --db antipatterns --collection products_casing --file anti-patterns/products-mixed-case.json --jsonArray
    ```

    Windows (PowerShell):

    ```powershell
    mongoimport --uri $env:URI --db antipatterns --collection products_casing --file anti-patterns\products-mixed-case.json --jsonArray
    ```

1. Connect to DocumentDB with mongosh:

    macOS/Linux (bash):

    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

1. In mongosh, see the problem: case-sensitive queries miss results:

    ```javascript
    use antipatterns
    ```
    
    ```javascript
    // Search for "Mountain Bikes" -- misses lowercase and uppercase variants
    var exact = db.products_casing.countDocuments({ "category.name": "Mountain Bikes" })
    var lower = db.products_casing.countDocuments({ "category.name": "mountain bikes" })
    var upper = db.products_casing.countDocuments({ "category.name": "MOUNTAIN BIKES" })

    print("Exact 'Mountain Bikes': " + exact)
    print("Lowercase 'mountain bikes': " + lower)
    print("Uppercase 'MOUNTAIN BIKES': " + upper)
    print("Total missed by any single query!")
    ```

1. Find the duplicate category entries at the end of the collection:

    ```javascript
    // The mixed-case file includes duplicate categories with different casing
    db.products_casing.aggregate([
      { $group: { _id: { $toLower: "$category.name" }, variants: { $addToSet: "$category.name" }, count: { $sum: 1 } } },
      { $match: { $expr: { $gt: [{ $size: "$variants" }, 1] } } }
    ])
    ```

    You should see categories like "Mountain Bikes", "mountain bikes", "MOUNTAIN BIKES", and "mOunTAin biKeS" all treated as different values.

1. Fix the issue by normalizing category names to proper case:

    ```javascript
    // Count total documents before fix
    print("Total products: " + db.products_casing.countDocuments())

    // Find all distinct category name variants (lowercased)
    var categories = db.products_casing.aggregate([
      { $group: { _id: { $toLower: "$category.name" } } }
    ]).toArray()

    print("Unique categories (case-insensitive): " + categories.length)
    ```

      For a real fix, normalize the category names using the correct casing from the categories collection

    ```javascript
    use cosmicworks
    ```
    
    ```javascript
    var correctCategories = {}
    db.categories.find().forEach(function(cat) {
      correctCategories[cat.name.toLowerCase()] = cat.name
    })
    ```
    
    ```javascript
    use antipatterns
    ```
    
    ```javascript
    db.products_casing.find({ "category.name": { $exists: true } }).forEach(function(product) {
      if (product.category && product.category.name) {
        var correctName = correctCategories[product.category.name.toLowerCase()]
        if (correctName && correctName !== product.category.name) {
          db.products_casing.updateOne(
            { _id: product._id },
            { $set: { "category.name": correctName } }
          )
        }
      }
    })

    // Remove the 10 standalone category documents that don't follow the product schema
    // These have a top-level "name" field instead of "category.name"
    var removed = db.products_casing.deleteMany({
      $or: [
        { "category.name": { $exists: false } },
        { "category.name": null }
      ]
    })
    print("Removed invalid entries: " + removed.deletedCount)

    // Verify the fix
    db.products_casing.aggregate([
      { $group: { _id: "$category.name", count: { $sum: 1 } } },
      { $sort: { count: -1 } },
      { $limit: 5 }
    ])
    ```

    All category names should now use consistent casing, and the invalid duplicate entries are removed.

## Clean up

Drop the antipatterns database:

```javascript
use antipatterns
```

```javascript
db.dropDatabase()
```

```javascript
use cosmicworks
```

```javascript
db.dropDatabase()
```

> [!TIP]
> If you plan to continue with the other modules in this learning path, keep the `cosmicworks` database. Those exercises use the same dataset.

In this exercise, you learned how to identify and fix schema design anti-patterns, including case-sensitivity issues in category names. You should now be able to apply these techniques to ensure consistent data quality in your own DocumentDB collections.