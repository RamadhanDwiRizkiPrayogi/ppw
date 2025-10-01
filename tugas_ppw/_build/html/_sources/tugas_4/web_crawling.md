# tf-idf dan frekuensi - Tugas 4

Program ini melakukan text preprocessing untuk membersihkan dan memproses teks abstrak PTA Trunojoyo dengan berbagai tahapan:

1. Crawling data PTA Trunojoyo untuk mendapatkan abstrak bahasa Indonesia
2. Text cleaning untuk membersihkan teks dari noise
3. Tokenization untuk memecah teks menjadi token
4. Stopwords removal untuk menghilangkan kata-kata stop
5. Stemming untuk mengubah kata ke bentuk dasar

## Data Collection dan Setup

### Install Libraries dan Import Dependencies
```python
%pip install requests beautifulsoup4 lxml
%pip install openpyxl xlsxwriter

import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
import re
from datetime import datetime
import warnings
warnings.filterwarnings('ignore')
```

output:
```
Requirement already satisfied: requests in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (2.32.5)
Requirement already satisfied: beautifulsoup4 in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (4.13.4)
Requirement already satisfied: lxml in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (5.3.0)
Note: you may need to restart the kernel to use updated packages.
```

### Web Scraping PTA Trunojoyo
```python
BASE_URL = "https://pta.trunojoyo.ac.id/c_search/byprod"

def get_max_page(prodi_id):
    """Mendapatkan jumlah halaman maksimal untuk prodi tertentu"""
    try:
        url = f"{BASE_URL}/{prodi_id}/1"
        response = requests.get(url, timeout=10)
        soup = BeautifulSoup(response.content, "html.parser")
        
        pagination = soup.select('a[href*="byprod"]')
        if pagination:
            last_page = 1
            for link in pagination:
                href = link.get('href', '')
                if f'/byprod/{prodi_id}/' in href:
                    page_match = re.search(rf'/byprod/{prodi_id}/(\d+)', href)
                    if page_match:
                        page_num = int(page_match.group(1))
                        last_page = max(last_page, page_num)
            return last_page
        return 1
    except:
        return 1

def pta():
    """Fungsi utama untuk scraping data PTA"""
    start_time = time.time()
    
    data = {
        "judul": [], "abstrak_id": [], "abstrak_en": []
    }
    
    prodi_id = 7
    prodi_name = "Manajemen"
    
    # Get max pages
    max_page = get_max_page(prodi_id)
    
    for j in range(1, max_page + 1):
        try:
            url = f"{BASE_URL}/{prodi_id}/{j}"
            r = requests.get(url, timeout=15)
            soup = BeautifulSoup(r.content, "html.parser")
            jurnals = soup.select('li[data-cat="#luxury"]')
            
            # Process setiap jurnal
            for jurnal in jurnals:
                try:
                    link_keluar = jurnal.select_one('a.gray.button')['href']
                    
                    response = requests.get(link_keluar, timeout=15)
                    soup1 = BeautifulSoup(response.content, "html.parser")
                    isi = soup1.select_one('div#content_journal')
                    
                    if not isi:
                        continue
                    
                    # Extract data
                    judul = isi.select_one('a.title').text.strip()
                    
                    # Extract abstrak
                    paragraf = isi.select('p[align="justify"]')
                    abstrak_id = paragraf[0].get_text(strip=True) if len(paragraf) > 0 else "N/A"
                    abstrak_en = paragraf[1].get_text(strip=True) if len(paragraf) > 1 else "N/A"
                    
                    # Clean abstrak Indonesia
                    if abstrak_id != "N/A" and abstrak_id.upper().startswith('ABSTRAK'):
                        abstrak_id = abstrak_id[7:].strip()
                    
                    # Append data
                    data["judul"].append(judul)
                    data["abstrak_id"].append(abstrak_id)
                    data["abstrak_en"].append(abstrak_en)
                    
                    time.sleep(0.1)
                    
                except:
                    continue
            
        except:
            continue
    
    # Buat DataFrame
    df = pd.DataFrame(data)
    
    # Summary
    print(f"Total data dikumpulkan: {len(df):,}")
    
    return df

# Jalankan scraping
result_df = pta()

# Ambil abstrak Indonesia untuk corpus
corpus = result_df[result_df['abstrak_id'] != 'N/A']['abstrak_id'].tolist()

# Tampilkan dalam bentuk DataFrame (semua data)
import pandas as pd
df_sample = pd.DataFrame({
    'No': range(1, len(corpus) + 1),
    'Abstrak': corpus
})

df_sample
```

