# Designing Data Tables for a Fruit Catalog Feature

**Author:** Ignatius Fairlight | **Date:** 5th May 2026

---

## Summary

This report covers two data table designs for a fruit catalog feature in a e-commerce platform, their SQL queries, advantages and disadvantages, and a recommendation. The designs differ in their approach to storing metadata pertaining to the fruit listing.

---

## Context

The data tables presented in this report are intended for implementing a fruit catalog feature into a e-commerce platform. This fruit listing behaves similarly to those commonly seen in other e-commerce platforms, where the fruit listing appears after the app initializes, displaying product listings and promotions.

When an administrator creates a fruit listing, it consists of a title, a related media file, and optional metadata. This metadata ensures a fruit listing is only served to targeted users based on the intention behind the fruit listing. Some fruit listings will appear to every user, meaning those fruit listings will have no metadata tied to them. Others will have metadata that could be based on discounts from an ongoing campaign, the user's country, location, the last warehouse the user visited within a timeframe, etc.

---

## Disclaimer

The examples in this report use a test database containing both designs labelled `d1` for Design 1 and `d2` for Design 2, as well as other related reference tables, and contains dummy records to simulate real-world use cases.

Schema diagrams were produced using DrawDB, and all queries were executed and verified using DBeaver.

---

## Design 1 – Dedicated Metadata Table (EAV Approach)

### Description

In this approach, metadata is stored in one table `fruit_listing_meta`. Pop-ups with metadata are referenced by `fruit_listing_meta.parent`. The `attribute` column contains the type of metadata paired with its respective value. For example, `discount` refers to a discount ID, `warehouse` to an warehouse ID, `country` to a country ID, while some attributes store their values directly — such as `starts_at`, `ends_at`, and `coordinate` (containing four decimal values representing a geographic bounding box).

### SQL Queries

#### FETCH QUERIES

**1. General fetch query — "Give me a list of fruit listings and their metadata"**

```sql
SELECT
    dpu.id AS FruitListingID,
    dpu.title AS FruitListingTitle,
    dpum.attribute AS Attributes,
    dpum.value AS Values
FROM d1_fruit_listing dpu
JOIN d1_fruit_listing_meta dpum ON dpum.parent = dpu.id
```

---

**2. Specific fetch query — "Give me the metadata for fruit listing 32"**

```sql
SELECT
    dpu.id          AS FruitListingID,
    dpu.title       AS FruitListingTitle,
    v.code          AS Voucher,
    cc.name         AS Country,
    ho.name         AS Outlet,
    coord_meta.value AS Coordinate,
    spend_start.value AS AvailableFrom,
    spend_end.value   AS AvailableUntil
FROM d1_fruit_listing dpu
LEFT JOIN d1_fruit_listing_meta discount_meta
    ON discount_meta.parent = dpu.id AND discount_meta.attribute = 'discount'
LEFT JOIN discount v
    ON v.id = CAST(discount_meta.value AS UNSIGNED)
LEFT JOIN d1_fruit_listing_meta country_meta
    ON country_meta.parent = dpu.id AND country_meta.attribute = 'country'
LEFT JOIN country cc
    ON cc.id = CAST(country_meta.value AS UNSIGNED)
LEFT JOIN d1_fruit_listing_meta warehouse_meta
    ON warehouse_meta.parent = dpu.id AND warehouse_meta.attribute = 'warehouse'
LEFT JOIN kokona_warehouse ho
    ON ho.id = CAST(warehouse_meta.value AS UNSIGNED)
LEFT JOIN d1_fruit_listing_meta coord_meta
    ON coord_meta.parent = dpu.id AND coord_meta.attribute = 'coordinate'
LEFT JOIN d1_fruit_listing_meta spend_start
    ON spend_start.parent = dpu.id AND spend_start.attribute = 'available_from'
LEFT JOIN d1_fruit_listing_meta spend_end
    ON spend_end.parent = dpu.id AND spend_end.attribute = 'available_until'
WHERE dpu.id = 32;
```

