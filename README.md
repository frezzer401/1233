--sales, product, outlet
--sales
create table stg.ext_sales
(outlet_id numeric,
 n_volume numeric,
 product_id numeric,
 sales_date timestamp); 


--product
create table stg.ext_product
(product_id numeric,
 product_name varchar,
 is_active boolean);

--outlet
create table stg.ext_outlet
(outlet_id numeric,
 outlet_name varchar,
 is_active boolean);


--рандомное наполнение
CREATE OR REPLACE FUNCTION stg.f_fill_test_data()
RETURNS void AS $$
DECLARE
    v_outlet_count INTEGER := 50;    -- всего 50 уникальных магазинов
    v_product_count INTEGER := 100;   -- всего 100 уникальных товаров
    v_sales_count INTEGER := 2000;    -- 2000 записей продаж
    v_i INTEGER;
    v_outlet_id INTEGER;
    v_product_id INTEGER;
    v_date TIMESTAMP;
    v_volume NUMERIC;
BEGIN
    -- Очистка таблиц (опционально)
    -- TRUNCATE TABLE stg.ext_sales CASCADE;
    -- TRUNCATE TABLE stg.ext_outlet CASCADE;
    -- TRUNCATE TABLE stg.ext_product CASCADE;
    
    -- 1. Заполнение справочника outlet (50 уникальных)
    FOR v_i IN 1..v_outlet_count LOOP
        INSERT INTO stg.ext_outlet (outlet_id, outlet_name, is_active)
        VALUES (
            v_i,
            'Outlet_' || v_i,
            CASE WHEN random() > 0.1 THEN true ELSE false END
        );
    END LOOP;
    
    -- 2. Заполнение справочника product (100 уникальных)
    FOR v_i IN 1..v_product_count LOOP
        INSERT INTO stg.ext_product (product_id, product_name, is_active)
        VALUES (
            v_i,
            'Product_' || v_i,
            CASE WHEN random() > 0.1 THEN true ELSE false END
        );
    END LOOP;
    
    -- 3. Заполнение таблицы sales (2000 записей)
    FOR v_i IN 1..v_sales_count LOOP
        -- Выбираем случайный outlet_id из существующих (1-50)
        v_outlet_id := floor(random() * v_outlet_count + 1)::INTEGER;
        
        -- Выбираем случайный product_id из существующих (1-100)
        v_product_id := floor(random() * v_product_count + 1)::INTEGER;
        
        -- Генерируем случайную дату за последние 365 дней
        v_date := CURRENT_TIMESTAMP - (random() * 365 || ' days')::INTERVAL;
        
        -- Генерируем объем продаж от 1 до 1000
        v_volume := (random() * 999 + 1)::NUMERIC(10,2);
        
        INSERT INTO stg.ext_sales (outlet_id, n_volume, product_id, sales_date)
        VALUES (v_outlet_id, v_volume, v_product_id, v_date);
    END LOOP;
    
    RAISE NOTICE 'Заполнение завершено. Sales: %, Outlets: %, Products: %', 
        (SELECT COUNT(*) FROM stg.ext_sales),
        (SELECT COUNT(*) FROM stg.ext_outlet),
        (SELECT COUNT(*) FROM stg.ext_product);
    
    -- Вывод статистики
    RAISE NOTICE 'Статистика по продажам:';
    RAISE NOTICE '  - Уникальных outlet_id в sales: %', 
        (SELECT COUNT(DISTINCT outlet_id) FROM stg.ext_sales);
    RAISE NOTICE '  - Уникальных product_id в sales: %', 
        (SELECT COUNT(DISTINCT product_id) FROM stg.ext_sales);
    RAISE NOTICE '  - Всего записей: %', 
        (SELECT COUNT(*) FROM stg.ext_sales);
END;
$$ LANGUAGE plpgsql;


select stg.f_fill_test_data();

select *
from stg.ext_sales;


select *
from stg.ext_product;

select *
from stg.ext_outlet;



create view bv.vw_int_b_sales_info (outlet_name,product_name,n_volume,sales_date) as
select outlet_name,product_name,n_volume,sales_date
from stg.ext_sales es
join stg.ext_product ep on ep.product_id = es.product_id and ep.is_active is true
join stg.ext_outlet eo on eo.outlet_id = es.outlet_id and eo.is_active is true
where extract(year from sales_date) = extract(year from now());




CREATE OR REPLACE FUNCTION bv.f_update_tables()
RETURNS void AS $$
DECLARE 
    table_rec RECORD;
    v_table_name TEXT;  -- Переименовали переменную
    v_view_name TEXT;   -- Переименовали переменную
BEGIN
    -- Цикл по таблицам
    FOR table_rec IN 
        SELECT table_name 
        FROM information_schema.tables 
        WHERE table_schema = 'bv' 
          AND table_name LIKE 'b_%'
          AND table_type = 'BASE TABLE'
    LOOP
        v_table_name := table_rec.table_name;  -- Используем RECORD
        v_view_name := 'vw_int_' || v_table_name;
        
        -- Проверяем существование представления
        IF EXISTS (
            SELECT 1 
            FROM information_schema.views 
            WHERE table_schema = 'bv' 
              AND table_name = v_view_name  -- Используем переменную
        ) THEN
            -- Очищаем таблицу
            EXECUTE format('TRUNCATE TABLE bv.%I', v_table_name);
            
            -- Заполняем из представления
            EXECUTE format('INSERT INTO bv.%I SELECT * FROM bv.%I', 
                          v_table_name, v_view_name);
            
            RAISE NOTICE '% обновлена из %', v_table_name, v_view_name;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;



select  bv.f_update_tables();



create view pres.vw_sales_info (outlet_name,product_name,n_volume,sales_date, calculation_date) as
select outlet_name,product_name,n_volume,sales_date, now() as calculation_date
from bv.b_sales_info;

select *
from  pres.vw_sales_info;

select *
from stg.ext_outlet eo;
select *
from stg.ext_sales eo;
select *
from stg.ext_product eo;


explain analyze
select *
from pres.vw_sales_info eo;


create view pres.vw_sales_info_aaa (outlet_name,product_name,n_volume,sales_date, calculation_date) as
select o.outlet_name,product_name,n_volume,sales_date, now() as calculation_date
from pres.vw_sales_info i
join stg.ext_outlet o on o.outlet_name = i.outlet_name;




create view pres.vw_sales_info_from (n_volume,sales_date, fromy) as
select n_volume,sales_date,'pres' as fromy
from pres.vw_sales_info_aaa
union all
select n_volume,sales_date, 'stg' as fromy
from stg.ext_sales;


create table bv.table_1 
(id numeric, n_volume numeric)

insert into bv.table_1 values (10,3000)


select *
from  bv.table_1 ;


create view bv.vw_view_sales_info (id,n_volume) as
select *
from bv.table_1;


create materialized view bv.mv_view_sales_info as 
select *
from bv.table_1;


select *
from bv.vw_view_sales_info;


refresh materialized view bv.mv_view_sales_info

select *
from bv.mv_view_sales_info;










explain 
select *
from pres.vw_sales_info_from;

distributed by
partition by
index gtin, b-tree

mat_views
view




select *
from  pres.vw_sales_info_from



























