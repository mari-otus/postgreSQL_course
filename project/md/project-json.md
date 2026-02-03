# PIVOT-преобразования. JSON подход

Этот подход не является чистым PIVOT-преобразованием.
В этом случае все атрибуты агрегируются в json.

```sql
SELECT products.id AS product_id,
       products.code AS product_code,
       products.name AS product_name,
       products.status AS product_status,
       products.product_group_code,
       jsonb_strip_nulls(jsonb_object_agg(COALESCE(lower(attribute.code), 'code_empty'), COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value))) AS attributes
FROM product products
         JOIN product_version ON products.id = product_version.product_id 
                                     AND product_version.status = 'ACTIVE' 
                                     AND product_version.start_date <= CURRENT_TIMESTAMP
         LEFT JOIN attribute ON products.id = attribute.product_id 
                                    AND attribute.version_id = product_version.id
GROUP BY products.id, products.code, products.name, products.status, products.product_group_code
ORDER BY products.code;
```

Получаем такой результат:

| product_code | product_name                 | product_status | product_group_code | attributes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|--------------|------------------------------|----------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 2DC001000    | Дебетовая карта Синяя        | ACTIVE         | 1702               | {"carddesign": ["BLUE"], "servicecode": ["201"], "expirefornew": ["84"]}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| 2DC002000    | Дебетовая карта Белая        | ACTIVE         | 1702               | {"system": ["SYSTEM_3"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["PERSONALIZED"], "cardtype": ["WHITE"], "priority": ["3"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["WHITE"], "cardholder": ["0/1"], "cardcategory": ["MAIN/ADDITIONAL"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "reissuechannel": ["OFFICE", "ONLINE"], "typeofcontract": ["ДЕПОЗИТ"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "packageofservicesdc": ["MULTICARTA", "PRIVILEGE2", "PRIME2"], "orderandperiodofstatements": ["ByRequest"]}          |
| 2DC003000    | Дебетовая карта Зелёная      | ACTIVE         | 1702               | {"system": ["SYSTEM_1"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["NON_PERSONALIZED"], "cardtype": ["GREEN"], "priority": ["1"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["GREEN"], "cardholder": ["0/1"], "cardcategory": ["MAIN"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "typeofcontract": ["ДЕПОЗИТ"], "freecardsamount": ["2"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "firstfreecardnumber": ["2"], "packageofservicesdc": ["MULTICARTA", "PRIME2", "PRIVILEGE2"], "orderandperiodofstatements": ["ByRequest"]} |
| 2DC004000    | Дебетовая карта Автолюбитель | ACTIVE         | 0301               | {"autoprolongation": ["false"], "insurance_company_short": ["Страховая компания"]}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 2DC005000    | Карта Льготная               | ACTIVE         | 0301               | {"system": ["SYSTEM_2"], "typeofcredit": ["Льготная карта"], "classofcredit": ["24"], "typeofcontract": ["Бенефит"], "typeofguarantee": ["ПРОЧИЕ"]}                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| 2DC006000    | Карта Льготная +             | ACTIVE         | 0301               | {"system": ["SYSTEM_1"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["PERSONALIZED"], "cardtype": ["BENEFITPLUS"], "priority": ["3"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["MRNBL7D"], "cardholder": ["0/1"], "cardcategory": ["MAIN/ADDITIONAL"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "reissuechannel": ["OFFICE", "ONLINE"], "typeofcontract": ["ДЕПОЗИТ"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "packageofservicesdc": ["MULTICARTA", "PRIVILEGE2", "PRIME2"], "orderandperiodofstatements": ["ByRequest"]}  |
| 2DC007000    | Карта VIP                    | ACTIVE         | 1702               | {"system": ["SYSTEM_3"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["PERSONALIZED"], "cardtype": ["VIP"], "priority": ["3"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["VIPDESIGN"], "cardholder": ["0/1"], "cardcategory": ["MAIN/ADDITIONAL"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "reissuechannel": ["OFFICE", "ONLINE"], "typeofcontract": ["ДЕПОЗИТ"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "packageofservicesdc": ["MULTICARTA", "PRIVILEGE2", "PRIME2"], "orderandperiodofstatements": ["ByRequest"]}        |
| 2DC008000    | Карта VIP +                  | ACTIVE         | 1702               | {"system": ["SYSTEM_3"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["PERSONALIZED"], "cardtype": ["VIP"], "priority": ["3"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["VIPDESIGNPLUS"], "cardholder": ["0/1"], "cardcategory": ["MAIN/ADDITIONAL"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "reissuechannel": ["OFFICE", "ONLINE"], "typeofcontract": ["ДЕПОЗИТ"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "packageofservicesdc": ["MULTICARTA", "PRIVILEGE2", "PRIME2"], "orderandperiodofstatements": ["ByRequest"]}    |
| 2DC009000    | Дебетовая карта Мишки        | ACTIVE         | 1702               | {"system": ["SYSTEM_2"], "isdebit": ["true"], "pintype": ["VIRTUAL"], "cardmode": ["PERSONALIZED"], "cardtype": ["MCURR_MRSTUMP"], "priority": ["3"], "scenario": ["RTL"], "cardclass": ["MIR"], "carddesign": ["MISHKI"], "cardholder": ["0/1"], "cardcategory": ["MAIN/ADDITIONAL"], "cardcurrency": ["RUB"], "typeofaccount": ["CurrentAccounts"], "reissuechannel": ["OFFICE", "ONLINE"], "typeofcontract": ["ДЕПОЗИТ"], "cardissuepriority": ["BASIC"], "deliveryavailable": ["false"], "packageofservicesdc": ["MULTICARTA", "PRIVILEGE2", "PRIME2"], "orderandperiodofstatements": ["ByRequest"]} |

По аналогии с двумя другими преобразованиями, создаём `MATERIALIZED VIEW`:

```sql
CREATE MATERIALIZED VIEW prodcat_view
AS SELECT attr.product_code,
          attr.product_name,
          attr.product_status,
          attr.product_group_code,
          attr.attributes
   FROM ( SELECT products.id AS product_id,
                 products.code AS product_code,
                 products.name AS product_name,
                 products.status AS product_status,
                 products.product_group_code,
                 jsonb_strip_nulls(jsonb_object_agg(COALESCE(lower(attribute.code), 'code_empty'), COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value))) AS attributes
          FROM product products
                   JOIN product_version ON products.id = product_version.product_id AND product_version.status = 'ACTIVE' AND product_version.start_date <= CURRENT_TIMESTAMP
                   LEFT JOIN attribute ON products.id = attribute.product_id AND attribute.version_id = product_version.id
          GROUP BY products.id, products.code, products.name, products.status, products.product_group_code
          ORDER BY products.code) attr
WITH DATA;
```

## Скорость построения MATERIALIZED VIEW

Выполним запрос, который был сформирован для создания MATERIALIZED VIEW выше:

```sql
explain analyze
SELECT attr.product_code,
       attr.product_name,
       attr.product_status,
       attr.product_group_code,
       attr.attributes
FROM ( SELECT products.id AS product_id,
              products.code AS product_code,
              products.name AS product_name,
              products.status AS product_status,
              products.product_group_code,
              jsonb_strip_nulls(jsonb_object_agg(COALESCE(lower(attribute.code), 'code_empty'), COALESCE(string_to_array(NULLIF(attribute.standard, ''), ','), attribute.collection_value))) AS attributes
       FROM product products
                JOIN product_version ON products.id = product_version.product_id AND product_version.status = 'ACTIVE' AND product_version.start_date <= CURRENT_TIMESTAMP
                LEFT JOIN attribute ON products.id = attribute.product_id AND attribute.version_id = product_version.id
       GROUP BY products.id, products.code, products.name, products.status, products.product_group_code
       ORDER BY products.code) attr;
```

Результат **384.524 ms**
```sql
Subquery Scan on attr  (cost=8254.34..8308.67 rows=4346 width=94) (actual time=379.384..381.819 rows=4342 loops=1)
  ->  Sort  (cost=8254.34..8265.21 rows=4346 width=131) (actual time=379.381..381.443 rows=4342 loops=1)
        Sort Key: products.code
        Sort Method: external merge  Disk: 4968kB
        ->  GroupAggregate  (cost=7861.35..7991.73 rows=4346 width=131) (actual time=126.766..366.627 rows=4342 loops=1)
              Group Key: products.id
              ->  Sort  (cost=7861.35..7872.21 rows=4346 width=363) (actual time=126.683..172.419 rows=147968 loops=1)
                    Sort Key: products.id
                    Sort Method: external merge  Disk: 19920kB
                    ->  Hash Right Join  (cost=471.45..7598.73 rows=4346 width=363) (actual time=3.663..54.818 rows=147968 loops=1)
                          Hash Cond: (((attribute.product_id)::text = (products.id)::text) AND ((attribute.version_id)::text = (product_version.id)::text))
                          ->  Seq Scan on attribute  (cost=0.00..6015.68 rows=148168 width=338) (actual time=0.005..8.710 rows=147968 loops=1)
                          ->  Hash  (cost=406.26..406.26 rows=4346 width=136) (actual time=3.652..3.655 rows=4342 loops=1)
                                Buckets: 8192  Batches: 1  Memory Usage: 783kB
                                ->  Hash Join  (cost=249.79..406.26 rows=4346 width=136) (actual time=1.355..2.631 rows=4342 loops=1)
                                      Hash Cond: ((product_version.product_id)::text = (products.id)::text)
                                      ->  Seq Scan on product_version  (cost=0.00..145.06 rows=4346 width=74) (actual time=0.005..0.489 rows=4346 loops=1)
                                            Filter: (((status)::text = 'ACTIVE'::text) AND (start_date <= CURRENT_TIMESTAMP))
                                      ->  Hash  (cost=195.46..195.46 rows=4346 width=99) (actual time=1.344..1.345 rows=4342 loops=1)
                                            Buckets: 8192  Batches: 1  Memory Usage: 626kB
                                            ->  Seq Scan on product products  (cost=0.00..195.46 rows=4346 width=99) (actual time=0.002..0.505 rows=4342 loops=1)
Planning Time: 0.357 ms
Execution Time: 384.524 ms
```
