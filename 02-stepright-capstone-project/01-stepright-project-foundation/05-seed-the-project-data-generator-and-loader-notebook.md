---
title: "Seed the Project - Data Generator and Loader Notebook"
parent: "StepRight - Project Foundation"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/01-stepright-project-foundation/seed-the-project-data-generator-and-loader-notebook/
read_minutes: 8
---

# Seed the Project - Data Generator and Loader Notebook
{: .no_toc }

*Estimated read: 8 min*

Lecture 4 set the targets; this lecture writes the code -- a local Python script that generates
batch zero with [Faker](https://faker.readthedocs.io/), and a Databricks notebook that loads it
into `dev.step_right`.

## The generator script

`scripts/generate_batch.py` runs locally (not on a Databricks cluster) and writes plain
CSV/JSON files -- deliberately outside Databricks, since a real source system doesn't run inside
your Lakehouse either:

```python
# scripts/generate_batch.py
import csv
import json
import random
import uuid
from datetime import datetime, timedelta
from faker import Faker

fake = Faker()
Faker.seed(42)  # reproducible batch zero across runs

CATEGORIES = ["Running", "Trail", "Basketball", "Casual", "Training", "Sandals"]

def generate_customers(n=2000):
    customers = []
    for i in range(n):
        customers.append({
            "customer_id": f"CUST-{i:06d}",
            "first_name": fake.first_name(),
            "last_name": fake.last_name(),
            # ~2% missing emails, matching Lecture 4's "realistic imperfection" target
            "email": fake.email() if random.random() > 0.02 else None,
            "region": random.choice(["NORTHEAST", "SOUTHEAST", "MIDWEST", "WEST"]),
            "signup_date": fake.date_between(start_date="-3y", end_date="today").isoformat(),
            "updated_at": datetime.utcnow().isoformat(),
        })
    return customers

def generate_products(n=400):
    products = []
    for i in range(n):
        products.append({
            "product_id": f"PROD-{i:05d}",
            "product_name": f"{fake.word().title()} {random.choice(CATEGORIES)} Shoe",
            "category": random.choice(CATEGORIES),
            "list_price": round(random.uniform(39.99, 189.99), 2),
            "launch_date": fake.date_between(start_date="-2y", end_date="today").isoformat(),
        })
    return products

def generate_orders_and_items(customers, products, n_orders=12000):
    orders, items = [], []
    for i in range(n_orders):
        order_id = f"ORD-{i:07d}"
        # ~1% reference a customer not yet in the feed -- CDC ordering/timing gap
        cust = random.choice(customers)["customer_id"] if random.random() > 0.01 else "CUST-999999"
        orders.append({
            "order_id": order_id,
            "customer_id": cust,
            "order_date": fake.date_between(start_date="-6M", end_date="today").isoformat(),
            "status": random.choice(["PLACED", "SHIPPED", "DELIVERED", "CANCELLED"]),
            "updated_at": datetime.utcnow().isoformat(),
        })
        for _ in range(random.randint(1, 4)):
            # ~2% reference a product not yet in the catalog -- late catalog update
            prod = random.choice(products)["product_id"] if random.random() > 0.02 else "PROD-99999"
            items.append({
                "order_item_id": str(uuid.uuid4()),
                "order_id": order_id,
                "product_id": prod,
                "quantity": random.randint(1, 3),
                # occasional negative price, exercising Section 3's DQ rules
                "unit_price": round(random.uniform(39.99, 189.99), 2) if random.random() > 0.005 else -9.99,
                "discount_pct": round(random.uniform(0, 30), 1),
            })
    return orders, items

if __name__ == "__main__":
    customers = generate_customers()
    products = generate_products()
    orders, items = generate_orders_and_items(customers, products)

    for name, rows in [("customers", customers), ("products", products),
                        ("orders", orders), ("order_items", items)]:
        with open(f"batch_zero_{name}.json", "w") as f:
            json.dump(rows, f)
    print(f"Batch zero: {len(customers)} customers, {len(products)} products, "
          f"{len(orders)} orders, {len(items)} order_items")
```

Running `python scripts/generate_batch.py` produces four JSON files locally -- the deliberate
imperfections from Lecture 4's plan (missing emails, orphaned foreign keys, negative prices) are
baked in at generation time via the `random.random() > threshold` checks, not added later.

## Uploading to the staging volume

Upload the generated files to `dev.step_right.staging` (create it first, the same way Lecture 3
created `landing`) using the Databricks CLI, standing in for however a real source system would
drop files onto a shared location:

```bash
databricks fs cp batch_zero_customers.json dbfs:/Volumes/dev/step_right/staging/customers/batch_zero.json
databricks fs cp batch_zero_products.json dbfs:/Volumes/dev/step_right/staging/products/batch_zero.json
databricks fs cp batch_zero_orders.json dbfs:/Volumes/dev/step_right/staging/orders/batch_zero.json
databricks fs cp batch_zero_order_items.json dbfs:/Volumes/dev/step_right/staging/order_items/batch_zero.json
```

## The loader notebook

`notebooks/load_batch_to_landing.py` runs inside Databricks and does two things: for the two
file-shaped sources bronze pipelines will read directly (`products`), it copies straight into the
matching `landing` subfolder; for the CDC-shaped sources (`customers`, `orders`, `order_items`),
it writes into `dev.raw_cdc`, simulating what the managed CDC connector would have produced:

```python
# notebooks/load_batch_to_landing.py
dbutils.widgets.text("batch_name", "batch_zero")
batch_name = dbutils.widgets.get("batch_name")

# File-shaped source: straight copy into the landing volume
dbutils.fs.cp(
    f"/Volumes/dev/step_right/staging/products/{batch_name}.json",
    f"/Volumes/dev/step_right/landing/products/{batch_name}.json",
)

# CDC-shaped sources: land as governed tables under dev.raw_cdc
for source in ["customers", "orders", "order_items"]:
    df = (
        spark.read.json(f"/Volumes/dev/step_right/staging/{source}/{batch_name}.json")
    )
    df.write.mode("append").saveAsTable(f"dev.raw_cdc.{source}_changefeed")

print(f"Loaded {batch_name} into landing volume and dev.raw_cdc")
```

Parameterizing on `batch_name` rather than hardcoding `batch_zero` means this same notebook loads
every incremental batch Lecture 4 planned for later sections -- run it again with
`batch_name=batch_one` once Section 2's pipelines exist and need something new to process.

## Verify the seed landed

```sql
SELECT COUNT(*) FROM dev.raw_cdc.customers_changefeed;
SELECT COUNT(*) FROM dev.raw_cdc.orders_changefeed;
SELECT COUNT(*) FROM dev.raw_cdc.order_items_changefeed;
LIST '/Volumes/dev/step_right/landing/products/';
```

With real (if imperfect) rows in `dev.raw_cdc` and a first file in the landing volume, Section 2
can build its first bronze pipeline against actual data instead of an empty schema.

<!-- prevnext:start -->

---

| [&larr; Previous: Test Data Strategy - Planning your Test Data Preparation]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/test-data-strategy-planning-your-test-data-preparation/' | relative_url }}) | [Next: Download Project Source Code &rarr;]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/download-project-source-code/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

