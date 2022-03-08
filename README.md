# CRM-Analytics-customer-segmentation-with-RFM
An e-commerce company wants to segment its customers and determine marketing strategies according to these segments. For this, we will define the behavior of customers and create groups according to the clusters in these behaviors. In other words, we will take those who show common behaviors into the same groups and we will try to develop sales and marketing-specific techniques for them. for more info please read from my medium acccount: https://medium.com/@bitaazari71/crm-analytics-customer-segmentation-with-rfm-customer-lifetime-value-part-1-d1773c7c5cd9 



###############################################################
# DATA PREPRATION
###############################################################


import datetime as dt
import pandas as pd
pd.set_option('display.max_columns', None)
# pd.set_option('display.max_rows', None)
# pd.set_option('display.float_format', lambda x: '%.5f' % x)

#READING DATA
df_ = pd.read_excel(r"E:\bootcamp\03week\Ders_Öncesi_Notlar\online_retail_II.xlsx", sheet_name="Year 2010-2011")
df = df_.copy()
df.head()

# 2. DESCRIPTIVE STATISTIC
df.describe([0.9,0.95,0.99]).T

# 3. N OF MISSING VALUE
df.isnull().sum()

# 4. REMOVE MISSING VALUE
df.dropna(inplace=True)
#df.dropna(subset=['name', 'born']) faqat jahaie k emikhaim drop kon df.dropna(thresh=2)
# 5. No OR UNIQUE VALUE IN DATASET
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


# 6. NUMBER OF PRODUCT
df["Description"].value_counts().head()

# 7. CHECK THE MOST ORDERED PRODUCT
df.groupby("Description").agg({"Quantity": "sum"}).sort_values("Quantity", ascending=False).head()

# 8. WE ARE REMOVING INVOICES WHICH INCLUDE 'C'
df = df[~df["Invoice"].str.contains("C", na=False)]

# 9.CALCULATE TOTAL PRICE
df["TotalPrice"] = df["Quantity"] * df["Price"]
df.head(20)
df["TotalPrice"].max()
###############################################################
#  RFM CALCULATION
###############################################################


today_date = dt.datetime(2011, 12, 11)

rfm = df.groupby('Customer ID').agg({'InvoiceDate': lambda date: (today_date - date.max()).days,
                                     'Invoice': lambda num: num.nunique(),
                                     'TotalPrice': lambda TotalPrice: TotalPrice.sum()})

rfm.head()

rfm.columns = ['recency', 'frequency', 'monetary']
rfm = rfm[rfm["monetary"] > 0]


###############################################################
# TURN RFM METRICS INTO LABEL FROM 1-5 WHITH QCUT
###############################################################

rfm["recency_score"] = pd.qcut(rfm['recency'], 5, labels=[5, 4, 3, 2, 1])
rfm["frequency_score"] = pd.qcut(rfm['frequency'].rank(method="first"), 5, labels=[1, 2, 3, 4, 5])
rfm["monetary_score"] = pd.qcut(rfm['monetary'], 5, labels=[1, 2, 3, 4, 5])
rfm['monetaryscore'] = pd.qcut(rfm['monetary'].rank(method='first'),5,labels=[5,4,3,2,1])
rfm.head()
rfm["RFM_SCORE"] = (rfm['recency_score'].astype(str) +
                    rfm['frequency_score'].astype(str))

###############################################################
#  MAKE SEGMENTATION WITH RFM SCORES
###############################################################

# RFM SEGMENTATION MAP NAMES
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


#ANALYSIS AND EXPORT LOYAL CUSTOMER LIST 

rfm[["segment", "recency", "frequency", "monetary"]].groupby("segment").agg(["mean", "count"])
rfm[["recency", "frequency", "monetary", "segment"]].groupby("segment").agg(["mean","min","max","count"])

new_df = pd.DataFrame()
new_df["loyal_customers"] = rfm[rfm["segment"] == "loyal_customers"].index
new_df.head()


new_df.to_csv("loyal_customers.csv")


