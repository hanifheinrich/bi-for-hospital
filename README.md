# Penerapan Business Intelligence Berbasis Dashboard, Clustering, dan Forecasting Pada Data Pasien Rawat Inap di RSUD M.Natsir 

Ini adalah tugas akhir saya di Departemen Sistem Informasi Universitas Andalas. Penelitian ini ingin menunjukkan bahwa data rumah sakit bisa lebih dimanfaatkan dengan machine learing. Dengan adanya machine learning kita dapat mengetahui pola dan karakteristik pasien berdasarakan data. Selain itu kita juga bisa memanfaatkan data pasien menjadi dashboard sebagai upaya data-driven decision making.

## Table of Contents:

- Dataset
- Data Preprocessing
- Extract-Transform-Load
- Data Analysis
- Forecasting
- Clustering
- Alert Notification
- Dashboard
  
## Dataset
Sumber data yang digunakan adalah data rawat inap pasien di RSUD M. Natsir sebanyak 31197 records. Data ini terdiri dari 19 Bangsal dari tahun 2020 hingga bulan Agustus 2023. Data tersebut berasal dari bidang rekam medis RSUD M. Natsir yang terdiri dari empat file .xlsx yang dipisah berdasarkan tahun pencatatan. 

## Data Preprocessing
Data diagnosa banyak yang mengandung simbol seperti +, -, &, dan – untuk merangkap pasien dengan diagnosa yang banyak. Selain itu data pada fields diagnosa banyak yang mengandung keterangan seperti sedang, riwayat, dan memanjang yang merupakan keterangan dari diagnosa pasien. Sedangkan untuk fields Cara Keluar juga banyak yang mengandung keterangan seperti pindah ke ICU atau rujuk ke RSSN. Untuk itu diperlukan data cleaning untuk meningkatkan kualitas analisis.
```python
import pandas as pd
df = pd.read_csv("dataset.csv")
df['Cara Keluar'] = df['Cara Keluar'].str.replace(r'(Pindah Ruangan\s).*$', r'\1', regex=True)
df['Cara Keluar'] = df['Cara Keluar'].str.replace(r'(Rujuk\s).*$', r'\1', regex=True)
diagnosa_split = df['Diagnosa'].str.split(' \+ ', expand=True)
df = pd.concat([df, diagnosa_split], axis=1)
kolom_baru = ['Diagnosa' + str(i+1) for i in range(diagnosa_split.shape[1])]
df.columns = list(df.columns[:-diagnosa_split.shape[1]]) + kolom_baru
df.to_csv("preprocessing1.csv", index=False)
```
```python
import pandas as pd
df = pd.read_csv('preprocessing3.csv', skip_blank_lines=False)
kata_awal_hapus = pd.read_csv('breakword.csv')
for kata_awal in kata_awal_hapus:
    df['Diagnosa'] = df['Diagnosa'].str.replace(fr'({kata_awal}\s).*$', fr'\1', regex=True)
df.to_csv('preprocessing4.csv', index=False)
```
Hasil dari proses ini dapat mengoptimalkan 6144 jenis diagnosa menjadi 1754 jenis diagnosa
![image](https://github.com/user-attachments/assets/b7994479-e4db-4ac6-ab2f-fe12c2062f55)

## Extract-Transform-Load

ETL (Extract, Transform, Load) adalah proses yang terdiri dari tiga tahap mengekstraksi data dari berbagai sumber, mentransformasikan data ke dalam format yang sesuai untuk analisis, dan memuatnya ke dalam data warehouse. Proses ETL ini penting untuk memastikan kualitas data dalam data warehouse sehingga mendukung pengambilan keputusan bisnis yang lebih baik.
### ETL Tabel Asal Daerah
```python
import pandas as pd
import mysql.connector
df = pd.read_csv('dataetl.csv')
df = df[['asal_daerah']]
df_no_duplicates = df.drop_duplicates()
df_no_duplicates['id_daerah'] = range(1, len(df_no_duplicates) + 1)
print(df_no_duplicates)
try:
    connection = mysql.connector.connect(host='localhost',
database='db_mnatsir', user='root', password='')
    if connection.is_connected():
        cursor = connection.cursor()
        cursor.execute("USE db_mnatsir”)
        for index, row in df_no_duplicates.iterrows():
            cursor.execute("INSERT INTO dim_asal_daerah (id_daerah, asal_daerah) VALUES (%s, %s)", (row['id_daerah'], row['asal_daerah']))
            connection.commit()
        print("Data berhasil dimuat ke MySQL")     
except mysql.connector.Error as e:
    print("Error:", e)
finally:
    if 'connection' in locals():
        connection.close()
        print("Koneksi ke MySQL ditutup")
```
### ETL Tabel Fakta Rawatan
```python
import pandas as pd
import mysql.connector

data_etl = pd.read_csv("dataetl.csv", usecols=['mr', 'cara_keluar', 'cara_bayar', 'bangsal', 'diagnosa1', 'diagnosa2', 'diagnosa3', 'tanggal'])

try:
    connection = mysql.connector.connect(host='localhost',
database='db_mnatsir', user='root', password='')
    if connection.is_connected():
        dim_waktu = pd.read_sql("SELECT * FROM dim_waktu", connection)
        dim_bangsal = pd.read_sql("SELECT * FROM dim_bangsal", connection)
        dim_cara_bayar = pd.read_sql("SELECT * FROM dim_cara_bayar", connection)
        dim_cara_keluar = pd.read_sql("SELECT * FROM dim_cara_keluar", connection)
        dim_diagnosa = pd.read_sql("SELECT * FROM dim_diagnosa", connection)

data_etl = pd.merge(data_etl, dim_waktu, how="left", on="tanggal").drop(columns=['tanggal'])
        data_etl = pd.merge(data_etl, dim_bangsal, how="left", on="bangsal").drop(columns=['bangsal'])
        data_etl = pd.merge(data_etl, dim_cara_bayar, how="left", on="cara_bayar").drop(columns=['cara_bayar'])
        data_etl = pd.merge(data_etl, dim_cara_keluar, how="left", on="cara_keluar").drop(columns=['cara_keluar'])
        data_etl = pd.merge(data_etl, dim_diagnosa, how="left", left_on="diagnosa1", right_on="diagnosa").drop(columns=['diagnosa1', 'diagnosa']).rename(columns={'id_diagnosa': 'id_diag1'})
        data_etl = pd.merge(data_etl, dim_diagnosa, how="left", 
left_on="diagnosa2", right_on="diagnosa").drop(columns=['diagnosa2', 'diagnosa']).rename(columns={'id_diagnosa': 'id_diag2'})
        data_etl = pd.merge(data_etl, dim_diagnosa, how="left", left_on="diagnosa3", right_on="diagnosa").drop(columns=['diagnosa3', 'diagnosa']).rename(columns={'id_diagnosa': 'id_diag3'})
        data_etl['id_rawatan'] = data_etl.index + 1
        data_etl['tanggal'] = pd.to_datetime(data_etl['tanggal']).dt.strftime('%Y-%m-%d')
        print(data_etl)
        cursor = connection.cursor()
        for index, row in data_etl.iterrows():
            cursor.execute("INSERT INTO fakta_rawatan (id_rawatan,id_waktu,mr,id_bangsal,id_cb,id_ck,id_diag1,id_diag2,id_diag3) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)", 
                           (row['id_rawatan'], row['id_Waktu'], row['mr'], row['id_bangsal'], row['id_cb'], row['id_ck'], row['id_diag1'], row['id_diag2'], row['id_diag3']))
            connection.commit()
        print("Data berhasil dimuat ke MySQL ke tabel fakta_rawatan")
except mysql.connector.Error as e:
    print("Error:", e)
finally:
    if 'connection' in locals():
        connection.close()
        print("Koneksi ke MySQL ditutup")
```
## Data Mart Scheme
![image](https://github.com/user-attachments/assets/8a45e49f-a860-4bac-a100-1d6548a8626f)

## Data Analysis
Sebelum melakukan forecasting untuk jumlah rawatan, dilakukan analisis data untuk mengetahui  pola, tren, seasoan, dan lainnya dari data ini.
```python
import pandas as pd
import matplotlib.pyplot as plt
from statsmodels.tsa.seasonal import seasonal_decompose
file_path = 'analytic/fakta_rawatan.csv'
data = pd.read_csv(file_path)
data['tanggal'] = pd.to_datetime(data['tanggal'])
data.set_index('tanggal', inplace=True)
rawatan_per_bulan = data.resample('M').size()
decomposition = seasonal_decompose(rawatan_per_bulan, model='additive')
plt.figure(figsize=(12, 8))
plt.subplot(411)
plt.plot(rawatan_per_bulan, label='Original', color='blue')
plt.title('Original Time Series')
plt.legend(loc='upper left')
plt.subplot(412)
plt.plot(decomposition.trend, label='Trend', color='orange')
plt.title('Trend Component')
plt.legend(loc='upper left')
plt.subplot(413)
plt.plot(decomposition.seasonal, label='Seasonality', color='green')
plt.title('Seasonal Component')
plt.legend(loc='upper left')
plt.subplot(414)
plt.plot(decomposition.resid, label='Residuals', color='red')
plt.title('Residuals')
plt.legend(loc='upper left')
plt.tight_layout()
plt.show()
```
![image](https://github.com/user-attachments/assets/0442d736-d837-4fa6-99ee-921637ad2eab)

## Forecasting
Forecasting adalah proses memprediksi kejadian atau tren di masa depan berdasarkan data historis dan analisis statistik. Peramalan ini bertujuan untuk membantu pihak manajerial dalam memperkirakan apakah terjadi kenaikan jumlah rawatan untuk tahun berikutnya. Selain itu, peramalan ini juga dapat membantu dalam membuat kebijakan semisal terjadi lonjakan jumlah rawatan di RSUD M. Natsir dengan tetap mempertahankan kualitas pelayanan. 
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from statsmodels.tsa.arima.model import ARIMA
from statsmodels.tsa.statespace.sarimax import SARIMAX
from statsmodels.tsa.holtwinters import ExponentialSmoothing
from sklearn.metrics import mean_absolute_percentage_error

df = pd.read_csv('Analytic/fakta_rawatan.csv')
df['tanggal'] = pd.to_datetime(df['tanggal'])
df['bulan_tahun'] = df['tanggal'].dt.to_period('M')
data = df.groupby('bulan_tahun').size().reset_index(name='jumlah_rawatan')

train_size = int(len(data) * 0.8)
data_train = data[:train_size]
data_test = data[train_size:]

def weighted_moving_average(data, weights):
    wma = np.convolve(data, weights[::-1], mode='valid') / sum(weights)
    return wma
weights = np.arange(1, len(data_train) + 1)
wma_train = weighted_moving_average(data_train['jumlah_rawatan'], weights)
last_wma_value = wma_train[-1]
forecast_wma = np.full(len(data_test), last_wma_value)
mape_wma = mean_absolute_percentage_error(data_test['jumlah_rawatan'], forecast_wma)
model_arima = ARIMA(data_train['jumlah_rawatan'], order=(1, 2, 1))
model_fit_arima = model_arima.fit()
forecast_arima = model_fit_arima.forecast(len(data_test))
mape_arima = mean_absolute_percentage_error(data_test['jumlah_rawatan'], forecast_arima)
model_sarima = SARIMAX(data_train['jumlah_rawatan'], order=(1, 1, 1), seasonal_order=(1, 1, 1, 12))
model_fit_sarima = model_sarima.fit()
forecast_sarima = model_fit_sarima.forecast(steps=len(data_test))
mape_sarima = mean_absolute_percentage_error(data_test['jumlah_rawatan'], forecast_sarima)
model_tes_seasonal = ExponentialSmoothing(data_train['jumlah_rawatan'], trend='add', seasonal='add', seasonal_periods=7)
model_fit_tes_seasonal = model_tes_seasonal.fit()
forecast_tes_seasonal = model_fit_tes_seasonal.forecast(len(data_test))
mape_tes_seasonal = mean_absolute_percentage_error(data_test['jumlah_rawatan'], forecast_tes_seasonal)
plt.figure(figsize=(12, 6))
plt.plot(data_test['bulan_tahun'].astype(str), data_test['jumlah_rawatan'], label='Data Uji', color='black')
plt.plot(data_test['bulan_tahun'].astype(str), forecast_wma, label='WMA (MAPE: {:.2f}%)'.format(mape_wma * 100), linestyle='--')
plt.plot(data_test['bulan_tahun'].astype(str), forecast_arima, label='ARIMA (MAPE: {:.2f}%)'.format(mape_arima * 100), linestyle='--')
plt.plot(data_test['bulan_tahun'].astype(str), forecast_sarima, label='SARIMA (MAPE: {:.2f}%)'.format(mape_sarima * 100), linestyle='--')
plt.plot(data_test['bulan_tahun'].astype(str), forecast_tes_seasonal, label='TES (MAPE: {:.2f}%)'.format(mape_tes_seasonal * 100), linestyle='--')
plt.title('Perbandingan Model Peramalan')
plt.xlabel('Bulan')
plt.ylabel('Jumlah Rawatan')
plt.legend()
plt.grid(True)
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```
![image](https://github.com/user-attachments/assets/bcec2d52-5574-41f0-a7b4-6cbebc5e5c46)

### Insight
- Kasus: Pada akhir 2021 terjadi lonjakan rawatan di bangsal Interne. Untuk itu dibutuhkan perawat tambahan di bangsal tsb.
- Solusi: Dengan adanya forecasting, pola jumlah rawatan pasien bisa dibaca dan manajerial bisa mempersiapkan sumber daya dan penjadwalan perawatan.

## Clustering
Clustering adalah teknik analisis data yang mengelompokkan objek-objek yang memiliki karakteristik serupa ke dalam kelompok-kelompok yang berbeda untuk menemukan pola atau struktur dalam data.
```python
import pandas as pd
from sklearn.cluster import KMeans
import matplotlib.pyplot as plt
from matplotlib.ticker import MaxNLocator
import numpy as np
from sklearn.metrics import davies_bouldin_score

df = pd.read_csv('Analytic/dataset_cluster.csv')  
X = df[['id_bangsal', 'range_umur']]
kmeans = KMeans(n_clusters=3, random_state=42)
df['cluster'] = kmeans.fit_predict(X)
dbi_score = davies_bouldin_score(X, df['cluster'])

df.to_csv('cluster_umur_bangsal.csv', index=False)  
plt.figure(figsize=(10, 6))
plt.scatter(df['id_bangsal'], df['range_umur'], c=df['cluster'], cmap='viridis', marker='o')
plt.xlabel('Bangsal')
plt.ylabel('Range Umur')
plt.title('K-Means Clustering (DBI: {:.2f})'.format(dbi_score))
plt.colorbar(label='Cluster')

plt.gca().xaxis.set_major_locator(MaxNLocator(integer=True))
plt.gca().yaxis.set_major_locator(MaxNLocator(integer=True))

for cluster in range(3):
    cluster_points = df[df['cluster'] == cluster][['id_bangsal', 'range_umur']]
    center = cluster_points.mean()
    radius = np.max(np.sqrt(np.sum((cluster_points - center) ** 2, axis=1)))
    circle = plt.Circle((center['id_bangsal'], center['range_umur']), radius, color='black', fill=False, linestyle='--')
    plt.gca().add_artist(circle)

plt.grid(True)
plt.show()
```
![image](https://github.com/user-attachments/assets/9815d447-005b-40fd-8982-e085e3fa5417)
### Insight
-	Persiapan infrastruktur pegangan besi untuk cluster lansia, penyetokan kursi bayi pada cluster balita
-	Penyusunan ulang denah bangsal rumah sakit berdasarkan cluster.

## Alert Notification
![image](https://github.com/user-attachments/assets/e6257f61-64ae-4023-85bf-99ea006bfd00)

## Dashboard
![image](https://github.com/user-attachments/assets/140c989a-04b8-4393-b3c8-21762269d350)
![image](https://github.com/user-attachments/assets/8f9b575a-9846-481a-a852-db34a84fd214)
![image](https://github.com/user-attachments/assets/a8ac9e8b-de30-459b-8c1c-c8724edd7e45)










