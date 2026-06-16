---
lab:
    title: 'Lab 4 - Design a relationship model for an e-commerce platform'
    module: 'Model data relationships in Azure DocumentDB'
    description: 'Apply relationship modeling patterns to design and implement a data model for an e-commerce platform in Azure DocumentDB, using embedding, referencing, subset, and many-to-many patterns.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

# Design a relationship model for an e-commerce platform

In this exercise, you explore the data relationship modeling approaches from this module using the Cosmicworks e-commerce dataset. You load a real dataset into your Azure DocumentDB cluster and examine how one-to-one, one-to-many, many-to-many, and hybrid patterns are implemented across its collections. You then run queries that validate each relationship pattern and modify relationships in the data.

> [!NOTE]
> This exercise assumes you have an Azure DocumentDB cluster with public access enabled. If you need to set up a cluster, refer to the earlier module in this learning path: *Create and configure an Azure DocumentDB cluster*.

## Prerequisites

Before you begin this exercise, ensure you have the following installed and configured in your environment:

- [Visual Studio Code](https://code.visualstudio.com/) installed
- [MongoDB Shell (mongosh)](https://www.mongodb.com/try/download/shell) installed
- [MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools) installed (provides the `mongoimport` command)
- An Azure DocumentDB cluster with your admin credentials

## Set up the working environment

Skip this section if you already uploaded your collections to the database.

The `Cosmicworks` dataset contains collections that demonstrate the data modeling patterns covered in this module: customers with embedded addresses, orders with embedded line items and extended customer references, products with review subsets, and products linked to categories in a many-to-many relationship.

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

## Connect with mongosh

Once your Azure DocumentDB has the sample collections loaded, it's time to connect to the cluster and start working:

1. If you haven't already, open **Visual Studio Code** and open a terminal (**Terminal > New Terminal**).

1. In the same Visual Studio Code terminal, connect to your cluster using mongosh:

    macOS/Linux (bash):

    ```bash
    mongosh "$URI"
    ```

    Windows (PowerShell):

    ```powershell
    mongosh $env:URI
    ```

    Enter your password when prompted.

1. Switch to the `cosmicworks` database and verify the collections loaded:

    ```javascript
    use cosmicworks;
    ```

    Run each command separately to verify the document counts:

    ```javascript
    db.products.countDocuments()
    ```

    ```javascript
    db.customers.countDocuments()
    ```

    ```javascript
    db.orders.countDocuments()
    ```

    ```javascript
    db.reviews.countDocuments()
    ```

    ```javascript
    db.categories.countDocuments()
    ```

    You should see approximately 1,000 products, 249 customers, 4,000 orders, 5,000 reviews, and 37 categories.

## Explore embedded one-to-one and one-to-few relationships

Customers in the dataset embedded addresses (one-to-few, bounded at 1-2) and embedded profile information (one-to-one). These data sets are always accessed together and change infrequently, making embedding the right choice.

1. Examine a customer document to see the embedded addresses:

    ```javascript
    db.customers.findOne(
      { firstName: "Haladhar" },
      { firstName: 1, lastName: 1, email: 1, addresses: 1, membershipTier: 1 }
    )
    ```

    Notice that the addresses array contains 1-2 address objects directly inside the customer document. A single query returns everything a profile page needs.

1. Query a nested address field using dot notation:

    ```javascript
    db.customers.find(
      { "addresses.city": "Seattle" },
      { firstName: 1, lastName: 1, "addresses.city": 1, "addresses.state": 1 }
    ).limit(5)
    ```

    This query demonstrates the advantage of embedding: you can filter and project on nested fields without a second query.

1. Add a new address to an existing customer using the `$push` operator:

    ```javascript
    db.customers.updateOne(
      { firstName: "Haladhar", lastName: "Keot" },
      {
        $push: {
          addresses: {
            label: "vacation",
            street: "789 Beach Road",
            city: "Miami",
            state: "FL",
            country: "US",
            zipCode: "33139"
          }
        }
      }
    )
    ```

1. Verify the address was added:

    ```javascript
    db.customers.findOne(
      { firstName: "Haladhar", lastName: "Keot" },
      { firstName: 1, lastName: 1, addresses: 1 }
    )
    ```

    You should see three addresses: the original home address plus the new vacation address you added.

## Explore embedded order items (one-to-many with strong ownership)

Orders contain embedded line items with strong ownership, an embedded shipping address (snapshot at order time), and an extended reference to the customer (name and email embedded for display without a join). Items don't exist without the order.

1. Find a full order document (one that isn't an archived stub) and examine its structure:

    ```javascript
    db.orders.findOne(
      { isFullyLoaded: true },
      {
        customerName: 1,
        customerEmail: 1,
        status: 1,
        items: 1,
        shippingAddress: 1,
        subtotal: 1,
        shipping: 1,
        tax: 1,
        total: 1
      }
    )
    ```

    Notice three relationship patterns in this single document:
    - **Extended reference:** `customerName` and `customerEmail` are denormalized from the customer, so no `$lookup` is needed to display the customer name on the order page.
    - **Embedded items:** The `items` array contains line items with `sku`, `name`, `price`, and `quantity`. These fields have strong ownership and exist only as part of this order.
    - **Embedded shipping address:** A snapshot of the customer's address at order time. Even if the customer later updates their address, this order preserves the original delivery location.

1. Query orders by a specific product SKU using the embedded items array:

    ```javascript
    db.orders.find(
      { "items.sku": "BK-M100S-44", isFullyLoaded: true },
      { customerName: 1, "items.sku": 1, "items.name": 1, total: 1 }
    ).limit(3)
    ```

    This multikey query on the embedded `items.sku` field works because the items array is bounded (1-5 items per order).

1. Calculate total revenue for a specific customer using an aggregation pipeline on the embedded data:

    ```javascript
    db.orders.aggregate([
      { $match: { customerName: "Haladhar Keot", isFullyLoaded: true } },
      {
        $group: {
          _id: "$customerName",
          totalOrders: { $sum: 1 },
          totalRevenue: { $sum: "$total" },
          avgOrderValue: { $avg: "$total" }
        }
      }
    ])
    ```

    All the data needed for this calculation comes from the orders collection alone because the customer name and financial totals are embedded. No joins are required.

## Explore the subset pattern (product reviews)

Products use the subset pattern for reviews: the `reviewSummary` field contains precomputed statistics (average rating, total count), while the full reviews live in a separate `reviews` collection. Some products (outliers with 100+ reviews) also have `topReviews` embedded.

1. Examine a product's review summary, which is what the product listing page displays:

    ```javascript
    db.products.findOne(
      { "reviewSummary.totalCount": { $gt: 5 } },
      { name: 1, price: 1, "reviewSummary.averageRating": 1, "reviewSummary.totalCount": 1 }
    )
    ```

    The product page loads with one query. No need to aggregate the reviews collection.

1. Find an outlier product that embedded top reviews:

    ```javascript
    db.products.findOne(
      { isOutlier: true },
      { name: 1, "reviewSummary.totalCount": 1, "reviewSummary.topReviews": 1 }
    )
    ```

    Outlier products embed their top three reviews for immediate display. This embedding is the subset pattern: the most useful reviews are embedded. All the reviews are in the `reviews` collection.

1. Simulate a "Show all reviews" action by querying the separate reviews collection:

    ```javascript
    var product = db.products.findOne({ isOutlier: true })

    db.reviews.find({ productId: product._id })
      .sort({ helpful: -1 })
      .limit(5)
    ```

    This second query only runs when the user selects "Show all reviews." For the initial page load, the embedded `topReviews` is sufficient.

1. Verify that the precomputed `reviewSummary.totalCount` matches the actual review count:

    ```javascript
    var product = db.products.findOne({ isOutlier: true })

    var actualCount = db.reviews.countDocuments({ productId: product._id })

    print("Product: " + product.name)
    print("Embedded count: " + product.reviewSummary.totalCount)
    print("Actual review count: " + actualCount)
    ```

    The counts should match. This example demonstrates the computed pattern working alongside the subset pattern.

## Explore the customer's recent orders (subset pattern)

Customers also use the subset pattern: `recentOrders` embeds the five most recent orders (summary only: _id, date, total, status), while the full orders live in the orders collection.

1. View a customer's recent orders, which is what the customer dashboard displays:

    ```javascript
    db.customers.findOne(
      { firstName: "Haladhar", lastName: "Keot" },
      { firstName: 1, lastName: 1, orderCount: 1, recentOrders: 1 }
    )
    ```

    The dashboard loads with one query. The `recentOrders` subset gives "Your last five orders" without querying the orders collection.

1. Compare the subset to the full order history:

    ```javascript
    var customer = db.customers.findOne({ firstName: "Haladhar", lastName: "Keot" })

    print("Recent orders embedded: " + customer.recentOrders.length)
    print("Total order count: " + customer.orderCount)

    // Full order history requires a separate query
    db.orders.find({ customerId: customer._id })
      .sort({ orderDate: -1 })
      .limit(10)
    ```

## Explore many-to-many relationships (products and categories)

Products and categories have a many-to-many relationship. Products embed their category as an extended reference (category `_id` and `name`), while categories exist as a separate collection.

1. Find a product and examine its embedded category:

    ```javascript
    db.products.findOne(
      { name: "Mountain-100 Silver, 44" },
      { name: 1, "category._id": 1, "category.name": 1, tags: 1 }
    )
    ```

    The product embeds the category `_id` and `name` as an extended reference that avoids a `$lookup` for product listings.

1. Query all products in a specific category using the embedded reference:

    ```javascript
    db.products.find(
      { "category.name": "Mountain Bikes" },
      { name: 1, price: 1, "category.name": 1 }
    ).sort({ price: 1 }).limit(5)
    ```

    Because the category name is denormalized into each product, this query doesn't need a join.

1. Count products per category using an aggregation:

    ```javascript
    db.products.aggregate([
      {
        $group: {
          _id: "$category.name",
          count: { $sum: 1 },
          avgPrice: { $avg: "$price" }
        }
      },
      { $sort: { count: -1 } },
      { $limit: 10 }
    ])
    ```

    This pipeline works entirely on the products collection because each product carries its category name.

1. Find all products that share specific tags (tags as a many-to-many via embedded array):

    ```javascript
    db.products.find(
      { tags: { $all: ["mountain", "aluminum"] } },
      { name: 1, price: 1, tags: 1 }
    ).limit(5)
    ```

    Tags demonstrate a simpler many-to-many: each product embeds tag names directly. The `$all` operator finds products matching multiple tags.

## Validate relationships with a cross-collection aggregation

To calculate revenue by product category, run an aggregation pipeline that joins orders with products. This pipeline validates that the relationship model supports analytical queries.

```javascript
db.orders.aggregate([
  { $match: { isFullyLoaded: true } },
  { $unwind: "$items" },
  {
    $lookup: {
      from: "products",
      localField: "items.productId",
      foreignField: "_id",
      as: "productInfo"
    }
  },
  { $unwind: "$productInfo" },
  {
    $group: {
      _id: "$productInfo.category.name",
      totalRevenue: { $sum: { $multiply: ["$items.price", "$items.quantity"] } },
      totalItemsSold: { $sum: "$items.quantity" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } },
  { $limit: 10 }
])
```

You should see revenue grouped by category, with Mountain Bikes and Road Bikes likely at the top. This pipeline works because:

- Orders embed items with productId (reference for join) and price (denormalized for computation)
- Products embed category.name (denormalized for grouping)

You now reviewed everything needed to understand how to model relationships in Azure DocumentDB.

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

You now completed the exercise on designing relationship models in Azure DocumentDB. This exercise helps you to understand how to model relationships between entities, denormalize data for performance, and use aggregation pipelines for analytical queries.