---

**3. Specific fetch query — "Give me the fruit listings related to warehouse 3"**

```sql
SELECT
    dpu.id AS FruitListingID,
    dpu.title AS FruitListingTitle,
    dpum.value AS WarehouseID,
    ho.name AS WarehouseName
FROM d1_fruit_listing dpu
JOIN d1_fruit_listing_meta dpum ON dpum.parent = dpu.id
JOIN kokona_warehouse ho ON ho.id = CAST(dpum.value AS UNSIGNED)
WHERE
    dpum.attribute = 'warehouse' AND
    dpum.value = '3';
```

---

**4. Specific fetch query — "Give me the number of fruit listings that start from 2025-01-01 to 2025-12-31, where the user spends from 2025-03-01 to 2025-09-30"**

```sql
SELECT COUNT(DISTINCT dpu.id) AS FruitListingCount
FROM d1_fruit_listing dpu
JOIN d1_fruit_listing_meta spend_start
    ON spend_start.parent = dpu.id AND spend_start.attribute = 'available_from'
JOIN d1_fruit_listing_meta spend_end
    ON spend_end.parent = dpu.id AND spend_end.attribute = 'available_until'
WHERE dpu.starts_at >= '2025-01-01 00:00:00'
  AND dpu.ends_at <= '2025-12-31 23:59:59'
  AND spend_start.value >= '2025-03-01 00:00:00'
  AND spend_end.value <= '2025-09-30 23:59:59';
```

---

**5. Specific fetch query — "Give me all active fruit listings visible to users in Malaysia who last visited warehouse 3 in the last 3 months"**

```sql
SELECT DISTINCT dpu.id AS FruitListingID, dpu.title AS FruitListingTitle
FROM d1_fruit_listing dpu
JOIN d1_fruit_listing_meta country_meta
    ON country_meta.parent = dpu.id AND country_meta.attribute = 'country'
JOIN country cc
    ON cc.id = CAST(country_meta.value AS UNSIGNED) AND cc.code = 'MY'
JOIN d1_fruit_listing_meta warehouse_meta
    ON warehouse_meta.parent = dpu.id AND warehouse_meta.attribute = 'warehouse'
JOIN d1_fruit_listing_meta spend_start
    ON spend_start.parent = dpu.id AND spend_start.attribute = 'available_from'
JOIN d1_fruit_listing_meta spend_end
    ON spend_end.parent = dpu.id AND spend_end.attribute = 'available_until'
WHERE dpu.status = 'active'
  AND CAST(warehouse_meta.value AS UNSIGNED) = 3
  AND spend_start.value <= NOW()
  AND spend_end.value >= DATE_SUB(NOW(), INTERVAL 3 MONTH);
```

---

#### INSERT QUERIES

**1. General insert, no metadata**

```sql
INSERT INTO d1_fruit_listing (title, media_id, status, starts_at, ends_at, created_by)
VALUES ("SUMMER_MANGO", 2, "draft", "2026-05-29 00:00:00", "2026-06-22", 2);
```

---

**2. Insert with metadata — "Create a fruit listing from 1 June to 31 August 2026 for a free-shipping discount, targeting everyone in Malaysia"**

```sql
-- First: create the fruit listing
INSERT INTO d1_fruit_listing (title, media_id, status, starts_at, ends_at, created_by)
VALUES ("Free Shipping Fruits", 6, "active", "2026-06-01 00:00:00", "2026-08-31 23:59:59", 1);

-- Second: create metadata
INSERT INTO d1_fruit_listing_meta (parent, attribute, value)
VALUES
    (47, 'discount', '3'),
    (47, 'country', '1');
```

---

**3. Insert one metadata to an existing fruit listing — "Add country to our test fruit listing"**

```sql
INSERT INTO d1_fruit_listing_meta (parent, attribute, value)
VALUES (40, 'country', '1');
```

---

**4. Insert and update metadata — "Add missing discount and fix wrong country for Summer Sale listing"**

