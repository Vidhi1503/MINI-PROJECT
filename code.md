```java
WITH final_shipping_details AS (
    SELECT 
        od."Order No", 
        bm.box_name,  
        CASE 
            WHEN bm.dimensional_weight > od."Dead Weight" THEN bm.dimensional_weight
            ELSE od."Dead Weight"
        END AS actual_weight,
        CASE 
            WHEN CASE 
                     WHEN bm.dimensional_weight > od."Dead Weight" THEN bm.dimensional_weight
                     ELSE od."Dead Weight"
                 END < 2 THEN 0.5
            WHEN CASE 
                     WHEN bm.dimensional_weight > od."Dead Weight" THEN bm.dimensional_weight
                     ELSE od."Dead Weight"
                 END < 5 THEN 2
            WHEN CASE 
                     WHEN bm.dimensional_weight > od."Dead Weight" THEN bm.dimensional_weight
                     ELSE od."Dead Weight"
                 END < 10 THEN 5
            WHEN CASE 
                     WHEN bm.dimensional_weight > od."Dead Weight" THEN bm.dimensional_weight
                     ELSE od."Dead Weight"
                 END < 20 THEN 10
            ELSE 20
        END AS weight_slab
    FROM 
        order_data AS od 
    INNER JOIN 
        box_master AS bm ON bm.box_name = od."Box size"
    INNER JOIN 
        shipping_zone AS sz ON sz."State" = od."Ship state_"
    WHERE 
        od."Order No" = 'M1421790' 
    LIMIT 1
),
min_shipping_charges AS (
    SELECT 
        weight_slab, 
        MIN("Shipping Charge(Base)") AS min_charge
    FROM 
        shipping_charges
    GROUP BY 
        weight_slab
)
SELECT 
    fsd."Order No",
    fsd.box_name,
    fsd.actual_weight,
    fsd.weight_slab,
    sc.courier_partner,
    sc."COD_value_INR",
    sc."COD_perce",
    sc."Shipping Charge(Base)",
    sc.add_charges,
    sc.add_charge_weight,
    sc.service_type,
    
    (sc."Shipping Charge(Base)" + 
   
    CASE 
        WHEN od."Order Type" = 'COD' THEN ((od."Order Amount" - od."Voucher Amount") * sc."COD_perce")
        ELSE 0
    END + 
    
    GREATEST(fsd.actual_weight - fsd.weight_slab, 0) * sc.add_charges) AS total_cost
FROM 
    final_shipping_details AS fsd
INNER JOIN 
    shipping_charges AS sc 
    ON fsd.weight_slab = sc.weight_slab
INNER JOIN 
    min_shipping_charges AS msc 
    ON sc.weight_slab = msc.weight_slab 
    AND sc."Shipping Charge(Base)" = msc.min_charge

INNER JOIN
    order_data AS od ON od."Order No" = fsd."Order No" limit 1;
```