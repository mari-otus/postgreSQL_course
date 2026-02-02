# PIVOT-преобразования. CASE WHEN подход

Как будет выглядеть запрос с CASE WHEN в случае явного указания всех атрибутов.

```sql
select
-- Поля группировки данных
product.id, product.code, product.name, product.status, product.product_group_code,
-- Агрегаты по каждому атрибуту
-- Атрибут CardDesign
array_remove(my_array_concat(
                     CASE WHEN attribute.code = 'CardDesign' THEN 
                         COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array['']) 
                     ELSE array[''] 
                     END), 
             '') AS CardDesign,
-- Атрибут CardType
array_remove(my_array_concat(CASE WHEN attribute.code = 'CardType' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])  else array[''] END), '') AS CardType,
array_remove(my_array_concat(CASE WHEN attribute.code = 'ReissueChannel' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array['']) else array[''] END), '') AS ReissueChannel
...
-- Запрос
FROM product
   JOIN product_version ON product.id = product_version.product_id 
       AND product_version.status = 'ACTIVE' 
       AND product_version.start_date <= CURRENT_TIMESTAMP
   LEFT JOIN attribute ON product.id = attribute.product_id 
       AND attribute.version_id = product_version.id
GROUP BY product.id, product.code, product.name, product.status, product.product_group_code
ORDER BY product.code;
```

Такое решение не подхоит, т.к. коды атрибутов и их количесвто заранее неизвестно.

## Агрегация значений атрибутов

Агрегация усложняется, т.к. каждый атрибут может принимать либо строковое значение, либо массив.
Поэтому было принято решение трансформировать каждое значение в массив:
```sql
COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array['']) 
```
Массивы в этом случае получаются разной размерности, шатные функции postgreSQL могут  агрегрировать данные только для массивов с одинаковой размерностью.
Поэтому пришлось создать кастомную функцию, которая может собирать массивы разной длины:
```sql
CREATE AGGREGATE my_array_concat (anycompatiblearray) (
  sfunc = array_cat,
  stype = anycompatiblearray,
  initcond = '{}'
);
```

И последним действием удаляем из массива все пустые элементы через функцию `array_remove`:
```sql
array_remove(my_array_concat(
                     CASE WHEN attribute.code = 'CardDesign' THEN 
                         COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array['']) 
                     ELSE array[''] 
                     END), 
             '')
```

## PIVOT-преобразования. CASE WHEN подход для динамического числа столбцов

Решение заключается в создании хранимой процедуры:

```sql
CREATE OR REPLACE procedure create_view_dynamic_case_when()
LANGUAGE plpgsql
AS $$
DECLARE
ct_query text;
idx_query text default 'create index if not exists idx_pvcw_product_code ON prodcat_view_case_when USING btree (code); ';
cat_columns text default '';
attribute_codes text[];
i int;
BEGIN
    -- Получаем список уникальных атрибутов
    SELECT array_agg(attribute_code ORDER BY attribute_code)
    INTO attribute_codes
    FROM (
    select
    distinct lower(attribute.code) as attribute_code
    FROM product products
       JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = 'ACTIVE'::text AND product_version.start_date <= CURRENT_TIMESTAMP
       LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
    ) t;
    
    -- Формируем строки с агрегациями
    FOR i IN 1..array_length(attribute_codes, 1) LOOP
    cat_columns := cat_columns || ', array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = ''' || attribute_codes[i] || ''' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''''), '',''), attribute.collection_value, array[''''])::text[] else array[''''] end), '''') as ' || attribute_codes[i];
    -- Формируем запрос для создания индексов
    idx_query := idx_query || 'create index if not exists idx_pvcw_' || attribute_codes[i] || ' ON prodcat_view_case_when USING gin ("' || attribute_codes[i] || '"); ';
    END LOOP;

-- Формируем полный запрос
    ct_query := '
        DROP MATERIALIZED VIEW IF EXISTS prodcat_view_case_when;
        
        CREATE MATERIALIZED VIEW prodcat_view_case_when
        AS
        SELECT 
        products.id, products.code, products.name, products.status, products.product_group_code ' ||
        cat_columns
        || ' 
        FROM product products
           JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = ''ACTIVE''::text AND product_version.start_date <= CURRENT_TIMESTAMP
           LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
        GROUP BY products.id, products.code, products.name, products.status, products.product_group_code
        with data';

    -- Создаём MATERIALIZED VIEW
    execute ct_query;
    
    -- Создаём индексы
    execute idx_query;  

END;
$$
```