```sql
-- Add discount
INSERT INTO d1_fruit_listing_meta (parent, attribute, value)
VALUES (39, 'discount', '10');

-- Fix country
UPDATE d1_fruit_listing_meta dpum
SET value = '1'
WHERE dpum.parent = 39 AND attribute = 'country';
```

---

#### DELETE QUERIES

**1. Delete fruit listing with no metadata**

```sql
DELETE FROM d1_fruit_listing WHERE id = 46;
```

---

**2. Delete fruit listing with metadata**

```sql
-- Delete metadata first
DELETE FROM d1_fruit_listing_meta WHERE parent = 16;

-- Then delete the fruit listing
DELETE FROM d1_fruit_listing WHERE id = 16;
```

---

**3. Remove one metadata — "Make the Singapore Launch fruit listing visible to everyone"**

```sql
DELETE FROM d1_fruit_listing_meta
WHERE parent = 37 AND attribute = 'country';
```

---

**4. Remove multiple metadata — "Make Summer Mango visible to everyone"**

```sql
DELETE FROM d1_fruit_listing_meta
WHERE
    parent = 38 AND
    attribute IN ('country', 'warehouse');
```

---

### Advantages

- Creating the table is straightforward
- Inserting new data is trivial
- No new columns or tables needed to accommodate new attribute types

### Disadvantages

- Cannot enforce mandatory attributes; hard to generate reports with structured attribute columns
- No SQL data types — all attribute values are `VARCHAR`, allowing inconsistent values
- Cannot enforce referential integrity
- Possibility of inconsistent attribute naming (e.g., `discount_id` vs `discounts_id`)
- Query complexity scales proportionally with business requirement complexity
- Hard to reshape data to fit business reporting needs
- Data integrity responsibility is offloaded to the codebase, resulting in complex business logic

---

## Design 2 – Separate Table Per Metadata Type (Normalized Approach)

### Description

In contrast to Design 1, this approach splits each metadata type into its own dedicated table. Each metadata table references its respective fruit listing via `fruit_listing.id`. Each metadata table is independent with no dependencies on other metadata tables, better reflecting the optional nature of metadata.

**Metadata tables:** `fruit_listing_discount`, `fruit_listing_warehouse`, `fruit_listing_country`, `fruit_listing_region`, `fruit_listing_availability`

### SQL Queries

#### FETCH QUERIES

**1. General fetch query — "Give me a list of fruit listings and their metadata"**

```sql
SELECT
    dpu.id AS FruitListingID,
    dpu.title AS FruitListingTitle,
    v.code AS Voucher,
    cc.name AS Country,
    ho.name AS Outlet,
    dpuc2.lat_max AS LatMax,
    dpuc2.lat_min AS LatMin,
    dpuc2.lng_max AS LngMax,
    dpuc2.lng_min AS LngMin,
    dpusp.starts_at AS AvailableFrom,
    dpusp.ends_at AS AvailableUntil
FROM d2_fruit_listing dpu
LEFT JOIN d2_fruit_listing_discount dpuv ON dpuv.fruit_listing_id = dpu.id
LEFT JOIN discount v ON v.id = dpuv.discount_id
LEFT JOIN d2_fruit_listing_country dpuc ON dpuc.fruit_listing_id = dpu.id
LEFT JOIN country cc ON cc.id = dpuc.country_id
LEFT JOIN d2_fruit_listing_warehouse dpuo ON dpuo.fruit_listing_id = dpu.id
LEFT JOIN kokona_warehouse ho ON ho.id = dpuo.kokona_warehouse_id
LEFT JOIN d2_fruit_listing_coordinate dpuc2 ON dpuc2.fruit_listing_id = dpu.id
LEFT JOIN d2_fruit_listing_spending_period dpusp ON dpusp.fruit_listing_id = dpu.id
```

---

**2. Specific fetch query — "Give me the metadata for fruit listing 32"**

