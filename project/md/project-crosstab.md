# PIVOT-преобразования. CROSSTAB подход

Функция CROSSTAB из расширения tablefunc предоставляет возможность для создания PIVOT-преобразований.

Для начала необходимо установить расширение, если оно ещё не установлено:
```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
```

```sql
SELECT * FROM crosstab(
                      'SELECT 
                      products.id, products.code, products.name, products.status, products.product_group_code,
                      attribute.code, 
                      array_remove(my_array_concat(COALESCE(string_to_array(NULLIF(attribute.standard, ''''), '',''), attribute.collection_value, array[''''])[]), '''')
                      FROM product products
                         JOIN product_version ON products.id = product_version.product_id AND product_version.status = ''ACTIVE'' AND product_version.start_date <= CURRENT_TIMESTAMP
                         LEFT JOIN attribute ON products.id = attribute.product_id AND attribute.version_id = product_version.id
                      GROUP BY products.id, products.code, products.name, products.status, products.product_group_code, attribute.code
                      ORDER BY 1,2',
                      '
                      select
                      distinct attribute.code
                      FROM product products
                         JOIN product_version ON products.id = product_version.product_id AND product_version.status = ''ACTIVE'' AND product_version.start_date <= CURRENT_TIMESTAMP
                         LEFT JOIN attribute ON products.id = attribute.product_id AND attribute.version_id = product_version.id
                         order by 1
                      '
              ) AS ct (
                       "id" text,
                       "code" text,
                       "name" text,
                       "status" text,
                       "product_group_code" text,
                       "amount" text[],
                       "amountCurrency" TEXT[],
                       "amountTransactions" TEXT[],
                       "amountWriteOff" TEXT[],
                       "autoProlongation" TEXT[],
                       "CardCategory" TEXT[],
                       "CardClass" TEXT[],
                       "CardCurrency" TEXT[],
                       "CardDesign" TEXT[],
                       "CardHolder" TEXT[],
                       "CardIssuePriority" TEXT[],
                       "CardMode" TEXT[],
                       "CardType" TEXT[],
                       "ClassOfCredit" TEXT[],
                       "CodeMCC" TEXT[],
                       "cost" TEXT[],
                       "costCurrency" TEXT[],
                       "DeliveryAvailable" TEXT[],
                       "duration" TEXT[],
                       "durationUnit" TEXT[],
                       "ExpDate" TEXT[],
                       "ExpireForNew" TEXT[],
                       "firstFreeCardNumber" TEXT[],
                       "freeCardsAmount" TEXT[],
                       "insurance_company_short" TEXT[],
                       "IsDebit" TEXT[],
                       "OrderAndPeriodOfStatements" TEXT[],
                       "PackageOfServicesDC" TEXT[],
                       "paymentType" TEXT[],
                       "Photo" TEXT[],
                       "PinType" TEXT[],
                       "premiumPrc" TEXT[],
                       "Priority" TEXT[],
                       "ReissueChannel" TEXT[],
                       "ReissueMassProduct" TEXT[],
                       "ReissuePrimeProduct" TEXT[],
                       "ReissuePrivProduct" TEXT[],
                       "Scenario" TEXT[],
                       "ServiceCode" TEXT[],
                       "SubProductID" TEXT[],
                       "system" TEXT[],
                       "TypeOfAccount" TEXT[],
                       "TypeOfContract" TEXT[],
                       "TypeOfCredit" TEXT[],
                       "TypeOfGuarantee" TEXT);
```
Здесь также используется кастомная агрегация массивов, как и в случае CASE WHEN преобразования.

Одно из ограничений CROSSTAB — необходимость заранее определять структуру результирующей таблицы. 
Чтобы обойти это ограничение, можно использовать динамический SQL.

## PIVOT-преобразования. CROSSTAB подход для динамического числа столбцов

Решение заключается в создании хранимой процедуры:

```sql
CREATE OR REPLACE procedure create_view_dynamic_crosstab()
    LANGUAGE plpgsql
AS $$
DECLARE
    ct_query text;
    idx_query text default 'create index if not exists idx_product_code ON prodcat_view_crosstab USING btree (code); ';
    cat_columns text;
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

-- Формируем строку с определением столбцов
    cat_columns := '"id" text, "code" text, "name" text, "status" text, "product_group_code" text';

    FOR i IN 1..array_length(attribute_codes, 1) LOOP
            cat_columns := cat_columns || ', "' || attribute_codes[i] || '" TEXT[]';
-- Формируем запрос для создания индексов
            idx_query := idx_query || 'create index if not exists idx_' || attribute_codes[i] || ' ON prodcat_view_crosstab USING gin ("' || attribute_codes[i] || '"); ';
        END LOOP;

-- Формируем полный запрос
    ct_query := '
        DROP MATERIALIZED VIEW IF EXISTS prodcat_view_crosstab;
        
        CREATE MATERIALIZED VIEW prodcat_view_crosstab
        AS
        SELECT * FROM crosstab(
        ''SELECT 
        products.id, products.code, products.name, products.status, products.product_group_code,
        lower(attribute.code), 
        array_remove(my_array_concat(COALESCE(string_to_array(NULLIF(attribute.standard, ''''''''), '''',''''), attribute.collection_value, array[''''''''])::text[]), '''''''')
        FROM product products
           JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = ''''ACTIVE''''::text AND product_version.start_date <= CURRENT_TIMESTAMP
           LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
        GROUP BY products.id, products.code, products.name, products.status, products.product_group_code, attribute.code
        ORDER BY 1,2''::text,
        ''
        select
        distinct lower(attribute.code)
        FROM product products
           JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = ''''ACTIVE''''::text AND product_version.start_date <= CURRENT_TIMESTAMP
           LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
           order by 1
        ''::text
        ) AS ct (' || cat_columns || ')
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
CALL create_view_dynamic_crosstab();
```

