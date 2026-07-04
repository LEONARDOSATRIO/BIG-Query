# BIG-Query
#mengambil kolom order id dari table orders di dataset toko_peralatan_dapur

Select order_id
from Toko_Peralatan_Dapur.olders;


#mengambil kolom order id dan sales date dari table orders di dataset toko_peralatan_dapur

Select order_id, sales_date
from Toko_Peralatan_Dapur.olders;

#mengambil semua kolom

select *
from Toko_Peralatan_Dapur.olders
where sales_date is not null;

#mengambil semua kolom hanya untuk orderan dengan status cancel
select *
from Toko_Peralatan_Dapur.olders
where status_clean = 'cancel';

#mengambil kolom sales_date, order_id, product_name, category_clean hanya untuk orderan yang categorynya alat masak
select sales_date, order_id, product_name, category_clean
from Toko_Peralatan_Dapur.olders
where category_clean = 'Alat Masak'
and status_clean = 'cancel'
order by sales_date
limit 5;

#total sales selama tahun 2025
select SUM(total_sales) as revenue, AVG(total_sales) as rerata_total_sales
from Toko_Peralatan_Dapur.olders
where status_clean = 'complete';

#total sales per kategori
select category_clean, sum(total_sales) as revenue
from Toko_Peralatan_Dapur.olders
where status_clean = 'complete'
group by category_clean
order by revenue desc;

#Mengelompokkan penjualan per order berdasarkan penjulan kurang dari 500.00 > kecil
#lebih dari 500.00 tapi kurang dari 2.000.000 > sedang
#lebih dari 2 juta masuk kategori besar
with sementara as (
                      select
                      order_id,
                      case
                      when total_sales > 2000000 then 'besar'
                      when total_sales > 500000 then 'sedang'
                      else 'kecil'
                      end as category_spending
                      from Toko_Peralatan_Dapur.olders
                      where sales_date is not null
)
select
category_spending,
count(order_id)
from sementara
group by category_spending;

#No. 1 Total ongkos kirim dan rata-rata ongkos kirim per pesanan tahun 2025
SELECT
    SUM(shipping_fee) AS total_ongkos_kirim,
    AVG(shipping_fee) AS rata_rata_ongkos_kirim
FROM `Toko_Peralatan_Dapur.olders`
WHERE EXTRACT(YEAR FROM sales_date) = 2025
AND status_clean = 'complete';

# No. 2 Top 5 produk berdasarkan unit terjual
SELECT
product_name,
SUM(quantity) AS total_unit
FROM Toko_Peralatan_Dapur.olders
WHERE status_clean='complete'
GROUP BY product_name
ORDER BY total_unit DESC
LIMIT 5;
#Top 5 berdasarkan revenue
SELECT
product_name,
SUM(total_sales) AS revenue
FROM Toko_Peralatan_Dapur.olders
WHERE status_clean='complete'
GROUP BY product_name
ORDER BY revenue DESC
LIMIT 5;

# No.3 Jumlah pesanan dan revenue completed selama Q4 (Oktober–Desember)
SELECT
    COUNT(DISTINCT order_id) AS jumlah_pesanan,
    SUM(total_sales) AS total_revenue
FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean = 'complete'
  AND EXTRACT(YEAR FROM sales_date) = 2025
  AND EXTRACT(MONTH FROM sales_date) BETWEEN 10 AND 12;

#No. 4 Kota dengan rata-rata ongkos kirim tertinggi dan selisih dengan yang terendah
#4a Rata-rata ongkir setiap kota
SELECT
    city_clean,
    AVG(shipping_fee) AS rata_rata_ongkir
FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean = 'complete'
GROUP BY city_clean
ORDER BY rata_rata_ongkir DESC;

#4b Selisih ongkir kota termahal dan termurah

WITH rata_ongkir AS (
    SELECT
        city_clean,
        AVG(shipping_fee) AS rata_rata_ongkir
    FROM `Toko_Peralatan_Dapur.olders`
    WHERE status_clean = 'complete'
    GROUP BY city_clean
)

SELECT
    MAX(rata_rata_ongkir) AS ongkir_tertinggi,
    MIN(rata_rata_ongkir) AS ongkir_terendah,
    MAX(rata_rata_ongkir) - MIN(rata_rata_ongkir) AS selisih_ongkir
FROM rata_ongkir;

