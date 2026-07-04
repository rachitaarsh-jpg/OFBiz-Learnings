# SQL Assignment 2 - Module 8

---

# 5. Mixed Party + Order Queries

## 5.1 Shipping Addresses for October 2023 Orders

### Business Problem

Customer Service might need to verify addresses for orders placed or completed in October 2023. This helps ensure shipments are delivered correctly and prevents address-related issues.

```sql
SELECT
    oh.ORDER_ID,
    pcm.PARTY_ID AS "CUSTOMER ID",
    CONCAT(pr.FIRST_NAME) AS CUSTOMER_NAME,
    pa.ADDRESS1,
    pa.CITY,
    pa.STATE_PROVINCE_GEO_ID,
    pa.POSTAL_CODE,
    pa.COUNTRY_GEO_ID,
    oh.STATUS_ID,
    oh.ORDER_DATE
FROM order_header oh
JOIN order_contact_mech ocm
    ON oh.ORDER_ID = ocm.ORDER_ID
LEFT JOIN postal_address pa
    ON ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
JOIN party_contact_mech pcm
    ON ocm.CONTACT_MECH_ID = pcm.CONTACT_MECH_ID
JOIN person pr
    ON pcm.PARTY_ID = pr.PARTY_ID
WHERE oh.STATUS_ID IN ('ORDER_CREATED', 'ORDER_COMPLETED')
  AND DATE(oh.ORDER_DATE) BETWEEN '2023-10-01' AND '2023-10-31';
```

---

## 5.2 Orders from New York

### Business Problem

Companies often want region-specific analysis to plan local marketing, staffing, or promotions in certain areas—here, specifically, New York.

```sql
SELECT
    oh.ORDER_ID,
    CONCAT(pr.FIRST_NAME) AS CUSTOMER_NAME,
    pa.ADDRESS1 AS STREET_ADDRESS,
    pa.CITY,
    pa.STATE_PROVINCE_GEO_ID AS STATE_PROVINCE,
    pa.POSTAL_CODE,
    oh.GRAND_TOTAL,
    oh.STATUS_ID,
    DATE(oh.ORDER_DATE) AS ORDER_DATE
FROM order_header oh
JOIN order_contact_mech ocm
    ON oh.ORDER_ID = ocm.ORDER_ID
LEFT JOIN postal_address pa
    ON ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
JOIN party_contact_mech pcm
    ON ocm.CONTACT_MECH_ID = pcm.CONTACT_MECH_ID
JOIN person pr
    ON pcm.PARTY_ID = pr.PARTY_ID
WHERE pa.STATE_PROVINCE_GEO_ID = 'NY';
```

---

## 5.3 Top-Selling Product in New York

### Business Problem

Merchandising teams need to identify the best-selling product(s) in a specific region (New York) for targeted restocking or promotions.

```sql
SELECT
    oi.PRODUCT_ID,
    p.INTERNAL_NAME,
    SUM(oi.QUANTITY) AS TOTAL_QUANTITY_SOLD,
    pa.CITY,
    pa.STATE_PROVINCE_GEO_ID AS STATE,
    SUM(oi.QUANTITY * oi.UNIT_PRICE) AS REVENUE
FROM order_header oh
JOIN order_item oi
    ON oh.ORDER_ID = oi.ORDER_ID
JOIN product p
    ON oi.PRODUCT_ID = p.PRODUCT_ID
JOIN order_contact_mech ocm
    ON oi.ORDER_ID = ocm.ORDER_ID
   AND ocm.CONTACT_MECH_PURPOSE_TYPE_ID = 'SHIPPING_LOCATION'
LEFT JOIN postal_address pa
    ON ocm.CONTACT_MECH_ID = pa.CONTACT_MECH_ID
WHERE pa.STATE_PROVINCE_GEO_ID = 'NY'
GROUP BY
    oi.PRODUCT_ID,
    p.INTERNAL_NAME,
    pa.CITY,
    pa.STATE_PROVINCE_GEO_ID
ORDER BY
    TOTAL_QUANTITY_SOLD DESC,
    REVENUE DESC;
```

---

## 7.3 Store-Specific (Facility-Wise) Revenue

### Business Problem

Different physical or online stores (facilities) may have varying levels of performance. The business wants to compare revenue across facilities for sales planning and budgeting.

