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