#No 5 Total refund dan persentasenya terhadap gross sales
SELECT
    SUM(CASE
            WHEN status_clean = 'refund'
            THEN total_sales
            ELSE 0
        END) AS total_refund,

    SUM(total_sales) AS gross_sales,

    ROUND(
        SAFE_DIVIDE(
            SUM(CASE
                    WHEN status_clean='refund'
                    THEN total_sales
                    ELSE 0
                END),
            SUM(total_sales)
        ) * 100,
        2
    ) AS persen_refund
FROM `Toko_Peralatan_Dapur.olders`;

#No 6 Lima produk dengan rata-rata quantity tertinggi (minimal 50 order completed)
SELECT
    product_name,
    COUNT(DISTINCT order_id) AS jumlah_order,
    AVG(quantity) AS rata_rata_quantity
FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean = 'complete'
GROUP BY product_name
HAVING COUNT(DISTINCT order_id) >= 50
ORDER BY rata_rata_quantity DESC
LIMIT 5;

#No 7 Bulan dengan revenue completed tertinggi untuk setiap kategori
WITH revenue_bulanan AS (
  SELECT
    category_clean,
    EXTRACT(MONTH FROM sales_date) AS bulan,
    SUM(total_sales) AS revenue
  FROM `Toko_Peralatan_Dapur.olders`
  WHERE status_clean = 'complete'
  GROUP BY category_clean, bulan
),

ranking AS (
  SELECT
    category_clean,
    bulan,
    revenue,
    RANK() OVER (
      PARTITION BY category_clean
      ORDER BY revenue DESC
    ) AS rank_bulan
  FROM revenue_bulanan
)

SELECT
  category_clean,
  bulan,
  revenue
FROM ranking
WHERE rank_bulan = 1
ORDER BY category_clean;

#No 8 Berapa produk yang menyumbang 80% revenue (Pareto)
WITH revenue_produk AS (

SELECT
product_name,
SUM(total_sales) AS revenue
FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean='complete'
GROUP BY product_name

),

pareto AS (

SELECT
product_name,
revenue,

SUM(revenue)
OVER(ORDER BY revenue DESC)
AS kumulatif,

SUM(revenue)
OVER()
AS total_revenue

FROM revenue_produk

)

SELECT
COUNT(*) AS jumlah_produk_80_persen
FROM pareto
WHERE kumulatif <= total_revenue*0.8;

#Jika ingin melihat nama produknya
WITH revenue_produk AS (

SELECT
    product_name,
    SUM(total_sales) AS revenue
FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean = 'complete'
GROUP BY product_name

),

pareto AS (

SELECT
    product_name,
    revenue,

    SUM(revenue) OVER(ORDER BY revenue DESC) AS kumulatif,

    SUM(revenue) OVER() AS total_revenue

FROM revenue_produk

)

SELECT
    product_name,
    revenue,
    kumulatif,
    total_revenue
FROM pareto
WHERE kumulatif <= total_revenue * 0.8
ORDER BY revenue DESC;

# No 9 Pelanggan dengan rata-rata jeda hari antar pembelian
WITH pembelian AS (

SELECT
customer_name_clean,
sales_date,

LAG(sales_date)
OVER(
PARTITION BY customer_name_clean
ORDER BY sales_date
) AS tanggal_sebelumnya

FROM `Toko_Peralatan_Dapur.olders`
WHERE status_clean='complete'

),

selisih AS (

SELECT
customer_name_clean,

DATE_DIFF(
sales_date,
tanggal_sebelumnya,
DAY
) AS jeda_hari

FROM pembelian
WHERE tanggal_sebelumnya IS NOT NULL

)

SELECT
customer_name_clean,
AVG(jeda_hari) AS rata_rata_jeda
FROM selisih
GROUP BY customer_name_clean
HAVING COUNT(*)>=5
ORDER BY rata_rata_jeda
LIMIT 5;

#No 10 
WITH produk AS (

SELECT

product_name,

COUNTIF(status_clean='refund') AS jumlah_refund,

COUNT(*) AS total_transaksi,

SUM(
CASE
WHEN status_clean='refund'
THEN total_sales
ELSE 0
END
) AS nilai_refund

FROM `Toko_Peralatan_Dapur.olders`

GROUP BY product_name

),

hasil AS (

SELECT

product_name,

jumlah_refund,

total_transaksi,

SAFE_DIVIDE(jumlah_refund,total_transaksi)
AS refund_rate,

nilai_refund,

nilai_refund*0.95
AS potensi_revenue_diselamatkan

FROM produk

)

SELECT
product_name,
refund_rate,
potensi_revenue_diselamatkan
FROM hasil
ORDER BY refund_rate DESC
LIMIT 1;
