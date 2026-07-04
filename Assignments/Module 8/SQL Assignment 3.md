# SQL Assignment 3

## 1 Completed Sales Orders (Physical Items)

### Business Problem  
Merchants need to track only physical items (requiring shipping and fulfillment) for logistics and shipping-cost analysis.

```sql
SELECT oh.order\_id,  
       oi.order\_item\_seq\_id,  
       p.product\_id,  
       p.product\_type\_id,  
       oh.sales\_channel\_enum\_id,  
       oh.order\_date,  
       oh.entry\_date,  
       os.status\_id,  
       os.status\_datetime,  
       oh.order\_type\_id,  
       oh.product\_store\_id  
         
       FROM ORDER\_HEADER oh   
       JOIN ORDER\_ITEM oi   
       ON oh.order\_id \= oi.order\_id  
       JOIN ORDER\_STATUS os   
       ON oh.order\_id \= os.order\_id AND oi.order\_item\_seq\_id \= os.order\_item\_seq\_id  
       JOIN PRODUCT p   
       ON oi.product\_id  \= p.product\_id  
         
       WHERE p.is\_variant \= 'Y' AND p.is\_virtual \= 'N'  
        AND oh.order\_type\_id \= 'SALES\_ORDER'  
        AND os.status\_id \= 'ITEM\_COMPLETED';
```
;
```



---

## 2. Completed Return Items

### Business Problem  
Customer service and finance often need insights into returned items to manage refunds, replacements, and inventory restocking

```sql
SELECT rh.return\_id,  
       ri.order\_id,         
       oh.product\_store\_id,  
       rs.status\_datetime,  
       oh.order\_name,  
       rh.from\_party\_id,  
       rh.return\_date,  
       rh.entry\_date,  
       rh.return\_channel\_enum\_id  
         
       FROM return\_header rh   
       JOIN return\_item ri   
       ON rh.return\_id \= ri.return\_id  
       JOIN order\_header oh  
       ON ri.order\_id \= oh.order\_id  
       JOIN return\_status rs   
       ON rh.return\_id \= rs.return\_id  
         
       WHERE rs.status\_id \= 'RETURN\_COMPLETED';
```



---

## 3 Single-Return Orders (Last Month)

### Business Problem  
The mechandising team needs a list of orders that only have one return.

```sql
SELECT p.party\_id,  
       pr.first\_name  
         
       FROM party p   
       JOIN person pr   
       ON p.party\_id \= pr.party\_id  
       JOIN return\_header rh  
       ON p.party\_id \= rh.from\_party\_id  
       JOIN return\_item ri   
       ON rh.return\_id \= ri.return\_id  
       JOIN order\_header oh   
       ON ri.order\_id \= oh.order\_id  
         
       WHERE oh.order\_id IN (  
       SELECT order\_id from return\_item  
       GROUP BY order\_id   
       HAVING COUNT(\*) \= 1  
       )  
         
       GROUP BY p.party\_id, pr.first\_name  
       ORDER BY pr.first\_name;
```



---

## 4 Returns and Appeasements

### Business Problem  
The retailer needs the total amount of items, were returned as well as how many appeasements were issued.

```sql
SELECT  
    r.total\_returns,  
    r.return\_dollar\_total,  
    a.total\_appeasements,  
    a.appeasement\_dollar\_total  
FROM  
(  
    SELECT  
        COUNT(DISTINCT return\_id) AS total\_returns,  
        SUM(amount) AS return\_dollar\_total  
    FROM return\_item\_billing  
) r,  
(  
    SELECT  
        COUNT(\*) AS total\_appeasements,  
        SUM(amount) AS appeasement\_dollar\_total  
    FROM return\_adjustment  
    WHERE return\_adjustment\_type\_id \= 'APPEASEMENT'  
) a;
```

---

## 5 Detailed Return Information

### Business Problem  
Certain teams need granular return data (reason, date, refund amount) for analyzing return rates, identifying recurring issues, or updating policies.

```sql
SELECT rh.return\_id,  
       rh.entry\_date,  
       ra.return\_adjustment\_type\_id,  
       ri.return\_reason\_id,  
       rib.amount,  
       ra.comments,  
       oh.order\_id,  
       oh.order\_date,  
       rh.entry\_date AS return\_date,  
       oh.product\_store\_id

FROM return\_header rh  
JOIN return\_item ri   
  ON rh.return\_id \= ri.return\_id  
JOIN return\_item\_billing rib  
  ON ri.return\_id \= rib.return\_id   
 AND ri.return\_item\_seq\_id \= rib.return\_item\_seq\_id  
JOIN order\_header oh   
  ON ri.order\_id \= oh.order\_id  
LEFT JOIN return\_adjustment ra   
  ON rh.return\_id \= ra.return\_id;