В итоге будет создано материализованное представление `prodcat_view_crosstab`:

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

## Скорость построения MATERIALIZED VIEW

Выполним запрос, который был сформирован для создания MATERIALIZED VIEW через ХП выше:

```sql
explain analyze
SELECT * FROM crosstab(
                      'SELECT 
                      products.id, products.code, products.name, products.status, products.product_group_code,
                      lower(attribute.code), 
                      array_remove(product_lt.my_array_concat(COALESCE(string_to_array(NULLIF(attribute.standard, ''''), '',''), attribute.collection_value, array[''''])::text[]), '''')
                      FROM product products
                         JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = ''ACTIVE''::text AND product_version.start_date <= CURRENT_TIMESTAMP
                         LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
                      GROUP BY products.id, products.code, products.name, products.status, products.product_group_code, attribute.code
                      ORDER BY 1,2'::text,
                      '
                      select
                      distinct lower(attribute.code)
                      FROM product products
                         JOIN product_version ON products.id::text = product_version.product_id::text AND product_version.status::text = ''ACTIVE''::text AND product_version.start_date <= CURRENT_TIMESTAMP
                         LEFT JOIN attribute ON products.id::text = attribute.product_id::text AND attribute.version_id::text = product_version.id::text
                         order by 1
                      '::text
              ) AS ct ("id" text, "code" text, "name" text, "status" text, "product_group_code" text, "123" TEXT[], "amount" TEXT[], "amountcurrency" TEXT[], "amounttransactions" TEXT[], "amountwriteoff" TEXT[], "arm" TEXT[], "arp" TEXT[], "autoprolongation" TEXT[], "bankrupt" TEXT[], "blacklist" TEXT[], "cardcategory" TEXT[], "cardclass" TEXT[], "cardcurrency" TEXT[], "carddesign" TEXT[], "cardholder" TEXT[], "cardissuepriority" TEXT[], "cardmode" TEXT[], "cardstatus" TEXT[], "cardtype" TEXT[], "channel" TEXT[], "classofcredit" TEXT[], "codemcc" TEXT[], "commissionscomment" TEXT[], "cost" TEXT[], "costcurrency" TEXT[], "currency" TEXT[], "deliveryavailable" TEXT[], "dep_currency" TEXT[], "direction" TEXT[], "duration" TEXT[], "durationunit" TEXT[], "expdate" TEXT[], "expirefornew" TEXT[], "feeamountmax" TEXT[], "feeamountmin" TEXT[], "feebase" TEXT[], "feepercent" TEXT[], "firstfreecardnumber" TEXT[], "freecardsamount" TEXT[], "instrumenttype" TEXT[], "insurance_company_short" TEXT[], "isallowed" TEXT[], "isdebit" TEXT[], "isvalid" TEXT[], "maxnumber" TEXT[], "mcc" TEXT[], "minnumber" TEXT[], "nameemb" TEXT[], "opendate" TEXT[], "orderandperiodofstatements" TEXT[], "packageofservicesdc" TEXT[], "partner" TEXT[], "paymenttype" TEXT[], "photo" TEXT[], "pintype" TEXT[], "prefix" TEXT[], "premiumprc" TEXT[], "priority" TEXT[], "reissuechannel" TEXT[], "reissuemassproduct" TEXT[], "reissueprimeproduct" TEXT[], "reissueprivproduct" TEXT[], "scenario" TEXT[], "segment" TEXT[], "servicecode" TEXT[], "sevicepackage" TEXT[], "socialproductcode" TEXT[], "socialproductname" TEXT[], "socialservice" TEXT[], "sta_book_a" TEXT[], "sta_book_b" TEXT[], "sta_book_c" TEXT[], "sta_book_d" TEXT[], "sta_book_max" TEXT[], "subproductid" TEXT[], "system" TEXT[], "tariff" TEXT[], "testatribute" TEXT[], "transtype" TEXT[], "typecalculationscheme" TEXT[], "typeofaccount" TEXT[], "typeofcontract" TEXT[], "typeofcredit" TEXT[], "typeofguarantee" TEXT[]);
```

Результат **658.913 ms**
```sql
Function Scan on crosstab ct  (cost=0.00..10.00 rows=1000 width=2848) (actual time=656.201..657.189 rows=4342 loops=1)
Planning Time: 0.037 ms
Execution Time: 658.913 ms
```