```sql
SELECT
    f.FACILITY_ID,
    f.FACILITY_NAME,
    COUNT(DISTINCT oh.ORDER_ID) AS TOTAL_ORDERS,
    SUM(oi.QUANTITY * oi.UNIT_PRICE) AS TOTAL_REVENUE,
    '2026-04-01 TO 2026-04-30' AS DATE_RANGE
FROM order_header oh
JOIN order_item oi
    ON oh.ORDER_ID = oi.ORDER_ID
JOIN order_item_ship_group oisg
    ON oi.ORDER_ID = oisg.ORDER_ID
   AND oi.SHIP_GROUP_SEQ_ID = oisg.SHIP_GROUP_SEQ_ID
JOIN facility f
    ON oisg.FACILITY_ID = f.FACILITY_ID
WHERE DATE(oh.ORDER_DATE) BETWEEN '2026-04-01' AND '2026-04-30'
GROUP BY
    f.FACILITY_ID,
    f.FACILITY_NAME
ORDER BY
    TOTAL_REVENUE DESC,
    TOTAL_ORDERS DESC;
```

---

# 8. Inventory Management & Transfers

## 8.1 Lost and Damaged Inventory

### Business Problem

Warehouse managers need to track shrinkage such as lost or damaged inventory to reconcile physical vs. system counts.

```sql
SELECT
    iid.INVENTORY_ITEM_ID,
    oi.PRODUCT_ID,
    oisg.FACILITY_ID,
    SUM(ABS(iid.QUANTITY_ON_HAND_DIFF)) AS QUANTITY_LOST_OR_DAMAGED,
    iid.REASON_ENUM_ID AS REASON_CODE,
    DATE(iid.EFFECTIVE_DATE) AS TRANSACTION_DATE
FROM inventory_item_detail iid
JOIN order_item oi
    ON iid.ORDER_ID = oi.ORDER_ID
   AND iid.ORDER_ITEM_SEQ_ID = oi.ORDER_ITEM_SEQ_ID
JOIN order_item_ship_group oisg
    ON oi.ORDER_ID = oisg.ORDER_ID
   AND oi.SHIP_GROUP_SEQ_ID = oisg.SHIP_GROUP_SEQ_ID
WHERE iid.REASON_ENUM_ID IN ('VAR_LOST', 'VAR_DAMAGED')
GROUP BY
    iid.INVENTORY_ITEM_ID,
    oi.PRODUCT_ID,
    oisg.FACILITY_ID,
    iid.REASON_ENUM_ID,
    DATE(iid.EFFECTIVE_DATE)
ORDER BY QUANTITY_LOST_OR_DAMAGED DESC;
```

---

## 8.2 Low Stock or Out of Stock Items Report

### Business Problem

Avoiding out-of-stock situations is critical. This report flags items that have fallen below a certain reorder threshold or have zero available stock.

```sql
SELECT
    ii.PRODUCT_ID,
    p.INTERNAL_NAME AS PRODUCT_NAME,
    ii.FACILITY_ID,
    SUM(ii.QUANTITY_ON_HAND_TOTAL) AS QOH,
    SUM(ii.AVAILABLE_TO_PROMISE_TOTAL) AS ATP,
    pf.MINIMUM_STOCK AS REORDER_THRESHOLD,
    CURDATE() AS DATE_CHECKED
FROM inventory_item ii
JOIN product p
    ON ii.PRODUCT_ID = p.PRODUCT_ID
JOIN product_facility pf
    ON ii.PRODUCT_ID = pf.PRODUCT_ID
   AND ii.FACILITY_ID = pf.FACILITY_ID
WHERE ii.STATUS_ID = 'INV_AVAILABLE'
GROUP BY
    ii.PRODUCT_ID,
    p.INTERNAL_NAME,
    ii.FACILITY_ID,
    pf.MINIMUM_STOCK
HAVING
    SUM(ii.AVAILABLE_TO_PROMISE_TOTAL) <= pf.MINIMUM_STOCK
    OR SUM(ii.AVAILABLE_TO_PROMISE_TOTAL) <= 0
ORDER BY ATP ASC;
```

---

## 8.3 Retrieve the Current Facility (Physical or Virtual) of Open Orders

### Business Problem

The business wants to know where open orders are currently assigned, whether in a physical store or a virtual facility.