```

---

## 6 Orders with Multiple Returns

### Business Problem  
Analyzing orders with multiple returns can identify potential fraud, chronic issues with certain items, or inconsistent shipping processes

```sql
SELECT  
    ri.order\_id,  
    rh.return\_id,  
    rh.return\_date,  
    ri.return\_reason\_id AS return\_reason,  
    ri.return\_quantity AS return\_quantity  
FROM return\_header rh  
JOIN return\_item ri  
    ON rh.return\_id \= ri.return\_id  
WHERE ri.order\_id IN (  
    SELECT order\_id  
    FROM return\_item  
    GROUP BY order\_id  
    HAVING COUNT(DISTINCT return\_id) \> 1  
);
```

---

## 7 Store with Most One-Day Shipped Orders (Last Month)

### Business Problem  
Identify which facility (store) handled the highest volume of “one-day shipping” orders in the previous month, useful for operational benchmarking.

```sql
SELECT f.FACILITY\_ID,  
       f.FACILITY\_NAME,  
       COUNT(DISTINCT oisg.order\_id) AS TOTAL\_ONE\_DAY\_SHIP\_ORDERS,  
       DATE\_FORMAT(DATE\_SUB(CURRENT\_DATE,INTERVAL 1 MONTH),'%Y-%m') AS REPORTING\_PERIOD  
         
       FROM ORDER\_ITEM\_SHIP\_GROUP oisg  
       JOIN FACILITY f   
       ON oisg.FACILITY\_ID \= f.FACILITY\_ID  
       JOIN ORDER\_HEADER oh   
       ON oisg.ORDER\_ID \= oh.ORDER\_ID WHERE oisg.SHIPMENT\_METHOD\_TYPE\_ID \= 'NEXT\_DAY'  
       AND oh.ORDER\_DATE \>= DATE\_FORMAT(DATE\_SUB(CURRENT\_DATE, INTERVAL 1 MONTH),'%Y-%m-01') AND oh.ORDER\_DATE \< DATE\_FORMAT(CURRENT\_DATE, '%Y-%m-01')  
       AND oh.STATUS\_ID \= 'ORDER\_COMPLETED'  
       GROUP by f.FACILITY\_ID, f.FACILITY\_NAME  
       ORDER BY TOTAL\_ONE\_DAY\_SHIP\_ORDERS DESC  
       LIMIT 1;
```



---

## 8 List of Warehouse Pickers

### Business Problem  
Warehouse managers need a list of employees responsible for picking and packing orders to manage shifts, productivity, and training needs.

```sql
SELECT fp.PARTY\_ID,  
       CONCAT(p.FIRST\_NAME,' ',p.LAST\_NAME) AS NAME,  
       fp.ROLE\_TYPE\_ID,  
       fp.FACILITY\_ID,  
       pty.STATUS\_ID AS STATUS  
         
       FROM FACILITY\_PARTY fp  
       JOIN PERSON p   
       ON fp.PARTY\_ID \= p.PARTY\_ID  
       JOIN PARTY pty   
       ON fp.PARTY\_ID \= pty.PARTY\_ID WHERE fp.ROLE\_TYPE\_ID \= '%PICKER%'  
       AND (  
       fp.THRU\_DATE IS NULL OR fp.THRU\_DATE \> CURRENT\_TIMESTAMP  
       )  
         
       ORDER BY fp.FACILITY\_ID , fp.ROLE\_TYPE\_ID, NAME;
```

---

## 9 Total Facilities That Sell the Product

### Business Problem  
Retailers want to see how many (and which) facilities (stores, warehouses, virtual sites) currently offer a product for sale.

```sql
SELECT p.PRODUCT\_ID,  
       p.PRODUCT\_NAME,  
       COUNT(DISTINCT pf.FACILITY\_ID) AS FACILITY\_COUNT,  
       GROUP\_CONCAT(  
       DISTINCT pf.FACILITY\_ID  
       ORDER BY pf.FACILITY\_ID SEPARATOR ', ') AS ' list of FACILITY\_IDs'  
       FROM PRODUCT p   
       JOIN PRODUCT\_FACILITY pf   
       ON p.PRODUCT\_ID \= pf.PRODUCT\_ID  
       WHERE (p.SALES\_DISCONTINUATION\_DATE IS NULL OR p.SALES\_DISCONTINUATION\_DATE \> CURRENT\_TIMESTAMP)  
         
       GROUP BY p.PRODUCT\_ID, p.INTERNAL\_NAME  
       ORDER BY FACILITY\_COUNT DESC;