Вызываем процедуру для создания материализованного представления:

```
call create_view_dynamic_case_when();
```

В итоге будет создано материализованное представление `prodcat_view_case_when`:

| id   | code      | name                         | status | product_group_code | amount | amountcurrency | amounttransactions | amountwriteoff | autoprolongation | cardcategory      | cardclass | cardcurrency | carddesign      | cardholder | cardissuepriority | cardmode           | cardtype        | classofcredit | codemcc | cost | costcurrency | deliveryavailable | duration | durationunit | expdate | expirefornew | firstfreecardnumber | freecardsamount | insurance_company_short | isdebit | orderandperiodofstatements | packageofservicesdc            | paymenttype | photo | pintype   | premiumprc | priority | reissuechannel  | reissuemassproduct | reissueprimeproduct | reissueprivproduct | scenario | servicecode | subproductid | system     | typeofaccount     | typeofcontract | typeofcredit       | typeofguarantee |
|------|-----------|------------------------------|--------|--------------------|--------|----------------|--------------------|----------------|------------------|-------------------|-----------|--------------|-----------------|------------|-------------------|--------------------|-----------------|---------------|---------|------|--------------|-------------------|----------|--------------|---------|--------------|---------------------|-----------------|-------------------------|---------|----------------------------|--------------------------------|-------------|-------|-----------|------------|----------|-----------------|--------------------|---------------------|--------------------|----------|-------------|--------------|------------|-------------------|----------------|--------------------|-----------------|
| 1000 | 2DC001000 | Дебетовая карта Синяя        | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {}                | {}        | {}           | {BLUE}          | {}         | {}                | {}                 | {}              | {}            | {}      | {}   | {}           | {}                | {}       | {}           | {}      | {84}         | {}                  | {}              | {}                      | {}      | {}                         | {}                             | {}          | {}    | {}        | {}         | {}       | {}              | {}                 | {}                  | {}                 | {}       | {201}       | {}           | {}         | {}                | {}             | {}                 | {}              |
| 2000 | 2DC002000 | Дебетовая карта Белая        | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {MAIN/ADDITIONAL} | {MIR}     | {RUB}        | {WHITE}         | {0/1}      | {BASIC}           | {PERSONALIZED}     | {WHITE}         | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIVILEGE2,PRIME2} | {}          | {}    | {VIRTUAL} | {}         | {3}      | {OFFICE,ONLINE} | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_3} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |
| 3000 | 2DC003000 | Дебетовая карта Зелёная      | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {MAIN}            | {MIR}     | {RUB}        | {GREEN}         | {0/1}      | {BASIC}           | {NON_PERSONALIZED} | {GREEN}         | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {2}                 | {2}             | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIME2,PRIVILEGE2} | {}          | {}    | {VIRTUAL} | {}         | {1}      | {}              | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_1} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |
| 4000 | 2DC004000 | Дебетовая карта Автолюбитель | ACTIVE | 0301               | {}     | {}             | {}                 | {}             | {false}          | {}                | {}        | {}           | {}              | {}         | {}                | {}                 | {}              | {}            | {}      | {}   | {}           | {}                | {}       | {}           | {}      | {}           | {}                  | {}              | {"Страховая компания"}  | {}      | {}                         | {}                             | {}          | {}    | {}        | {}         | {}       | {}              | {}                 | {}                  | {}                 | {}       | {}          | {}           | {}         | {}                | {}             | {}                 | {}              |
| 5000 | 2DC005000 | Карта Льготная               | ACTIVE | 0301               | {}     | {}             | {}                 | {}             | {}               | {}                | {}        | {}           | {}              | {}         | {}                | {}                 | {}              | {24}          | {}      | {}   | {}           | {}                | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {}      | {}                         | {}                             | {}          | {}    | {}        | {}         | {}       | {}              | {}                 | {}                  | {}                 | {}       | {}          | {}           | {SYSTEM_2} | {}                | {Бенефит}      | {"Льготная карта"} | {ПРОЧИЕ}        |
| 6000 | 2DC006000 | Карта Льготная +             | ACTIVE | 0301               | {}     | {}             | {}                 | {}             | {}               | {MAIN/ADDITIONAL} | {MIR}     | {RUB}        | {MRNBL7D}       | {0/1}      | {BASIC}           | {PERSONALIZED}     | {BENEFITPLUS}   | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIVILEGE2,PRIME2} | {}          | {}    | {VIRTUAL} | {}         | {3}      | {OFFICE,ONLINE} | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_1} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |
| 7000 | 2DC007000 | Карта VIP                    | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {MAIN/ADDITIONAL} | {MIR}     | {RUB}        | {VIPDESIGN}     | {0/1}      | {BASIC}           | {PERSONALIZED}     | {VIP}           | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIVILEGE2,PRIME2} | {}          | {}    | {VIRTUAL} | {}         | {3}      | {OFFICE,ONLINE} | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_3} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |
| 8000 | 2DC008000 | Карта VIP +                  | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {MAIN/ADDITIONAL} | {MIR}     | {RUB}        | {VIPDESIGNPLUS} | {0/1}      | {BASIC}           | {PERSONALIZED}     | {VIP}           | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIVILEGE2,PRIME2} | {}          | {}    | {VIRTUAL} | {}         | {3}      | {OFFICE,ONLINE} | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_3} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |
| 9000 | 2DC009000 | Дебетовая карта Мишки        | ACTIVE | 1702               | {}     | {}             | {}                 | {}             | {}               | {MAIN/ADDITIONAL} | {MIR}     | {RUB}        | {MISHKI}        | {0/1}      | {BASIC}           | {PERSONALIZED}     | {MCURR_MRSTUMP} | {}            | {}      | {}   | {}           | {false}           | {}       | {}           | {}      | {}           | {}                  | {}              | {}                      | {true}  | {ByRequest}                | {MULTICARTA,PRIVILEGE2,PRIME2} | {}          | {}    | {VIRTUAL} | {}         | {3}      | {OFFICE,ONLINE} | {}                 | {}                  | {}                 | {RTL}    | {}          | {}           | {SYSTEM_2} | {CurrentAccounts} | {ДЕПОЗИТ}      | {}                 | {}              |