```sql
SELECT
    dpu.id AS FruitListingID,
    dpu.title AS FruitListingTitle,
    v.code AS Voucher,
    cc.name AS Country,
    ho.name AS Outlet,
    dpuc2.lat_max AS LatMax,
    dpuc2.lat_min AS LatMin,
    dpuc2.lng_max AS LngMax,
    dpuc2.lng_min AS LngMin,
    dpusp.starts_at AS AvailableFrom,
    dpusp.ends_at AS AvailableUntil
FROM d2_fruit_listing dpu
LEFT JOIN d2_fruit_listing_discount dpuv ON dpuv.fruit_listing_id = dpu.id
LEFT JOIN discount v ON v.id = dpuv.discount_id
LEFT JOIN d2_fruit_listing_country dpuc ON dpuc.fruit_listing_id = dpu.id
LEFT JOIN country cc ON cc.id = dpuc.country_id
LEFT JOIN d2_fruit_listing_warehouse dpuo ON dpuo.fruit_listing_id = dpu.id
LEFT JOIN kokona_warehouse ho ON ho.id = dpuo.kokona_warehouse_id
LEFT JOIN d2_fruit_listing_coordinate dpuc2 ON dpuc2.fruit_listing_id = dpu.id
LEFT JOIN d2_fruit_listing_spending_period dpusp ON dpusp.fruit_listing_id = dpu.id
WHERE dpu.id = 32;
```

---

**3. Specific fetch query — "Give me the fruit listings related to warehouse 3"**

```sql
SELECT
    dpu.id AS FruitListingID,
    dpu.title AS FruitListingTitle,
    dpuo.kokona_warehouse_id AS WarehouseID,
    ho.name AS WarehouseName
FROM d2_fruit_listing dpu
JOIN d2_fruit_listing_warehouse dpuo ON dpuo.fruit_listing_id = dpu.id
JOIN kokona_warehouse ho ON ho.id = dpuo.kokona_warehouse_id
WHERE dpuo.kokona_warehouse_id = 3;
```

---

**4. Specific fetch query — "Give me the number of fruit listings that start from 2025-01-01 to 2025-12-31, where the user spends from 2025-03-01 to 2025-09-30"**

```sql
SELECT COUNT(*) AS TotalFruitListings
FROM d2_fruit_listing dpu
JOIN d2_fruit_listing_spending_period dpusp ON dpusp.fruit_listing_id = dpu.id
WHERE
    dpu.starts_at >= '2025-01-01' AND
    dpu.ends_at <= '2025-12-31' AND
    dpusp.starts_at >= '2025-03-01' AND
    dpusp.ends_at <= '2025-09-30';
```

---

**5. Specific fetch query — "Give me all active fruit listings visible to users in Malaysia who last visited warehouse 3 in the last 3 months"**

```sql
SELECT DISTINCT dpu.id AS FruitListingID, dpu.title AS FruitListingTitle
FROM d2_fruit_listing dpu
JOIN d2_fruit_listing_warehouse dpuo ON dpuo.fruit_listing_id = dpu.id
JOIN kokona_warehouse ho ON ho.id = dpuo.kokona_warehouse_id
JOIN d2_fruit_listing_country dpuc ON dpuc.fruit_listing_id = dpu.id
JOIN country cc ON cc.id = dpuc.country_id
JOIN d2_fruit_listing_spending_period dpusp ON dpusp.fruit_listing_id = dpu.id
WHERE dpu.status = 'active'
  AND dpuo.kokona_warehouse_id = 3
  AND dpuc.country_id = 1
  AND dpusp.starts_at <= NOW()
  AND dpusp.ends_at >= DATE_SUB(NOW(), INTERVAL 3 MONTH);
```

---

#### INSERT QUERIES

**1. General insert, no metadata**

```sql
INSERT INTO d2_fruit_listing (title, media_id, status, starts_at, ends_at, created_by)
VALUES ("SUMMER_MANGO", 2, "draft", "2026-05-29 00:00:00", "2026-06-22", 2);
```

---

**2. Insert with metadata — "Create a fruit listing from 1 June to 31 August 2026 for a free-shipping discount, targeting everyone in Malaysia"**