```
       


---

## 10 Total Items in Various Virtual Facilities

### Business Problem  
Retailers need to study the relation of inventory levels of products to the type of facility it's stored at. Retrieve all inventory levels for products at locations and include the facility type Id. Do not retrieve facilities that are of type Virtual.

```sql
SELECT   
    ii.PRODUCT\_ID,  
    ii.FACILITY\_ID,  
    f.FACILITY\_TYPE\_ID,  
    SUM(ii.QUANTITY\_ON\_HAND\_TOTAL) AS QOH,  
    SUM(ii.AVAILABLE\_TO\_PROMISE\_TOTAL) AS ATP  
    FROM   
    INVENTORY\_ITEM ii  
    JOIN   
    FACILITY f ON ii.FACILITY\_ID \= f.FACILITY\_ID  
    WHERE   
    f.FACILITY\_TYPE\_ID NOT IN (  
        SELECT FACILITY\_TYPE\_ID   
        FROM FACILITY\_TYPE   
        WHERE FACILITY\_TYPE\_ID \= 'VIRTUAL\_FACILITY'   
           OR PARENT\_TYPE\_ID \= 'VIRTUAL\_FACILITY'  
    )  
    GROUP BY   
    ii.PRODUCT\_ID,   
    ii.FACILITY\_ID,   
    f.FACILITY\_TYPE\_ID;
```
    


---

## 11 Transfer Orders Without Inventory Reservation

### Business Problem  
When transferring stock between facilities, the system should reserve inventory. If it isn’t reserved, the transfer may fail or oversell.

```sql
SELECT  
    oh.ORDER\_ID AS TRANSFER\_ORDER\_ID,  
    oisg.FACILITY\_ID AS FROM\_FACILITY\_ID,  
    oh.ORIGIN\_FACILITY\_ID AS TO\_FACILITY\_ID,  
    oi.PRODUCT\_ID AS PRODUCT\_ID,  
    oi.QUANTITY AS REQUESTED\_QUANTITY,  
    COALESCE(SUM(oisgir.QUANTITY), 0\) AS RESERVED\_QUANTITY,  
    oh.ORDER\_DATE AS TRANSFER\_DATE,  
    oh.STATUS\_ID AS STATUS  
FROM  
    ORDER\_HEADER oh  
JOIN   
    ORDER\_ITEM oi   
        ON oh.ORDER\_ID \= oi.ORDER\_ID  
JOIN   
    ORDER\_ITEM\_SHIP\_GROUP oisg   
        ON oh.ORDER\_ID \= oisg.ORDER\_ID  
LEFT JOIN   
    ORDER\_ITEM\_SHIP\_GRP\_INV\_RES oisgir   
        ON oi.ORDER\_ID \= oisgir.ORDER\_ID   
        AND oi.ORDER\_ITEM\_SEQ\_ID \= oisgir.ORDER\_ITEM\_SEQ\_ID   
        AND oisg.SHIP\_GROUP\_SEQ\_ID \= oisgir.SHIP\_GROUP\_SEQ\_ID  
WHERE  
    oh.ORDER\_TYPE\_ID \= 'TRANSFER\_ORDER' AND oh.STATUS\_ID NOT IN ('ORDER\_COMPLETED', 'ORDER\_CANCELLED', 'ORDER\_REJECTED')  
    AND oi.STATUS\_ID \!= 'ITEM\_CANCELLED'  
GROUP BY  
    oh.ORDER\_ID,  
    oisg.FACILITY\_ID,  
    oh.ORIGIN\_FACILITY\_ID,  
    oi.PRODUCT\_ID,  
    oi.QUANTITY,  
    oh.ORDER\_DATE,  
    oh.STATUS\_ID  
HAVING COALESCE(SUM(oisgir.QUANTITY), 0\) \< oi.QUANTITY;
```

---

## 12 Orders Without Picklist

### Business Problem  
A picklist is necessary for warehouse staff to gather items. Orders missing a picklist might be delayed and need attention.

```sql
SELECT  
    oh.ORDER\_ID AS ORDER\_ID,  
    oh.ORDER\_DATE AS ORDER\_DATE,  
    oh.STATUS\_ID AS ORDER\_STATUS,  
    oisg.FACILITY\_ID AS FACILITY\_ID,  
    TIMESTAMPDIFF(HOUR, oh.ORDER\_DATE, CURRENT\_TIMESTAMP) AS DURATION\_IN\_HOURS  
FROM  
    ORDER\_HEADER oh  
JOIN   
    ORDER\_ITEM\_SHIP\_GROUP oisg   
        ON oh.ORDER\_ID \= oisg.ORDER\_ID  
LEFT JOIN   
    PICKLIST\_ITEM pli   
        ON oh.ORDER\_ID \= pli.ORDER\_ID   
        AND oisg.SHIP\_GROUP\_SEQ\_ID \= pli.SHIP\_GROUP\_SEQ\_ID  
WHERE  
    oh.ORDER\_TYPE\_ID \= 'SALES\_ORDER'  
    AND oh.STATUS\_ID \= 'ORDER\_APPROVED'   
    AND oisg.FACILITY\_ID IS NOT NULL  
    AND pli.PICKLIST\_BIN\_ID IS NULL  
GROUP BY  
    oh.ORDER\_ID,  
    oh.ORDER\_DATE,  
    oh.STATUS\_ID,  
    oisg.FACILITY\_ID;
```
