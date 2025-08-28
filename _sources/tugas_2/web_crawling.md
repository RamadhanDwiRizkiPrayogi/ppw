# web crawling

Penjelasan program:
Program ini melakukan crawling data dari Springer Nature API menggunakan beberapa kata kunci, menyimpan hasil ke file CSV, dan menampilkan ringkasan dataset.

## Kode crawling
```python
import requests
import pandas as pd
import os

api_key = "0bf70d30730517ae78a523710a0f6316"
keywords = ["web mining", "web usage mining", "data mining", "information retrieval"]

if os.path.exists('hasil_crawling.csv'):
    print("File hasil_crawling.csv sudah ada. Menggunakan data yang tersimpan.")
    df = pd.read_csv('hasil_crawling.csv')
    print(f"Data berhasil dimuat: {len(df)} records")
else:
    results = []
    for keyword in keywords:
        try:
            url = "https://api.springernature.com/meta/v2/json"
            params = {
                "q": keyword,
                "api_key": api_key,
                "p": 10
            }
            response = requests.get(url, params=params, timeout=10)
            if response.status_code == 200:
                data = response.json()
                records = data.get('records', [])
                for record in records:
                    doi = record.get('doi', 'No DOI')
                    title = record.get('title', 'No Title')
                    abstract = record.get('abstract', 'No Abstract')
                    results.append({
                        'Keyword': keyword,
                        'DOI': doi,
                        'Title': title,
                        'Abstract': abstract
                    })
            else:
                print(f"Error for keyword '{keyword}':", response.status_code)
        except requests.exceptions.Timeout:
            print(f"Timeout untuk keyword '{keyword}' - melanjutkan ke keyword berikutnya")
        except Exception as e:
            print(f"Error untuk keyword '{keyword}': {str(e)}")
    df = pd.DataFrame(results)
    if not df.empty:
        df.to_csv('hasil_crawling.csv', index=False)
        print(f"Data telah disimpan ke hasil_crawling.csv: {len(df)} records")
    else:
        print("Tidak ada data yang berhasil di-crawl")
```

## Menampilkan hasil crawling
```python
import pandas as pd

file_path = 'hasil_crawling.csv'
df = pd.read_csv(file_path)

# print(df)

# Jika Anda ingin melihat beberapa baris pertama dari DataFrame
print(df.head())
```

Contoh output:
```
  Keyword                           DOI  \
0  web mining   10.1007/978-3-032-00983-8_5   
1  web mining   10.1007/978-3-031-93802-3_7   
2  web mining   10.1007/978-981-96-7238-7_2   
3  web mining  10.1007/978-3-031-95296-8_15   
4  web mining   10.1007/978-3-031-90470-7_6   

                                               Title  \
0  Survey on Data Mining and Machine Learning Met...   
1  Unveiling Power Laws in Graph Mining: Techniqu...   
2  Architecture Mining Approach for Systems-of-Sy...   
3  A Mathematical Model and Algorithm for Data An...   
4  ‘Internet of Things’ and ‘Social Networking’: ...   

                                            Abstract  
0  It is observed that the Mental illness by the ...  
1  Power laws play a crucial role in understandin...  
2  Context: Systems of Systems (SoS) constitute a...  
3  Machine learning methods play an important rol...  
4  Moving to the post-2000 period, or the post-fo...  
```

## Menampilkan informasi dataset
```python
print("=== INFORMASI DATASET ===")
print(f"Total records: {len(df)}")
print(f"Kolom: {list(df.columns)}")
print(f"\nDistribusi per keyword:")
print(df['Keyword'].value_counts())
print(f"\nShape dataset: {df.shape}")
```

Contoh output:
```
=== INFORMASI DATASET ===
Total records: 40
Kolom: ['Keyword', 'DOI', 'Title', 'Abstract']

Distribusi per keyword:
Keyword
web mining               10
web usage mining         10
data mining              10
information retrieval    10
Name: count, dtype: int64

Shape dataset: (40, 4)
```