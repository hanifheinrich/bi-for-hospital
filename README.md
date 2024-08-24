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
Sumber data yang digunakan adalah data rawat inap pasien di RSUD M. Natsir. Data ini terdiri dari 19 Bangsal dari tahun 2020 hingga bulan Agustus 2023. Data tersebut berasal dari bidang rekam medis RSUD M. Natsir yang terdiri dari empat file .xlsx yang dipisah berdasarkan tahun pencatatan. 

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

