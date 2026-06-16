---
lab:
    title: 'Lab 5 - Apply patterns to the e-commerce platform'
    module: 'Apply schema design patterns in Azure DocumentDB'
    description: 'Apply schema design patterns to an e-commerce platform in Azure DocumentDB. Implement the inheritance, computed, subset, and single collection patterns with hands-on exercises.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

In this exercise, you explore 10 schema design patterns in the Cosmicworks e-commerce dataset. The dataset was designed with these patterns built in, so you examine how each pattern is implemented, run queries that demonstrate why the pattern matters, and make modifications that exercise the pattern's behavior.

> [!NOTE]
> This exercise assumes you have the `cosmicworks` database loaded from the previous module's exercise. If you need to reload it, follow the download and import steps from the *Model data relationships* exercise.

## Prerequisites

Before you begin this exercise, ensure you have the following installed and configured in your environment:

- [Visual Studio Code](https://code.visualstudio.com/) installed
- [MongoDB Shell (mongosh)](https://www.mongodb.com/try/download/shell) installed
- [MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools) installed (provides the `mongoimport` command)
- An Azure DocumentDB cluster with your admin credentials

## Set up the working environment

Skip this section if you already uploaded your collections to the database.

The `Cosmicworks` dataset contains collections that you use to review the schema design patterns in this module.

### Create the work folder

First we need a work folder to download the dataset we're using.

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

Now you retrieve the dataset from the GitHub repo, expand it, and load it into your Azure DocumentDB cluster.

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

Once your Azure DocumentDB has the sample collections loaded, it's time to connect to the cluster and start working:

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

1. Switch to the cosmicworks database:

    ```javascript
    use cosmicworks
    ```

## Explore the inheritance pattern (productType discriminator)

The products collection stores bikes, components, accessories, and clothing in a single collection. Each product has a `productType` discriminator field and type-specific `specs`.

1. View the distinct product types:

    ```javascript
    db.products.distinct("productType")
    ```

    You should see: `bike`, `component`, `accessory`, `clothing`.

1. Compare the specs structure across product types:

    ```javascript
    // Bike specs include frameSize, gears, brakeType
    db.products.findOne(
      { productType: "bike" },
      { name: 1, productType: 1, specs: 1, price: 1, category: 1, tags: 1,  _id: 0 }
    )
    ```

    ```javascript
    // Clothing specs include size, material, fit
    db.products.findOne(
      { productType: "clothing" },
      { name: 1, productType: 1, specs: 1, price: 1, category: 1, tags: 1,  _id: 0 }
    )
    ```

    Notice that both documents share common fields (`name`, `price`, `category`, `tags`) but have different `specs` structures. This difference is the inheritance pattern: one collection, multiple document shapes.

1. Query across all product types and group by type:

    ```javascript
    db.products.aggregate([
      {
        $group: {
          _id: "$productType",
          count: { $sum: 1 },
          avgPrice: { $avg: "$price" },
          minPrice: { $min: "$price" },
          maxPrice: { $max: "$price" }
        }
      },
      { $sort: { count: -1 } }
    ])
    ```

    The cross-type aggregation works because all products live in one collection. There's no need to query separate collections and merge results.

    The inheritance pattern lets you store related but structurally different documents together. This difference allows for cross-type queries for free, at the cost of handling multiple document shapes in your application code.

## Explore the computed pattern (reviewSummary)

Each product has a `reviewSummary` field with precomputed `averageRating` and `totalCount`. These values are calculated when reviews are written, not when the product page is loaded.

1. View a product's precomputed review statistics:

    ```javascript
    db.products.findOne(
      { "reviewSummary.totalCount": { $gt: 5 } },
      { name: 1, reviewSummary: 1, _id: 0 }
    )
    ```

    The product page displays these stats instantly. No aggregation pipeline is needed on every page view.

1. Verify the computed values match the actual data by comparing with the reviews collection:

    ```javascript
    var product = db.products.findOne({ "reviewSummary.totalCount": { $gt: 5 }, isOutlier: { $ne: true } })

    var actual = db.reviews.aggregate([
      { $match: { productId: product._id } },
      { $group: { _id: null, count: { $sum: 1 }, avgRating: { $avg: "$rating" } } }
    ]).toArray()[0]

    print("Product: " + product.name)
    print("Computed count: " + product.reviewSummary.totalCount + " | Actual: " + actual.count)
    print("Computed avg: " + product.reviewSummary.averageRating + " | Actual: " + Math.round(actual.avgRating * 10) / 10)
    ```

1. Simulate adding a new review and updating the computed stats:

    ```javascript
    // Insert a new review
    db.reviews.insertOne({
      productId: product._id,
      customerId: db.customers.findOne()._id,
      customerName: "Test Reviewer",
      rating: 5,
      title: "Testing the computed pattern",
      body: "This review was added to demonstrate updating computed stats.",
      helpful: 0,
      verified: true,
      createdAt: new Date()
    })

    // Update the computed stats on the product
    db.products.updateOne(
      { _id: product._id },
      {
        $inc: { "reviewSummary.totalCount": 1 }
      }
    )

    print("Updated review count: " + db.products.findOne({ _id: product._id }).reviewSummary.totalCount)
    ```

1. Revert the test review and stat update to return the data to its original state:

    ```javascript
    db.reviews.deleteOne({ productId: product._id, customerName: "Test Reviewer" })
    db.products.updateOne(
      { _id: product._id },
      { $inc: { "reviewSummary.totalCount": -1 } }
    )

    print("Reverted review count: " + db.products.findOne({ _id: product._id }).reviewSummary.totalCount)
    ```

    The computed pattern trades write-time complexity for read-time speed. You pay a small cost on every review insert to keep the summary current, but every product page load avoids an expensive aggregation.

## Explore the approximation pattern (engagement metrics)

Products have an `approximateMetrics` field with `viewCount`, `wishlistCount`, and `cartAdditions`. These counters use the approximation pattern. They're flushed periodically rather than updated on every event, reducing write load by ~99%.

1. To see the current state, examine the approximation fields on a specific product:

    ```javascript
    db.products.findOne(
      { name: "Mountain-100 Silver, 44" },
      { name: 1, approximateMetrics: 1, _id: 0 }
    )
    ```

    Notice the `lastFlushed` timestamp and the current `viewCount`. In production, the application buffers view events in memory rather than writing each one to the database individually. The `lastFlushed` field records when the last batch was written.

1. Record the current view count and last flush time, then simulate a batch flush of 100 page views using `$inc`:

    ```javascript
    var before = db.products.findOne({ name: "Mountain-100 Silver, 44" })
    print("View count before flush: " + before.approximateMetrics.viewCount)
    print("Last flushed before: " + before.approximateMetrics.lastFlushed)
    ```

    ```javascript
    db.products.updateOne(
      { name: "Mountain-100 Silver, 44" },
      {
        $inc: { "approximateMetrics.viewCount": 100 },
        $set: { "approximateMetrics.lastFlushed": new Date() }
      }
    )
    ```

    Instead of 100 individual writes (one per page view), this single `$inc` operation adds the batch total.

1. Verify the update by checking the same product:

    ```javascript
    db.products.findOne(
      { name: "Mountain-100 Silver, 44" },
      { name: 1, "approximateMetrics.viewCount": 1, "approximateMetrics.lastFlushed": 1, _id: 0 }
    )
    ```

    The `viewCount` increased by 100 and `lastFlushed` reflects the current time. In production, these counts are intentionally approximate because exact real-time counts would require a write for every single page view.

    The approximation pattern sacrifices precision for throughput. When the exact number doesn't matter (page views, likes, impressions), batching updates into periodic flushes can reduce write operations by orders of magnitude.

## Explore the extended reference pattern (orders)

Orders embed `customerName` and `customerEmail` from the customer document. This embedding avoids a `$lookup` to the `customers` collection for every order page display.

1. Examine an order's extended customer reference:

    ```javascript
    db.orders.findOne(
      { isFullyLoaded: true },
      { customerName: 1, customerEmail: 1, customerId: 1, total: 1, _id: 0 }
    )
    ```

    The order page can display "Order for Haladhar Keot (haladhar@contoso.com)" without querying the `customers` collection.

1. Compare embedded vs. in the full customer document:

    ```javascript
    var order = db.orders.findOne({ isFullyLoaded: true })

    var customer = db.customers.findOne({ _id: order.customerId })

    print("In order: " + order.customerName + ", " + order.customerEmail)
    print("In customer: " + customer.firstName + " " + customer.lastName + ", " + customer.email)
    print("Not in order: addresses, membershipTier, recentOrders, orderCount")
    ```

    The extended reference includes only display fields. Sensitive or large data stays in the `customers` collection.

    The extended reference pattern eliminates `$lookup` joins for display fields you need on every read. The trade-off is that copied fields can become stale if the source document changes, so choose fields that rarely update (like names and emails).

## Explore the schema versioning pattern

Every document in the products collection has a `schemaVersion` field. This field enables lazy or batch migration when the schema evolves.

1. Check the current schema version across all products:

    ```javascript
    db.products.aggregate([
      { $group: { _id: "$schemaVersion", count: { $sum: 1 } } }
    ])
    ```

    All products should be at `schemaVersion: 1`.

1. Simulate a schema migration by adding a new field to a subset of products and incrementing the version:

    ```javascript
    // "Migrate" mountain bikes to version 2 by adding a new field
    db.products.updateMany(
      { "category.name": "Mountain Bikes", schemaVersion: 1 },
      {
        $set: {
          schemaVersion: 2,
          "specs.suspensionCategory": "hardtail"
        }
      }
    )

    // Check version distribution
    db.products.aggregate([
      { $group: { _id: "$schemaVersion", count: { $sum: 1 } } }
    ])
    ```

    Now some products are at v2 (with `suspensionCategory`) and the rest at v1. Application code reads `schemaVersion` to handle both formats.

1. Revert the migration to return the data to its original state:

    ```javascript
    db.products.updateMany(
      { "category.name": "Mountain Bikes", schemaVersion: 2 },
      {
        $set: { schemaVersion: 1 },
        $unset: { "specs.suspensionCategory": "" }
      }
    )

    // Check version distribution
    db.products.aggregate([
      { $group: { _id: "$schemaVersion", count: { $sum: 1 } } }
    ])
    ```

    The schema versioning pattern lets you evolve your document structure without downtime. Old and new versions coexist in the same collection, and your application reads the `schemaVersion` field to handle each format correctly.

## Explore the subset pattern (customer recentOrders)

Customers embed their five most recent orders in `recentOrders`. The full order history lives in the orders collection. This section demonstrates how to maintain a bounded subset when new data arrives.

1. View the current subset and note the oldest order:

    ```javascript
    var customer = db.customers.findOne({ firstName: "Haladhar", lastName: "Keot" })

    print("Embedded recent orders: " + customer.recentOrders.length)
    print("Total order count: " + customer.orderCount)
    print("Oldest embedded order date: " + customer.recentOrders[customer.recentOrders.length - 1].orderDate)
    printjson(customer.recentOrders[0])
    ```

    The embedded subset contains `_id`, `orderDate`, `total`, `itemCount`, and `status`. Those fields are just enough for a dashboard widget. Note the date of the oldest (fifth) order because it should drop off after you add a new one.

1. Simulate a new order arriving. Use `$push` with `$sort` and `$slice` to add the new order, sort by date descending, and keep only the five most recent:

    ```javascript
    db.customers.updateOne(
      { firstName: "Haladhar", lastName: "Keot" },
      {
        $push: {
          recentOrders: {
            $each: [{
              _id: ObjectId(),
              orderDate: new Date(),
              total: 549.99,
              itemCount: 2,
              status: "processing"
            }],
            $sort: { orderDate: -1 },
            $slice: 5
          }
        },
        $inc: { orderCount: 1 }
      }
    )
    ```

    This single atomic operation adds the new order, sorts the array, and trims it to five entries. No application-side logic is needed to manage the subset size.

1. Verify the subset updated correctly:

    ```javascript
    var updated = db.customers.findOne({ firstName: "Haladhar", lastName: "Keot" })

    print("Recent orders count: " + updated.recentOrders.length)
    print("Newest order status: " + updated.recentOrders[0].status)
    print("Newest order total: $" + updated.recentOrders[0].total)
    print("Oldest embedded order date: " + updated.recentOrders[updated.recentOrders.length - 1].orderDate)
    ```

    The new "processing" order appears at the top and the oldest order from the previous step dropped off. The subset stays bounded at five entries.

1. Revert the change to return the data to its original state:

    ```javascript
    db.customers.updateOne(
      { firstName: "Haladhar", lastName: "Keot" },
      {
        $pop: { recentOrders: -1 },
        $inc: { orderCount: -1 }
      }
    )
    ```

    The subset pattern keeps frequently accessed data close to the parent document while the full history lives in a separate collection. MongoDB's `$push` with `$sort` and `$slice` maintains the bounded array atomically, so you never need application-side trimming logic.

## Explore the outlier pattern (popular products)

Most products have fewer than 15 reviews embedded in a summary. A few "outlier" products have 100+ reviews and are flagged with `isOutlier: true`. Their full reviews live in the separate `reviews` collection, while the product document embeds only the top 3.

1. Run the following script to simulate how application code fetches reviews for any product. It checks `isOutlier` and branches to the right source:

    ```javascript
    function getTopReviews(productName) {
      var product = db.products.findOne({ name: productName })
      if (!product) { print("Product not found: " + productName); return }

      print("Product: " + product.name)
      print("isOutlier: " + product.isOutlier)
      print("Total reviews: " + product.reviewSummary.totalCount)

      if (!product.isOutlier) {
        // Normal product: only summary stats are embedded (no individual reviews)
        print("Source: summary stats embedded in product document")
        print("  Average rating: " + product.reviewSummary.averageRating)
        print("  Total count: " + product.reviewSummary.totalCount)
        print("  Individual reviews: query the reviews collection")
      } else {
        // Outlier product: top 3 are embedded, full list is in the reviews collection
        print("Source: topReviews embedded + full list in reviews collection")
        product.reviewSummary.topReviews.forEach(function(r) {
          print("  [embedded top] " + r.customerName + " - " + r.rating + " stars - " + r.title)
        })
        print("  ... +" + (product.reviewSummary.totalCount - product.reviewSummary.topReviews.length) + " more in reviews collection")
      }
    }

    getTopReviews("Mountain-100 Silver, 42")
    getTopReviews("Mountain-100 Silver, 44")
    ```

    Run `getTopReviews` on both products and compare the output. The normal product reads only from the embedded document. The outlier reads the top 3 from the embedded `topReviews` array and reports that the remaining reviews are in the `reviews` collection. The application uses the same function signature for both. The branching is invisible to the caller.

1. Simulate a new highly rated review coming in for the outlier product. Add it to the `reviews` collection and update the precomputed stats:

    ```javascript
    var outlier = db.products.findOne({ name: "Mountain-100 Silver, 44" })

    db.reviews.insertOne({
      productId: outlier._id,
      customerName: "Test Reviewer",
      rating: 5,
      title: "Best mountain bike I have ever owned",
      helpful: 500,
      verified: true,
      createdAt: new Date()
    })

    db.products.updateOne(
      { _id: outlier._id },
      { $inc: { "reviewSummary.totalCount": 1 } }
    )
    ```

    With the outlier pattern, this new review goes into the `reviews` collection. Application code can then decide whether to promote it to `topReviews` based on helpfulness.

1. Simulate the batch job that refreshes `topReviews`. Query the `reviews` collection for this product's top three reviews by helpful count and compare with the currently embedded ones:

    ```javascript
    var outlier = db.products.findOne({ name: "Mountain-100 Silver, 44" })

    // What a batch refresh job would pull from the reviews collection
    var freshTop3 = db.reviews.find({ productId: outlier._id })
      .sort({ helpful: -1 })
      .limit(3)
      .toArray()

    print("Currently embedded topReviews:")
    outlier.reviewSummary.topReviews.forEach(function(r) {
      print("  " + r.customerName + " - " + r.title)
    })

    print("\nTop 3 from reviews collection (by helpful count):")
    freshTop3.forEach(function(r) {
      print("  " + r.customerName + " - helpful: " + r.helpful + " - " + r.title)
    })
    ```

    The embedded subset and the fresh query may differ. In production, a scheduled job runs this comparison and updates `topReviews` when the rankings change. This difference is the key trade-off of the outlier pattern: reads are fast, but the embedded subset may lag slightly behind the full reviews collection.

1. Revert the test review to keep the data consistent:

    ```javascript
    var outlier = db.products.findOne({ name: "Mountain-100 Silver, 44" })

    db.reviews.deleteOne({ productId: outlier._id, customerName: "Test Reviewer" })
    db.products.updateOne(
      { _id: outlier._id },
      { $inc: { "reviewSummary.totalCount": -1 } }
    )
    ```

    The outlier pattern prevents a few high-volume documents from bloating the collection. Most documents stay lightweight with just summary stats, while outliers offload their bulk data to a separate collection. The `isOutlier` flag lets your application branch to the right read path transparently.

## Explore the archive pattern (orders)

The orders collection uses the archive pattern. Recent and active orders are full documents (`isFullyLoaded: true`). Old completed orders are archived. Their full data is in `ordersArchive`, and the orders collection keeps only a metadata stub (`isFullyLoaded: false`).

1. Compare a full order with an archived stub:

    ```javascript
    // Full order (recent/active)
    var full = db.orders.findOne({ isFullyLoaded: true })
    print("Full order fields: " + Object.keys(full).join(", "))

    // Archived stub
    var stub = db.orders.findOne({ isFullyLoaded: false })
    print("Stub fields: " + Object.keys(stub).join(", "))
    ```

    The stub retains `customerName`, `orderDate`, `status`, and `total` for list views, plus `archiveId` and `archiveLocation` pointing to the full data. The full order's heavy fields (`items`, `shippingAddress`, `subtotal`, `shipping`, `tax`, `shipDate`) are gone from the stub.

1. Count the split between active and archived orders:

    ```javascript
    var fullCount = db.orders.countDocuments({ isFullyLoaded: true })
    var stubCount = db.orders.countDocuments({ isFullyLoaded: false })

    print("Full orders: " + fullCount)
    print("Archived stubs: " + stubCount)
    print("Total in orders: " + (fullCount + stubCount))
    ```

1. Simulate retrieving an archived order by checking the stub, then fetching from the archive:

    ```javascript
    var stub = db.orders.findOne({ isFullyLoaded: false })

    print("Stub shows: " + stub.customerName + ", $" + stub.total + ", " + stub.status)
    print("Archive location: " + stub.archiveLocation)

    // Fetch full data from archive using the archiveId
    var archived = db.ordersArchive.findOne({ originalOrderId: stub._id })
    print("Archived order items: " + archived.items.length)
    print("Archived shipping city: " + archived.shippingAddress.city)
    ```

    The archive pattern keeps your active collection small and fast by moving completed records to a separate collection. Stubs remain for list views and search, and the application follows the `archiveLocation` pointer only when a user requests the full details.

## Explore the bucket pattern (inventory)

The inventory collection groups daily stock events into buckets with preaggregated summaries. Each document represents one product for one day.

1. Examine a daily inventory bucket:

    ```javascript
    db.inventory.findOne(
      {},
      { productSku: 1, date: 1, dailySummary: 1, "events": { $slice: 3 }, _id: 0 }
    )
    ```

    Each bucket has a `dailySummary` (startQuantity, endQuantity, totalSold, totalRestocked) and an `events` array with individual sale/restock records.

1. Query daily sales trends for a product using the preaggregated summaries:

    ```javascript
    db.inventory.aggregate([
      { $match: { productSku: db.inventory.findOne().productSku } },
      { $sort: { date: 1 } },
      { $limit: 7 },
      {
        $project: {
          date: 1,
          sold: "$dailySummary.totalSold",
          restocked: "$dailySummary.totalRestocked",
          endStock: "$dailySummary.endQuantity",
          _id: 0
        }
      }
    ])
    ```

    This query uses the preaggregated `dailySummary` instead of unwinding individual events. This approach is much faster for dashboards.

    The bucket pattern groups time-series or event data into fixed intervals (per day, per hour) with preaggregated summaries. Dashboards and trend queries read the summaries directly, avoiding expensive `$unwind` and `$group` operations over thousands of individual events.

## Clean up

If you're finished with the exercise and don't need the data for later modules, drop the database:

```javascript
use cosmicworks
```

```javascript
db.dropDatabase()
```

> [!TIP]
> If you plan to continue with the other modules in this learning path, keep the `cosmicworks` database. Those exercises use the same dataset.

You now completed the exercise on applying schema design patterns in Azure DocumentDB. This exercise helps you understand how to use patterns like the archive pattern and the bucket pattern to optimize data storage and query performance.