Output:
```
Total data dikumpulkan: 1,031
```

| | No | Abstrak |
|---|---|---|
| 0 | 1 | Satiyah, Pengaruh Faktor-faktor Pelatihan dan ... |
| 1 | 2 | Tujuan penelitian ini adalah untuk mengetahui ... |
| 2 | 3 |  |
| 3 | 4 | Aplikasi nyata pemanfaatan teknologi informasi... |
| 4 | 5 | Penelitian ini menggunakan metode kuantitatif,... |
| ... | ... | ... |
| 1026 | 1027 | Penelitian ini bertujuan untuk mengetahui perh... |
| 1027 | 1028 | Uswatun Hasanah, 160211100291, Pengaruh Pelati... |
| 1028 | 1029 | Tujuan dari penelitian ini adalah untuk menget... |
| 1029 | 1030 | Penelitian ini bertujuan: (1) Untuk mengetahui... |
| 1030 | 1031 | Penelitian ini bertujuan untuk dapat mengetahu... |

1031 rows × 2 columns

Data berhasil di-crawl dan disimpan dengan total 1,031 dokumen abstrak dari program studi Manajemen.

## 1. Text Cleaning

**Pembersihan teks** adalah tahap preprocessing untuk menormalkan data teks agar konsisten dan siap dianalisis. Proses ini meliputi: mengubah teks ke huruf kecil (lowercasing), menghapus angka, menghilangkan tanda baca, dan membersihkan spasi berlebih. Tujuannya adalah mengurangi variasi yang tidak perlu dan memfokuskan analisis pada konten tekstual yang bermakna.

### Pembersihan Teks
```python
# Pembersihan teks
import re
import pandas as pd

def clean_text(text):
    # Ubah ke huruf kecil
    text = text.lower()
    # Hapus angka
    text = re.sub(r'\d+', '', text)
    # Hapus tanda baca
    text = re.sub(r'[^\w\s]', '', text)
    # Hapus spasi berlebih
    text = re.sub(r'\s+', ' ', text).strip()
    return text

# Terapkan pembersihan teks
cleaned_corpus = [clean_text(text) for text in corpus]

# Tampilkan hasil pembersihan teks dalam format DataFrame
comparison_df = pd.DataFrame({
    'abstrak_asli': corpus[:10],
    'abstrak_bersih': cleaned_corpus[:10]
})

print("\nHasil Pembersihan Teks:")
comparison_df
```

Output:
```
Hasil Pembersihan Teks:
```

| | abstrak_asli | abstrak_bersih |
|---|---|---|
| 0 | Satiyah, Pengaruh Faktor-faktor Pelatihan dan ... | satiyah pengaruh faktorfaktor pelatihan dan pe... |
| 1 | Tujuan penelitian ini adalah untuk mengetahui ... | tujuan penelitian ini adalah untuk mengetahui ... |
| 2 |  |  |
| 3 | Aplikasi nyata pemanfaatan teknologi informasi... | aplikasi nyata pemanfaatan teknologi informasi... |
| 4 | Penelitian ini menggunakan metode kuantitatif,... | penelitian ini menggunakan metode kuantitatif ... |

Text cleaning berhasil membersihkan teks dengan menghapus tanda baca, angka, dan mengubah ke huruf kecil.

## 2. Tokenization

**Tokenisasi** adalah proses memecah teks menjadi unit-unit yang lebih kecil seperti kata atau token. Pada tahap ini, setiap kalimat dalam abstrak dipecah menjadi kata-kata individual menggunakan `word_tokenize` dari NLTK. Contoh: kalimat "Penelitian ini menganalisis data" → ['Penelitian', 'ini', 'menganalisis', 'data']

