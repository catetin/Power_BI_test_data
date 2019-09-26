3 CSV files below are from [**Brazilian E-Commerce Public Dataset by Olist**](https://www.kaggle.com/olistbr/brazilian-ecommerce)

- [olist_order_items_dataset.csv](https://github.com/catetin/Databricks_Handson_Seminar/raw/master/olist_order_items_dataset.csv)
- [olist_products_dataset.csv](https://github.com/catetin/Databricks_Handson_Seminar/blob/master/olist_products_dataset.csv)
- [product_category_name_translation.csv](https://github.com/catetin/Databricks_Handson_Seminar/blob/master/product_category_name_translation.csv)

Below CSV file has been merged from above files for Power BI visualization

- table for PBI.csv

Here's the [Power BI Dashboard](https://app.powerbi.com/view?r=eyJrIjoiOGFmOTM5NDEtNTZkMi00MmYxLWFmZDAtYzgzNWYxNjFlN2FlIiwidCI6IjYxNTc5NTU5LWNiM2EtNGZmYy1hOTVmLTkwNzYzMmJhNDRlOCJ9)

My Qiita account; https://qiita.com/Catetin0310

------------------------------------------------------------------------------

#### ライブラリのインポート
今回の用途であればこれだけでOKです。

```python
from pyspark.sql.functions import col, desc, to_timestamp, month, year, dayofweek, hour
```


#### データフレームの読み込み
先ほど作成したデータフレームの名称を変数に格納。

```python
orders = 'olist_order_items_dataset_csv'
items = 'olist_products_dataset_csv'
translation = 'product_category_name_translation_csv'
```

それぞれのデータフレームをSparkで読み込みます。

```python
t1 = spark.read.table(orders)
t2 = spark.read.table(items)
t3 = spark.read.table(translation)
```

日付データの書式をタイムスタンプに、結合に使用するキーの名称を整理しておきます。

```python
t1 = t1.withColumn('time', to_timestamp(t1.shipping_limit_date, 'yyyy/MM/dd HH:mm'))
t2 = t2.withColumnRenamed('product_id', 'product_id2')
t3 = t3.withColumnRenamed('product_category_name', 'product_category_name2')
```


#### データフレームの結合
フレーム結合のために、必要な列名を確認。

```python
t1.printSchema()
t2.printSchema()
t3.printSchema()
```

とりあえず3つのデータフレームを結合します。カテゴリの英語名が長いので、名称を変えておきます。

```python
temp_t = t1\
.join(t2, t1.product_id == t2.product_id2, 'inner')\
.join(t3, t2.product_category_name == t3.product_category_name2, 'inner')\
.withColumnRenamed('product_category_name_english', 'category')\
```

結合後のデータフレームのスキーマを確認しておきます。

```python
temp_t.printSchema()
```

#### ビューの作成
集計用の一時ビューとして、```temp_view``` を作成します。

```python
temp_t.select('order_id'\
              , 'product_id'\
              , 'category'\
              , 'price'\
              , 'freight_value'\
              , 'time'\
              , hour('time').alias('hour')\
              , dayofweek('time').alias('dayofweek')\
              , month('time').alias('month')\
              , year('time').alias('year'))\
.createOrReplaceTempView('temp_view')
```


#### SQL で集計 → 可視化
以上の操作で、マジックコマンド ```%sql``` を使用してクエリを直接書けるようになりました。

```python
%sql
SELECT * FROM temp_view
```

テーブルの詳細は以下のコードで確認できます。

```python
%sql
DESCRIBE temp_view
```



マジックコマンドを使用してのクエリでも、一時ビューを作成することができます。集計結果を格納するビュー ```agged_view``` をこの方法で作成してみましょう。(今回は使用しませんが、カテゴリ別売上のランキングのカラムを追加しています)

```python
%sql
CREATE OR REPLACE TEMPORARY VIEW agged_view AS
  SELECT
      category as cat
      , sum(price) / count(order_id) as cat_price_ave
      , sum(freight_value) / count(order_id) as cat_freight_ave
      , sum(freight_value) / (sum(freight_value) + sum(price)) as category_freight_ratio
      , RANK () OVER (ORDER BY sum(price) DESC) as ranking
  FROM
      temp_view
  GROUP BY
      category
```

agged_view の一部を抜き出し、可視化します。平均単価と送料の割合が同じくらいのスケールに収まるように、送料の割合を1000倍しておきます。

```python 
%sql
SELECT
  cat
  , category_freight_ratio * 1000 as freight_ratio_x_1000
  , cat_price_ave
FROM agged_view
WHERE ranking > 0 AND ranking <= 10
ORDER BY cat_price_ave DESC
```
