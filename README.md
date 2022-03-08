# CRM-Analytics-customer-segmentation-with-RFM
An e-commerce company wants to segment its customers and determine marketing strategies according to these segments. For this, we will define the behavior of customers and create groups according to the clusters in these behaviors. In other words, we will take those who show common behaviors into the same groups and we will try to develop sales and marketing-specific techniques for them. for more info please read from my medium acccount: https://medium.com/@bitaazari71/crm-analytics-customer-segmentation-with-rfm-customer-lifetime-value-part-1-d1773c7c5cd9 
############################################
# PROJE: RFM ile Müşteri Segmentasyonu
############################################


###############################################################
# Veriyi Anlama ve Hazırlama
###############################################################

# 1. Online Retail II excelindeki 2010-2011 verisini okuyunuz. Oluşturduğunuz dataframe’in kopyasını oluşturunuz.
# 2. Veri setinin betimsel istatistiklerini inceleyiniz.
# 3. Veri setinde eksik gözlem varmı? Varsa hangi değişkende kaç tane eksik gözlem vardır?
# 4. Eksik gözlemleri veri setinden çıkartınız. Çıkarma işleminde ‘inplace=True’parametresini kullanınız.
# 5. Eşsiz ürün sayısı kaçtır?
# 6. Hangi üründen kaçar tane vardır?
# 7. En çok sipariş edilen 5 ürünü çoktan aza doğru sıralayınız.
# 8. Faturalardaki ‘C’ iptal edilen işlemleri göstermektedir. İptal edilen işlemleri veri setinden çıkartınız.
# 9. Fatura başına elde edilen toplam kazancı ifade eden ‘TotalPrice’ adında bir değişken oluşturunuz.


import datetime as dt
import pandas as pd
pd.set_option('display.max_columns', None)
# pd.set_option('display.max_rows', None)
# pd.set_option('display.float_format', lambda x: '%.5f' % x)

# 1. Online Retail II excelindeki 2010-2011 verisini okuyunuz. Oluşturduğunuz dataframe’in kopyasını oluşturunuz.
df_ = pd.read_excel(r"E:\bootcamp\03week\Ders_Öncesi_Notlar\online_retail_II.xlsx", sheet_name="Year 2010-2011")
df = df_.copy()
df.head()

# 2. Veri setinin betimsel istatistiklerini inceleyiniz.
df.describe([0.9,0.95,0.99]).T

# 3. Veri setinde eksik gözlem varmı? Varsa hangi değişkende kaç tane eksik gözlem vardır?
df.isnull().sum()

# 4. Eksik gözlemleri veri setinden çıkartınız. Çıkarma işleminde ‘inplace=True’parametresini kullanınız.
df.dropna(inplace=True)
#df.dropna(subset=['name', 'born']) faqat jahaie k emikhaim drop kon df.dropna(thresh=2)
# 5. Eşsiz ürün sayısı kaçtır?
df["Description"].nunique()
df["StockCode"].nunique()

df.head()

df["Description"].value_counts()
df[df.Description=="WHITE HANGING HEART T-LIGHT HOLDER"]["StockCode"].value_counts()
df[df.StockCode=="85123A"]["Description"].value_counts()


df[df.StockCode=="79323W"]["Description"].value_counts()

pd.DataFrame(df["StockCode"].value_counts()).index[0:10]

for code in list(pd.DataFrame(df["StockCode"].value_counts()).index[0:10]):
    print(code,"\n",df[df.StockCode==code]["Description"].value_counts(),"\n\n")


# 6. Hangi üründen kaçar tane vardır?
df["Description"].value_counts().head()

# 7. En çok sipariş edilen 5 ürünü çoktan aza doğru sıralayınız.
df.groupby("Description").agg({"Quantity": "sum"}).sort_values("Quantity", ascending=False).head()

# 8. Faturalardaki ‘C’ iptal edilen işlemleri göstermektedir. İptal edilen işlemleri veri setinden çıkartınız.
df = df[~df["Invoice"].str.contains("C", na=False)]

# 9. Fatura başına elde edilen toplam kazancı ifade eden ‘TotalPrice’ adında bir değişken oluşturunuz.
df["TotalPrice"] = df["Quantity"] * df["Price"]
df.head(20)
df["TotalPrice"].max()
###############################################################
#  RFM Metriklerinin Hesaplanması
###############################################################


today_date = dt.datetime(2011, 12, 11)

rfm = df.groupby('Customer ID').agg({'InvoiceDate': lambda date: (today_date - date.max()).days,
                                     'Invoice': lambda num: num.nunique(),
                                     'TotalPrice': lambda TotalPrice: TotalPrice.sum()})

rfm.head()

rfm.columns = ['recency', 'frequency', 'monetary']
rfm = rfm[rfm["monetary"] > 0]


###############################################################
# RFM Skorlarının Oluşturulması ve Tek Bir Değişkene Çevrilmesi
###############################################################

rfm["recency_score"] = pd.qcut(rfm['recency'], 5, labels=[5, 4, 3, 2, 1])
rfm["frequency_score"] = pd.qcut(rfm['frequency'].rank(method="first"), 5, labels=[1, 2, 3, 4, 5])
rfm["monetary_score"] = pd.qcut(rfm['monetary'], 5, labels=[1, 2, 3, 4, 5])
rfm['monetaryscore'] = pd.qcut(rfm['monetary'].rank(method='first'),5,labels=[5,4,3,2,1])
rfm.head()
rfm["RFM_SCORE"] = (rfm['recency_score'].astype(str) +
                    rfm['frequency_score'].astype(str))

###############################################################
#  RFM Skorlarının Segment Olarak Tanımlanması
###############################################################

# RFM isimlendirmesi
seg_map = {
    r'[1-2][1-2]': 'hibernating',
    r'[1-2][3-4]': 'at_Risk',
    r'[1-2]5': 'cant_loose',
    r'3[1-2]': 'about_to_sleep',
    r'33': 'need_attention',
    r'[3-4][4-5]': 'loyal_customers',
    r'41': 'promising',
    r'51': 'new_customers',
    r'[4-5][2-3]': 'potential_loyalists',
    r'5[4-5]': 'champions'
}


rfm['segment'] = rfm['RFM_SCORE'].replace(seg_map, regex=True)


###############################################################
 Aksiyon zamanı!
###############################################################

rfm[["segment", "recency", "frequency", "monetary"]].groupby("segment").agg(["mean", "count"])
rfm[["recency", "frequency", "monetary", "segment"]].groupby("segment").agg(["mean","min","max","count"])

new_df = pd.DataFrame()
new_df["loyal_customers"] = rfm[rfm["segment"] == "loyal_customers"].index
new_df.head()


new_df.to_csv("loyal_customers.csv")