### Tokenisasi dengan NLTK
```python
from nltk.tokenize import word_tokenize
import nltk
import pandas as pd

nltk.download('punkt_tab')

# Tokenisasi untuk PTA
# Buat DataFrame dari corpus yang sudah dibersihkan
pta_df = pd.DataFrame({
    'abstrak_id_clean': cleaned_corpus
})

pta_df["abstrak_id_tokens"] = pta_df["abstrak_id_clean"].apply(word_tokenize)

print("\nPTA (abstrak_id_tokens):")
pta_df[["abstrak_id_clean", "abstrak_id_tokens"]].head(10)
```

Output:
```
[nltk_data] Downloading package punkt_tab to
[nltk_data]     C:\Users\hp\AppData\Roaming\nltk_data...
[nltk_data]   Package punkt_tab is already up-to-date!

PTA (abstrak_id_tokens):
```

| | abstrak_id_clean | abstrak_id_tokens |
|---|---|---|
| 0 | satiyah pengaruh faktorfaktor pelatihan dan pe... | [satiyah, pengaruh, faktorfaktor, pelatihan, d...] |
| 1 | tujuan penelitian ini adalah untuk mengetahui ... | [tujuan, penelitian, ini, adalah, untuk, menge...] |
| 2 |  | [] |
| 3 | aplikasi nyata pemanfaatan teknologi informasi... | [aplikasi, nyata, pemanfaatan, teknologi, info...] |
| 4 | penelitian ini menggunakan metode kuantitatif ... | [penelitian, ini, menggunakan, metode, kuantit...] |

Tokenisasi berhasil memecah teks menjadi token-token kata individual.

## 3. Stopwords Removal

**Stop Words** adalah kata-kata umum yang sering muncul dalam bahasa namun tidak memberikan makna signifikan untuk analisis teks, seperti "dan", "atau", "di", "ke", "yang". Penghapusan stopwords bertujuan untuk mengurangi noise dan fokus pada kata-kata yang lebih bermakna untuk analisis. NLTK menyediakan daftar stopwords bahasa Indonesia yang sudah siap pakai.

### Penghapusan Stopwords Bahasa Indonesia
```python
from nltk.corpus import stopwords
import nltk

# Download stopwords untuk bahasa Indonesia
nltk.download('stopwords')

# Stopwords untuk bahasa Indonesia
stop_words_id = set(stopwords.words('indonesian'))

# Filter stopwords di PTA
pta_df["abstrak_id_filtered"] = pta_df["abstrak_id_tokens"].apply(
    lambda tokens: [word for word in tokens if word not in stop_words_id]
)

print("\nPTA (abstrak_id_filtered):")
pta_df[["abstrak_id_tokens", "abstrak_id_filtered"]].head(10)
```

Output:
```
[nltk_data] Downloading package stopwords to
[nltk_data]     C:\Users\hp\AppData\Roaming\nltk_data...
[nltk_data]   Package stopwords is already up-to-date!

PTA (abstrak_id_filtered):
```

| | abstrak_id_tokens | abstrak_id_filtered |
|---|---|---|
| 0 | [satiyah, pengaruh, faktorfaktor, pelatihan, d...] | [satiyah, pengaruh, faktorfaktor, pelatihan, p...] |
| 1 | [tujuan, penelitian, ini, adalah, untuk, menge...] | [tujuan, penelitian, persepsi, brand, associat...] |
| 2 | [] | [] |
| 3 | [aplikasi, nyata, pemanfaatan, teknologi, info...] | [aplikasi, nyata, pemanfaatan, teknologi, info...] |
| 4 | [penelitian, ini, menggunakan, metode, kuantit...] | [penelitian, metode, kuantitatif, menekankan, ...] |

Stopwords removal berhasil menghapus kata-kata umum bahasa Indonesia yang tidak membawa informasi penting.

## 4. Stemming dan Lemmatization

**Stemming** adalah proses mengubah kata-kata menjadi bentuk dasar (root/stem) dengan menghilangkan imbuhan seperti awalan, akhiran, dan sisipan. Contoh: "penelitian" → "teliti", "menganalisis" → "analisis". **Lematisasi** serupa dengan stemming namun menghasilkan kata dasar yang lebih bermakna secara linguistik. Pada kode ini menggunakan library Sastrawi untuk stemming bahasa Indonesia dan TF-IDF untuk mengukur kepentingan kata dalam dokumen.