```sql
-- Create fruit listing
INSERT INTO d2_fruit_listing (title, media_id, status, starts_at, ends_at, created_by)
VALUES ("Free Shipping Fruits", 6, "active", "2026-06-01 00:00:00", "2026-08-31 23:59:59", 1);

-- Add discount
INSERT INTO d2_fruit_listing_discount (fruit_listing_id, discount_id)
VALUES (47, 3);

-- Add country
INSERT INTO d2_fruit_listing_country (fruit_listing_id, country_id)
VALUES (47, 1);
```

---

**3. Insert one metadata to an existing fruit listing — "Add country to our test fruit listing"**

```sql
INSERT INTO d2_fruit_listing_country (fruit_listing_id, country_id)
VALUES (40, 1);
```

---

**4. Insert and update metadata — "Add missing discount and fix wrong country for Summer Sale listing"**

```sql
-- Add discount
INSERT INTO d2_fruit_listing_discount (fruit_listing_id, discount_id)
VALUES (39, 10);

-- Fix country
UPDATE d2_fruit_listing_country
SET country_id = 1
WHERE fruit_listing_id = 39;
```

---

#### DELETE QUERIES

**1. Delete fruit listing with no metadata**

```sql
DELETE FROM d2_fruit_listing WHERE id = 46;
```

---

**2. Delete fruit listing with metadata**

```sql
DELETE FROM d2_fruit_listing WHERE id = 16;
```

> Other entries in the metadata tables are automatically removed via `ON DELETE CASCADE` on the foreign key referencing `d2_fruit_listing.id`. No manual cleanup required.

---

**3. Remove one metadata — "Make the Singapore Launch fruit listing visible to everyone"**

```sql
DELETE FROM d2_fruit_listing_country WHERE fruit_listing_id = 37;
```

---

**4. Remove multiple metadata — "Make Summer Mango visible to everyone"**

```sql
DELETE FROM d2_fruit_listing_country WHERE fruit_listing_id = 38;
DELETE FROM d2_fruit_listing_warehouse WHERE fruit_listing_id = 38;
```

---

### Advantages

- Search queries are efficient and predictable
- Query complexity is frontloaded — harder to design, easier to use
- Full SQL data type enforcement
- Referential integrity via foreign key constraints
- Schema is self-documenting — relationships are immediately readable
- `ON DELETE CASCADE` handles child record cleanup automatically

### Disadvantages

- Table design time scales with data complexity
- Each new metadata type requires a new table

> *Whether a new table per metadata type is an advantage or disadvantage depends on perspective — see Recommendation.*

---

## Comparison

| Criteria | Design 1 | Design 2 |
|---|---|---|
| Designing the data table(s) | Easy | Hard |
| Forming search queries | Easy → Hard | Hard → Easy |
| Forming insert queries | Easy | Easy |
| Forming delete queries | Easy | Easy |
| Data type integrity | No | Yes |
| Referential integrity | No | Yes |
| Extensibility | Very easy | Easy–Medium |
| Schema readability | Hard | Easy |

---

## Recommendation

**Design 2 is the recommended approach.**

Design 1 is easier to set up and requires fewer tables, and it accommodates new metadata types without schema changes. However, its difficulty in querying and lack of data integrity enforcement far outweigh these benefits. Query complexity grows proportionally with business requirements, and data integrity is pushed into the application layer.

Design 2 requires more upfront work, but the tradeoff pays for itself. SQL data types and foreign key constraints ensure accuracy. The schema is readable and predictable — any developer unfamiliar with the feature can understand the fruit listing metadata structure immediately from the schema alone.

When a new metadata type is needed, Design 2 handles it cleanly. The general fetch query in Design 2 already demonstrates that shaping data to fit business narratives requires significantly less cognitive overhead than its Design 1 counterpart, especially as requirements grow in complexity.

For these reasons, Design 2 is the stronger foundation for this feature and any metadata extensions that may follow.
