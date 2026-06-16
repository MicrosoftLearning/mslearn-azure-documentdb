---
lab:
    title: 'Lab 2 - Query and manipulate e-commerce data'
    module: 'Query and manipulate data in Azure DocumentDB'
    description: 'Practice inserting, querying, updating, and aggregating documents in an Azure DocumentDB cluster using an e-commerce scenario.'
    duration: 25
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

In this exercise, you populate your Azure DocumentDB cluster with cycling retail data, run queries to explore the catalog, update documents to reflect business operations, and build an aggregation pipeline to generate a sales summary. You use the cluster you created in the previous module's exercise.

## Prerequisites

- The Azure DocumentDB cluster from the previous module's exercise (or a new cluster if you deleted it).
- Access to MongoDB Shell through the Azure portal's Quick Start experience or a local `mongosh` installation.

## Connect to your cluster

1. In the Azure portal, open your Azure DocumentDB cluster.
1. Select **Quick start** from the navigation menu.
1. Select **Open MongoDB shell**.
1. Wait for the Cloud Shell environment to start, then accept the notice.
1. Enter your admin password when prompted.

Confirm the connection is working:

```javascript
db.runCommand({connectionStatus: 1})
```

## Seed the database

Switch to the `cosmicworks` database and populate it with product, customer, and order data for an outdoor cycling retailer.

```javascript
use cosmicworks
```

Insert products across multiple parent categories:

```javascript
db.products.insertMany([
  {
    _id: "BK-M82S-38",
    sku: "BK-M82S-38",
    name: "Mountain-100 Silver, 38",
    category: { name: "Mountain Bikes" },
    parentCategory: "Bikes",
    price: 3399.99,
    inventory: 45,
    tags: ["mountain", "aluminum", "disc-brake", "suspension"],
    specs: { frameMaterial: "aluminum", frameSize_cm: 38, wheelSize_inches: 29 },
    createdAt: new Date("2026-01-15")
  },
  {
    _id: "BK-R93R-52",
    sku: "BK-R93R-52",
    name: "Road-150 Red, 52",
    category: { name: "Road Bikes" },
    parentCategory: "Bikes",
    price: 3578.27,
    inventory: 28,
    tags: ["road", "carbon-fiber", "racing", "lightweight"],
    specs: { frameMaterial: "carbon fiber", frameSize_cm: 52, gearCount: 22 },
    createdAt: new Date("2026-01-20")
  },
  {
    _id: "HL-U509",
    sku: "HL-U509",
    name: "Sport-100 Helmet, Black",
    category: { name: "Helmets" },
    parentCategory: "Accessories",
    price: 34.99,
    inventory: 320,
    tags: ["adjustable", "reflective", "lightweight"],
    createdAt: new Date("2026-02-01")
  },
  {
    _id: "TT-M928",
    sku: "TT-M928",
    name: "Mountain Tire Tube",
    category: { name: "Tires and Tubes" },
    parentCategory: "Accessories",
    price: 4.99,
    inventory: 1500,
    tags: ["mountain", "replacement"],
    createdAt: new Date("2026-02-10")
  },
  {
    _id: "WB-T44U",
    sku: "WB-T44U",
    name: "Water Bottle - 30 oz.",
    category: { name: "Bottles and Cages" },
    parentCategory: "Accessories",
    price: 4.99,
    inventory: 600,
    tags: ["hydration", "lightweight"],
    createdAt: new Date("2026-02-15")
  },
  {
    _id: "SJ-0194-M",
    sku: "SJ-0194-M",
    name: "Short-Sleeve Classic Jersey, M",
    category: { name: "Jerseys" },
    parentCategory: "Clothing",
    price: 53.99,
    inventory: 220,
    tags: ["breathable", "lightweight"],
    specs: { size: "M", fabric: "moisture-wicking polyester" },
    createdAt: new Date("2026-03-01")
  }
])
```

Verify the insert produced six documents:

```javascript
db.products.countDocuments()
```

Now insert customer records:

```javascript
db.customers.insertMany([
  {
    _id: "CUST-001",
    firstName: "Alex",
    lastName: "Sanchez",
    email: "alex@adventure-works.com",
    membership: "premium",
    createdAt: new Date("2025-06-15")
  },
  {
    _id: "CUST-002",
    firstName: "Jordan",
    lastName: "Khan",
    email: "jordan@adventure-works.com",
    membership: "standard",
    createdAt: new Date("2025-11-20")
  },
  {
    _id: "CUST-003",
    firstName: "Sam",
    lastName: "Patel",
    email: "sam@adventure-works.com",
    membership: "premium",
    createdAt: new Date("2026-01-05")
  }
])
```

Finally, insert order records that reference customers and products. Each order embeds an `items` array of line items:

```javascript
db.orders.insertMany([
  {
    _id: "ORD-001",
    customerId: "CUST-001",
    items: [{ sku: "BK-M82S-38", quantity: 1, price: 3399.99 }],
    status: "delivered",
    orderDate: new Date("2026-03-10")
  },
  {
    _id: "ORD-002",
    customerId: "CUST-001",
    items: [{ sku: "HL-U509", quantity: 1, price: 34.99 }],
    status: "delivered",
    orderDate: new Date("2026-03-10")
  },
  {
    _id: "ORD-003",
    customerId: "CUST-002",
    items: [{ sku: "TT-M928", quantity: 3, price: 4.99 }],
    status: "delivered",
    orderDate: new Date("2026-03-12")
  },
  {
    _id: "ORD-004",
    customerId: "CUST-003",
    items: [{ sku: "BK-R93R-52", quantity: 1, price: 3578.27 }],
    status: "delivered",
    orderDate: new Date("2026-03-15")
  },
  {
    _id: "ORD-005",
    customerId: "CUST-002",
    items: [{ sku: "SJ-0194-M", quantity: 2, price: 53.99 }],
    status: "pending",
    orderDate: new Date("2026-03-18")
  },
  {
    _id: "ORD-006",
    customerId: "CUST-001",
    items: [{ sku: "WB-T44U", quantity: 5, price: 4.99 }],
    status: "delivered",
    orderDate: new Date("2026-03-20")
  },
  {
    _id: "ORD-007",
    customerId: "CUST-003",
    items: [{ sku: "SJ-0194-M", quantity: 1, price: 53.99 }],
    status: "delivered",
    orderDate: new Date("2026-03-22")
  }
])
```

## Query and filter products

Run these queries to explore the product catalog.

Find all bikes priced over $1,000:

```javascript
db.products.find(
  { parentCategory: "Bikes", price: { $gt: 1000 } },
  { name: 1, price: 1, _id: 0 }
)
```

You should see two products: Mountain-100 Silver, 38 ($3,399.99) and Road-150 Red, 52 ($3,578.27).

Find products that have a `specs` field and the "lightweight" tag:

```javascript
db.products.find({
  specs: { $exists: true },
  tags: "lightweight"
})
```

This query returns Road-150 Red, 52 and Short-Sleeve Classic Jersey, M because both have an embedded `specs` document and a "lightweight" tag.

Find the three cheapest products using sort, limit, and projection:

```javascript
db.products.find(
  {},
  { name: 1, price: 1, parentCategory: 1, _id: 0 }
).sort({ price: 1 }).limit(3)
```

You should see Mountain Tire Tube ($4.99, Accessories), Water Bottle - 30 oz. ($4.99, Accessories), and Sport-100 Helmet, Black ($34.99, Accessories).

## Update product data

Simulate business operations by updating documents.

A shipment arrived. The Mountain Tire Tube currently has 1,500 units in stock. Increase the inventory by 500:

```javascript
db.products.updateOne(
  { _id: "TT-M928" },
  { $inc: { inventory: 500 } }
)
```

Verify the update:

```javascript
db.products.findOne({ _id: "TT-M928" }, { name: 1, inventory: 1, _id: 0 })
```

The inventory should now show 2000.

The spring sale starts. Add a "sale" tag to all accessories:

```javascript
db.products.updateMany(
  { parentCategory: "Accessories" },
  { $addToSet: { tags: "sale" } }
)
```

Verify the result shows `modifiedCount: 3` and then check one of the documents:

```javascript
db.products.findOne({ _id: "HL-U509" }, { name: 1, tags: 1, _id: 0 })
```

The tags array should now include "sale" alongside the original tags.

## Build an aggregation pipeline

Build a pipeline that answers the question: "What is the total revenue and order count for each parent category, considering only delivered orders?"

```javascript
db.orders.aggregate([
  // "...considering only delivered orders"
  { $match: { status: "delivered" } },

  // Flatten each order's items array into individual line items
  { $unwind: "$items" },

  // "...for each parent category" (step 1: look up the product to get its parentCategory)
  {
    $lookup: {
      from: "products",
      localField: "items.sku",
      foreignField: "sku",
      as: "product"
    }
  },

  // Flatten the single-element product array so we can access its fields directly
  { $unwind: "$product" },

  // "total revenue and order count for each parent category"
  {
    $group: {
      _id: "$product.parentCategory",
      total_revenue: {
        $sum: { $multiply: ["$items.quantity", "$product.price"] }
      },
      order_count: { $sum: 1 }
    }
  },

  // Present results from highest to lowest revenue
  { $sort: { total_revenue: -1 } },

  // Clean up the output (returns only category, total_revenue, and order_count)
  {
    $project: {
      _id: 0,
      category: "$_id",
      total_revenue: { $round: ["$total_revenue", 2] },
      order_count: 1
    }
  }
])
```

Review the results. You should see three categories:

- **Bikes**: Revenue from orders ORD-001 (1 x $3,399.99) and ORD-004 (1 x $3,578.27), totaling $6,978.26 across two line items.
- **Accessories**: Revenue from orders ORD-002 (1 x $34.99), ORD-003 (3 x $4.99), and ORD-006 (5 x $4.99), totaling $74.91 across three line items.
- **Clothing**: Revenue from order ORD-007 (1 x $53.99), totaling $53.99 across one line item.

> [!TIP]
> If your results differ, double-check that you inserted all the seed data from the earlier steps. You can verify order count with `db.orders.countDocuments({ status: "delivered" })`, which should return 6.

You completed the exercise on querying and manipulating data in Azure DocumentDB.

## Clean up resources

If you're continuing to the next module, keep the cluster and data for later exercises. If not, delete the resource group to avoid charges:

1. In the Azure portal, search for and select **Resource groups**.
1. Select **rg-documentdb-learn**.
1. Select **Delete resource group**.
1. Enter the resource group name to confirm, then select **Delete**.