## Почему MATERIALIZED VIEW?

Из предусловий задачи выяснили, что данные в БД будут обновляться не часто.
Этот факт привёл к идее создания `MATERIALIZED VIEW` для сохранения результатов этого сложного PIVOT-преобразования.

## Скорость построения MATERIALIZED VIEW

Выполним запрос, который был сформирован для создания MATERIALIZED VIEW через ХП выше:

```sql
explain analyze 
SELECT 
products.id, products.code, products.name, products.status, products.product_group_code , 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'amount' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as amount, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'amountcurrency' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as amountcurrency, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'amounttransactions' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as amounttransactions, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'amountwriteoff' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as amountwriteoff, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'arm' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as arm, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'arp' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as arp, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'autoprolongation' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as autoprolongation, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'bankrupt' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as bankrupt, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'blacklist' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as blacklist, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardcategory' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardcategory, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardclass' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardclass, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardcurrency' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardcurrency, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'carddesign' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as carddesign, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardholder' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardholder, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardissuepriority' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardissuepriority, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardmode' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardmode, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardstatus' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardstatus, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cardtype' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cardtype, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'channel' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as channel, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'classofcredit' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as classofcredit, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'codemcc' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as codemcc, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'commissionscomment' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as commissionscomment, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'cost' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as cost, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'costcurrency' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as costcurrency, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'currency' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as currency, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'deliveryavailable' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as deliveryavailable, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'dep_currency' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as dep_currency, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'direction' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as direction, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'duration' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as duration, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'durationunit' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as durationunit, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'expdate' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as expdate, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'expirefornew' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as expirefornew, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'feeamountmax' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as feeamountmax, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'feeamountmin' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as feeamountmin, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'feebase' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as feebase, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'feepercent' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as feepercent, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'firstfreecardnumber' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as firstfreecardnumber, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'freecardsamount' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as freecardsamount, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'instrumenttype' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as instrumenttype, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'insurance_company_short' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as insurance_company_short, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'isallowed' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as isallowed, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'isdebit' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as isdebit, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'isvalid' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as isvalid, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'maxnumber' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as maxnumber, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'mcc' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as mcc, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'minnumber' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as minnumber, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'nameemb' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as nameemb, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'opendate' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as opendate, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'orderandperiodofstatements' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as orderandperiodofstatements, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'packageofservicesdc' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as packageofservicesdc, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'partner' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as partner, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'paymenttype' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as paymenttype, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'photo' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as photo, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'pintype' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as pintype, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'prefix' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as prefix, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'premiumprc' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as premiumprc, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'priority' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as priority, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'reissuechannel' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as reissuechannel, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'reissuemassproduct' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as reissuemassproduct, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'reissueprimeproduct' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as reissueprimeproduct, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'reissueprivproduct' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as reissueprivproduct, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'scenario' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as scenario, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'segment' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as segment, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'servicecode' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as servicecode, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sevicepackage' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sevicepackage, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'socialproductcode' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as socialproductcode, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'socialproductname' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as socialproductname, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'socialservice' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as socialservice, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sta_book_a' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sta_book_a, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sta_book_b' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sta_book_b, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sta_book_c' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sta_book_c, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sta_book_d' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sta_book_d, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'sta_book_max' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as sta_book_max, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'subproductid' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as subproductid, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'system' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as system, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'tariff' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as tariff, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'testatribute' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as testatribute, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'transtype' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as transtype, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'typecalculationscheme' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as typecalculationscheme, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'typeofaccount' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as typeofaccount, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'typeofcontract' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as typeofcontract, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'typeofcredit' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as typeofcredit, 
array_remove(product_lt.my_array_concat(CASE WHEN lower(attribute.code) = 'typeofguarantee' THEN COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value, array[''])::text[] else array[''] end), '') as typeofguarantee 
FROM product products
   JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = 'ACTIVE'::text AND product_version.start_date <= CURRENT_TIMESTAMP
   LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
GROUP BY products.id, products.code, products.name, products.status, products.product_group_code;
```