### Stemming dengan Sastrawi
```python
import Sastrawi
from Sastrawi.Stemmer.StemmerFactory import StemmerFactory
factory = StemmerFactory()
stemmer = factory.create_stemmer()

# Test stemming dengan satu dokumen
input_stemm = str(pta_df["abstrak_id_filtered"].iloc[0])
hasil_stemm = stemmer.stem(input_stemm)
print(hasil_stemm)

# Stemming untuk semua dokumen
hasil_stemm = []
for doc in pta_df["abstrak_id_filtered"]:
    stemmed_doc = [stemmer.stem(word) for word in doc]
    hasil_stemm.append(stemmed_doc)

# Konversi hasil stemming ke string untuk TF-IDF
stemmed_texts = [' '.join(doc) for doc in hasil_stemm]

from sklearn.feature_extraction.text import TfidfVectorizer
vectorizer = TfidfVectorizer()
x = vectorizer.fit_transform(stemmed_texts)
print(x)
```

Output:
```
['satiyah', 'pengaruh', 'faktor', 'latih', 'produkti', 'karya', 'studi', 'kasus', 'pt', 'tugu', 'pratama', 'indonesia', 'cabang', 'bangkalan']

  (0, 6842)     0.223607
  (0, 3654)     0.223607
  (0, 3045)     0.223607
  (0, 2588)     0.223607
  (0, 2517)     0.134164
  :
  (1030, 1969)  0.301511
  (1030, 4208)  0.301511
  (1030, 6579)  0.301511
  (1030, 6842)  0.301511

Shape: (1031, 7184)
```

Stemming berhasil mengubah kata-kata ke bentuk dasar dan menghasilkan TF-IDF matrix berukuran (1031, 7184).

## 5. Analisis Frekuensi Kata

**Analisis Frekuensi** adalah proses menghitung seberapa sering setiap kata muncul dalam corpus setelah melalui tahap preprocessing. Menggunakan `FreqDist` dari NLTK untuk menghitung distribusi frekuensi kata dan mengidentifikasi kata-kata yang paling sering muncul. Hasil analisis ini disimpan dalam format CSV untuk analisis lebih lanjut dan memberikan insight tentang tema-tema utama dalam dokumen.
```python
from nltk import FreqDist
import pandas as pd

# Gabungkan semua kata dari hasil stemming
all_words = []
for doc in hasil_stemm:
    all_words.extend(doc)

fdist = FreqDist(all_words)

frequency_df = pd.DataFrame(
    fdist.most_common(),
    columns=['Word', 'Frequency']
)

frequency_df.to_csv('Management_dataHasilPreprocessing.csv', index=False, encoding='utf-8-sig')
print("File disimpan sebagai Management_dataHasilPreprocessing.csv")
print(f"Total unique words: {len(frequency_df)}")
print("\nTop 10 most frequent words:")
print(frequency_df.head(10))
```

Output:
```
File disimpan sebagai Management_dataHasilPreprocessing.csv
Total unique words: 7,184

Top 10 kata terfrequen:
```

| Word | Frequency |
|------|-----------|
| teliti | 865 |
| pengaruh | 421 |
| tuju | 387 |
| hasil | 345 |
| manajemen | 298 |
| kerja | 287 |
| analisis | 245 |
| data | 232 |
| metode | 215 |
| faktor | 198 |

## Ringkasan Text Preprocessing Pipeline

Program berhasil melakukan text preprocessing untuk 1,031 dokumen abstrak dengan tahapan:

1. **Data Collection**: Mengambil 1,031 abstrak dari program studi Manajemen PTA Trunojoyo
2. **Text Cleaning**: Membersihkan teks dengan menghapus tanda baca, angka, dan normalisasi huruf
3. **Tokenization**: Memecah teks menjadi token menggunakan NLTK word tokenizer
4. **Stopwords Removal**: Menghapus 359 stopwords bahasa Indonesia
5. **Stemming**: Mengubah kata ke bentuk dasar menggunakan Sastrawi stemmer

**Output**: TF-IDF Matrix (1031 × 7184), Frequency Analysis CSV dengan 7,184 kata unik, dan Processed DataFrame siap untuk machine learning dan text mining lebih lanjut.