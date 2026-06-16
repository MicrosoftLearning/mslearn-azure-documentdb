---
lab:
    title: 'Lab 3 - Build a product management application - Python'
    module: 'Build applications with Azure DocumentDB SDKs'
    description: 'Build a console application that connects to Azure DocumentDB and performs CRUD operations using the MongoDB driver for Python.'
    duration: 30
    level: 300
    islab: true
    status: 'in-development'
    targetDate: '2026-06-30'
---

In this exercise, you build a console application that connects to your Azure DocumentDB cluster and performs CRUD operations against the product catalog. You use Visual Studio Code and its integrated terminal to create, edit, and run the application.

## Prerequisites

- [Visual Studio Code](https://code.visualstudio.com/) installed. If not already installed, follow the [download and install instructions](https://code.visualstudio.com/docs/setup/setup-overview).
- An Azure DocumentDB cluster with public access enabled (created in a previous module)
- Your cluster connection string
- Python 3.12 or later. If not installed, follow the [Python installation guide](https://wiki.python.org/moin/BeginnersGuide/Download).

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

1. Set up a virtual environment.

    macOS/Linux (bash):

    ```bash
    python -m venv venv
    source venv/bin/activate
    ```

    Windows (PowerShell):

    ```powershell
    py -m venv venv
    venv\Scripts\activate
    ```

1. Install PyMongo:

    ```bash
    pip install pymongo
    ```

1. Create and open a new Python file in Visual Studio Code:

    ```bash
    code app.py
    ```

## Write the application

You build the application incrementally, adding one section at a time so you can run and verify each step before moving on.

### Step 1: Connect and verify

Add the following code to `app.py`. This code creates a client, verifies the connection, and sets up placeholders for the remaining steps.

```python
import os
from pymongo import MongoClient
from pymongo.errors import ConnectionFailure, DuplicateKeyError

def main():
    connection_string = os.environ["AZURE_DOCUMENTDB_CONNECTION_STRING"]

    with MongoClient(connection_string) as client:
        # Verify connection
        client.admin.command("ping")
        print("Connected to Azure DocumentDB\n")

        db = client["cosmicworks"]
        products = db["products"]

        # Step 2: Insert three sample products (add code here)

        # Step 3: Find the helmet by SKU (add code here)

        # Step 4: Query products under $100 (add code here)

        # Step 5: Update the helmet inventory and tags (add code here)

        # Step 6: Delete the jersey (add code here)

if __name__ == "__main__":
    main()
```

Save the file and run it:

```bash
python app.py
```

You should see:

```output
Connected to Azure DocumentDB
```

### Step 2: Insert products

Replace the `# Step 2` comment with the following code. Make sure the indentation matches the surrounding code (two levels inside `with` and `def`).

```python
        # Step 2: Insert three sample products
        sample_products = [
            {
                "sku": "BK-M82S-38",
                "name": "Mountain-100 Silver, 38",
                "price": 3399.99,
                "category": {"name": "Mountain Bikes"},
                "tags": ["mountain", "aluminum", "high-performance"],
                "inventory": 45
            },
            {
                "sku": "HL-U509",
                "name": "Sport-100 Helmet, Black",
                "price": 34.99,
                "category": {"name": "Helmets"},
                "tags": ["adjustable", "reflective", "lightweight"],
                "inventory": 320
            },
            {
                "sku": "SJ-0194-M",
                "name": "Short-Sleeve Classic Jersey, M",
                "price": 53.99,
                "category": {"name": "Jerseys"},
                "tags": ["breathable", "summer"],
                "inventory": 185
            }
        ]

        products.delete_many({})  # Clear existing data
        result = products.insert_many(sample_products)
        print(f"Inserted {len(result.inserted_ids)} products")
```

Save and run. You should now see:

```output
Connected to Azure DocumentDB

Inserted 3 products
```

### Step 3: Find a product by SKU

Replace the `# Step 3` comment with the following code:

```python
        # Step 3: Find the helmet by SKU
        helmet = products.find_one({"sku": "HL-U509"})
        print(f"\nFound product: {helmet['name']}. ${helmet['price']}")
```

Save and run. The new output line should be:

```output
Found product: Sport-100 Helmet, Black. $34.99
```

### Step 4: Query products by price range

Replace the `# Step 4` comment with the following code:

```python
        # Step 4: Query products under $100
        print("\nProducts under $100:")
        for product in products.find({"price": {"$lt": 100}}):
            print(f"  {product['name']}: ${product['price']}")
```

Save and run. The new output lines should be:

```output
Products under $100:
  Sport-100 Helmet, Black: $34.99
  Short-Sleeve Classic Jersey, M: $53.99
```

### Step 5: Update inventory

Replace the `# Step 5` comment with the following code. This code retrieves the helmet before and after the update so you can compare the changes.

```python
        # Step 5: Update the helmet inventory and tags
        before = products.find_one({"sku": "HL-U509"})
        print(f"\nBefore update - inventory: {before['inventory']}, tags: {before['tags']}")

        products.update_one(
            {"sku": "HL-U509"},
            {"$inc": {"inventory": -5}, "$addToSet": {"tags": "popular"}}
        )

        after = products.find_one({"sku": "HL-U509"})
        print(f"After update  - inventory: {after['inventory']}, tags: {after['tags']}")
```

Save and run. The new output lines should be:

```output
Before update - inventory: 320, tags: ['adjustable', 'reflective', 'lightweight']
After update  - inventory: 315, tags: ['adjustable', 'reflective', 'lightweight', 'popular']
```

### Step 6: Delete a product

Replace the `# Step 6` comment with the following code. This displays the product count before and after the delete so you can confirm the operation.

```python
        # Step 6: Delete a jersey product
        print(f"\nProducts before delete: {products.count_documents({})}")

        delete_result = products.delete_one({"sku": "SJ-0194-M"})
        print(f"Deleted {delete_result.deleted_count} product")
        print(f"Products after delete: {products.count_documents({})}")
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

```python
db.drop_collection("products")
```

If you no longer need the project files, you can also delete the `product-manager` directory from your machine.

You now have a working application that connects to Azure DocumentDB and performs all four CRUD operations: inserting documents, querying with filters, updating fields with operators like `$inc` and `$addToSet`, and deleting by filter. These operations are the same patterns you use to build production applications on Azure DocumentDB, regardless of which programming language you choose.