```sql
SELECT
    oh.ORDER_ID,
    oh.STATUS_ID AS ORDER_STATUS,
    f.FACILITY_ID,
    f.FACILITY_NAME,
    f.FACILITY_TYPE_ID
FROM order_header oh
JOIN order_item_ship_group oisg
    ON oh.ORDER_ID = oisg.ORDER_ID
JOIN facility f
    ON oisg.FACILITY_ID = f.FACILITY_ID
WHERE oh.STATUS_ID NOT IN (
    'ORDER_COMPLETED',
    'ORDER_CANCELLED',
    'ORDER_REJECTED'
)
AND f.FACILITY_ID IS NOT NULL
ORDER BY oh.ORDER_ID;
```

---

## 8.4 Items Where QOH and ATP Differ

### Business Problem

Sometimes the Quantity on Hand (QOH) doesn't match the Available to Promise (ATP) due to pending orders, reservations, or data discrepancies.

```sql
SELECT
    ii.PRODUCT_ID,
    ii.FACILITY_ID,
    SUM(ii.QUANTITY_ON_HAND_TOTAL) AS QOH,
    SUM(ii.AVAILABLE_TO_PROMISE_TOTAL) AS ATP,
    SUM(ii.QUANTITY_ON_HAND_TOTAL) - SUM(ii.AVAILABLE_TO_PROMISE_TOTAL) AS DIFFERENCE
FROM inventory_item ii
JOIN product p
    ON ii.PRODUCT_ID = p.PRODUCT_ID
JOIN facility f
    ON ii.FACILITY_ID = f.FACILITY_ID
GROUP BY
    ii.PRODUCT_ID,
    ii.FACILITY_ID
HAVING
    SUM(ii.QUANTITY_ON_HAND_TOTAL) <> SUM(ii.AVAILABLE_TO_PROMISE_TOTAL)
ORDER BY
    ABS(SUM(ii.QUANTITY_ON_HAND_TOTAL) - SUM(ii.AVAILABLE_TO_PROMISE_TOTAL)) DESC;
```

---

## 8.5 Order Item Current Status Changed Date-Time

### Business Problem

Operations teams need to audit when an order item's status was last changed.

```sql
SELECT
    os.ORDER_ID,
    os.ORDER_ITEM_SEQ_ID,
    os.STATUS_ID AS CURRENT_STATUS_ID,
    os.STATUS_DATETIME AS STATUS_CHANGE_DATETIME,
    os.STATUS_USER_LOGIN AS CHANGED_BY
FROM order_status os
INNER JOIN (
    SELECT
        ORDER_ID,
        ORDER_ITEM_SEQ_ID,
        MAX(STATUS_DATETIME) AS LATEST_DATETIME
    FROM order_status
    WHERE ORDER_ITEM_SEQ_ID IS NOT NULL
    GROUP BY
        ORDER_ID,
        ORDER_ITEM_SEQ_ID
) latest
ON os.ORDER_ID = latest.ORDER_ID
AND os.ORDER_ITEM_SEQ_ID = latest.ORDER_ITEM_SEQ_ID
AND os.STATUS_DATETIME = latest.LATEST_DATETIME
ORDER BY
    os.ORDER_ID,
    os.ORDER_ITEM_SEQ_ID;
```

---

## 8.6 Total Orders by Sales Channel

### Business Problem

Marketing and sales teams want to see how many orders come from each sales channel.

```sql
SELECT
    COALESCE(oh.SALES_CHANNEL_ENUM_ID, 'NOT_SPECIFIED') AS SALES_CHANNEL,
    COALESCE(e.DESCRIPTION, 'Unknown') AS CHANNEL_DESCRIPTION,
    COUNT(oh.ORDER_ID) AS TOTAL_ORDERS,
    SUM(oh.GRAND_TOTAL) AS TOTAL_REVENUE,
    DATE_FORMAT(oh.ORDER_DATE, '%Y-%m') AS REPORTING_PERIOD
FROM order_header oh
LEFT JOIN enumeration e
    ON oh.SALES_CHANNEL_ENUM_ID = e.ENUM_ID
WHERE oh.STATUS_ID NOT IN (
    'ORDER_CANCELLED',
    'ORDER_REJECTED'
)
GROUP BY
    oh.SALES_CHANNEL_ENUM_ID,
    e.DESCRIPTION,
    DATE_FORMAT(oh.ORDER_DATE, '%Y-%m')
ORDER BY
    REPORTING_PERIOD DESC,
    TOTAL_ORDERS DESC;
```
