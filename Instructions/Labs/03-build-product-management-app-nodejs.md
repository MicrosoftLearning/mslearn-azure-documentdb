---
lab:
    title: 'Lab 3 - Build a product management application - Node.js'
    module: 'Build applications with Azure DocumentDB SDKs'
    description: 'Build a console application that connects to Azure DocumentDB and performs CRUD operations using the MongoDB driver for Node.js.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

# Build a product management application - Node.js

In this exercise, you build a console application that connects to your Azure DocumentDB cluster and performs CRUD operations against the product catalog. You use Visual Studio Code and its integrated terminal to create, edit, and run the application.

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/) installed. If not already installed, follow the [download and install instructions](https://code.visualstudio.com/docs/setup/setup-overview).
- An Azure DocumentDB cluster with public access enabled (created in a previous module)
- Your cluster connection string
- Node.js 22 or later. If not installed, follow the [Node.js installation guide](https://nodejs.org/en/download).

## Set up your connection string

Open Visual Studio Code and open a new terminal (**Terminal > New Terminal**). Set the connection string as an environment variable. Replace the placeholders with your cluster details.

macOS/Linux (bash):

```bash
export AZURE_DOCUMENTDB_CONNECTION_STRING="mongodb+srv://<username>:<password>@<cluster-name>.global.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
```

Windows (PowerShell):

```powershell
$env:AZURE_DOCUMENTDB_CONNECTION_STRING = "mongodb+srv://<username>:<password>@<cluster-name>.global.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000"
```

## Create the project

In the Visual Studio Code integrated terminal, navigate to a folder where you want to create the project, then create and enter a new project directory:

```bash
mkdir product-manager
cd product-manager
```

Now set up the project.

1. Initialize the project:

    ```bash
    npm init -y
    ```

1. Install the MongoDB driver:

    ```bash
    npm install mongodb
    ```

1. Create and open a new JavaScript file in Visual Studio Code:

    ```bash
    code app.js
    ```

## Write the application

You build the application incrementally, adding one section at a time so you can run and verify each step before moving on.

### Step 1: Connect and verify

Add the following code to `app.js`. This code creates a client, verifies the connection, and sets up placeholders for the remaining steps.

```javascript
const { MongoClient } = require('mongodb');

async function main() {
    const connectionString = process.env.AZURE_DOCUMENTDB_CONNECTION_STRING;
    const client = new MongoClient(connectionString);

    try {
        await client.connect();
        await client.db('admin').command({ ping: 1 });
        console.log('Connected to Azure DocumentDB\n');

        const db = client.db('cosmicworks');
        const products = db.collection('products');

        // Step 2: Insert three sample products (add code here)

        // Step 3: Find the helmet by SKU (add code here)

        // Step 4: Query products under $100 (add code here)

        // Step 5: Update the helmet inventory and tags (add code here)

        // Step 6: Delete the jersey (add code here)

    } finally {
        await client.close();
    }
}

main().catch(console.error);
```

Save the file and run it:

```bash
node app.js
```

You should see:

```output
Connected to Azure DocumentDB
```

### Step 2: Insert three sample products

Replace the `// Step 2` comment with the following code:

```javascript
        // Step 2: Insert three sample products
        const sampleProducts = [
            {
                sku: 'BK-M82S-38', name: 'Mountain-100 Silver, 38', price: 3399.99,
                category: { name: 'Mountain Bikes' },
                tags: ['mountain', 'aluminum', 'high-performance'], inventory: 45
            },
            {
                sku: 'HL-U509', name: 'Sport-100 Helmet, Black', price: 34.99,
                category: { name: 'Helmets' },
                tags: ['adjustable', 'reflective', 'lightweight'], inventory: 320
            },
            {
                sku: 'SJ-0194-M', name: 'Short-Sleeve Classic Jersey, M', price: 53.99,
                category: { name: 'Jerseys' },
                tags: ['breathable', 'summer'], inventory: 185
            }
        ];

        await products.deleteMany({});  // Clear existing data
        const insertResult = await products.insertMany(sampleProducts);
        console.log(`Inserted ${insertResult.insertedCount} products`);
```

Save and run. You should now see:

```output
Connected to Azure DocumentDB

Inserted 3 products
```

### Step 3: Find the helmet by SKU

Replace the `// Step 3` comment with the following code:

```javascript
        // Step 3: Find the helmet by SKU
        const helmet = await products.findOne({ sku: 'HL-U509' });
        console.log(`\nFound product: ${helmet.name}. $${helmet.price}`);
```

Save and run. The new output line should be:

```output
Found product: Sport-100 Helmet, Black. $34.99
```

### Step 4: Query products under $100

Replace the `// Step 4` comment with the following code:

```javascript
        // Step 4: Query products under $100
        const affordable = await products.find({ price: { $lt: 100 } }).toArray();
        console.log('\nProducts under $100:');
        for (const product of affordable) {
            console.log(`  ${product.name}: $${product.price}`);
        }
```

Save and run. The new output lines should be:

```output
Products under $100:
  Sport-100 Helmet, Black: $34.99
  Short-Sleeve Classic Jersey, M: $53.99
```

### Step 5: Update the helmet inventory and tags

Replace the `// Step 5` comment with the following code. This code retrieves the helmet before and after the update so you can compare the changes.

```javascript
        // Step 5: Update the helmet inventory and tags
        const before = await products.findOne({ sku: 'HL-U509' });
        console.log(`\nBefore update - inventory: ${before.inventory}, tags: ${JSON.stringify(before.tags)}`);

        await products.updateOne(
            { sku: 'HL-U509' },
            { $inc: { inventory: -5 }, $addToSet: { tags: 'popular' } }
        );

        const after = await products.findOne({ sku: 'HL-U509' });
        console.log(`After update  - inventory: ${after.inventory}, tags: ${JSON.stringify(after.tags)}`);
```

Save and run. The new output lines should be:

```output
Before update - inventory: 320, tags: ["adjustable","reflective","lightweight"]
After update  - inventory: 315, tags: ["adjustable","reflective","lightweight","popular"]
```

### Step 6: Delete the jersey

Replace the `// Step 6` comment with the following code. This displays the product count before and after the delete so you can confirm the operation.

```javascript
        // Step 6: Delete the jersey
        const countBefore = await products.countDocuments({});
        console.log(`\nProducts before delete: ${countBefore}`);

        const deleteResult = await products.deleteOne({ sku: 'SJ-0194-M' });
        console.log(`Deleted ${deleteResult.deletedCount} product`);

        const countAfter = await products.countDocuments({});
        console.log(`Products after delete: ${countAfter}`);
```

Save and run. The new output lines should be:

```output
Products before delete: 3
Deleted 1 product
Products after delete: 2
```

## Verify the final output

After completing all six steps, run your application one final time. The complete output should look similar to:

```output
Connected to Azure DocumentDB

Inserted 3 products

Found product: Sport-100 Helmet, Black. $34.99

Products under $100:
  Sport-100 Helmet, Black: $34.99
  Short-Sleeve Classic Jersey, M: $53.99

Before update - inventory: 320, tags: ['adjustable', 'reflective', 'lightweight']
After update  - inventory: 315, tags: ['adjustable', 'reflective', 'lightweight', 'popular']

Products before delete: 3
Deleted 1 product
Products after delete: 2
```

The output confirms that your application successfully connected to Azure DocumentDB and performed all four CRUD operations: insert, read, update, and delete.

## Clean up

To remove the sample data from your cluster, open MongoDB Shell (`mongosh`) or add the following line to your application and run it one more time:

```javascript
await db.dropCollection('products');
```

If you no longer need the project files, you can also delete the `product-manager` directory from your machine.

You now have a working application that connects to Azure DocumentDB and performs all four CRUD operations: inserting documents, querying with filters, updating fields with operators like `$inc` and `$addToSet`, and deleting by filter. These operations are the same patterns you use to build production applications on Azure DocumentDB, regardless of which programming language you choose.