Результат **3497.693 ms**
```sql
GroupAggregate  (cost=7861.35..13337.31 rows=4346 width=2755) (actual time=125.942..3494.949 rows=4342 loops=1)
  Group Key: products.id
  ->  Sort  (cost=7861.35..7872.21 rows=4346 width=363) (actual time=125.153..178.786 rows=147968 loops=1)
        Sort Key: products.id
        Sort Method: external merge  Disk: 19920kB
        ->  Hash Right Join  (cost=471.45..7598.73 rows=4346 width=363) (actual time=3.730..53.750 rows=147968 loops=1)
              Hash Cond: (((attribute.product_id)::text = (products.id)::text) AND ((attribute.version_id)::text = (product_version.id)::text))
              ->  Seq Scan on attribute  (cost=0.00..6015.68 rows=148168 width=338) (actual time=0.004..8.812 rows=147968 loops=1)
              ->  Hash  (cost=406.26..406.26 rows=4346 width=136) (actual time=3.720..3.723 rows=4342 loops=1)
                    Buckets: 8192  Batches: 1  Memory Usage: 783kB
                    ->  Hash Join  (cost=249.79..406.26 rows=4346 width=136) (actual time=1.376..2.700 rows=4342 loops=1)
                          Hash Cond: ((product_version.product_id)::text = (products.id)::text)
                          ->  Seq Scan on product_version  (cost=0.00..145.06 rows=4346 width=74) (actual time=0.007..0.497 rows=4346 loops=1)
                                Filter: (((status)::text = 'ACTIVE'::text) AND (start_date <= CURRENT_TIMESTAMP))
                          ->  Hash  (cost=195.46..195.46 rows=4346 width=99) (actual time=1.363..1.363 rows=4342 loops=1)
                                Buckets: 8192  Batches: 1  Memory Usage: 626kB
                                ->  Seq Scan on product products  (cost=0.00..195.46 rows=4346 width=99) (actual time=0.002..0.532 rows=4342 loops=1)
Planning Time: 0.906 ms
Execution Time: 3497.693 ms
```
