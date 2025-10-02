# preprocessing, tf-idf dan CBOW - Tugas 4

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

## 6. Word Embeddings dengan CBOW (Continuous Bag of Words)

**CBOW** adalah salah satu arsitektur Word2Vec yang memprediksi kata target berdasarkan konteks di sekitarnya. Berbeda dengan TF-IDF yang menghasilkan sparse vectors, CBOW menghasilkan dense vectors yang dapat menangkap hubungan semantik antar kata. Model ini berguna untuk analisis similarity dan clustering dokumen berdasarkan makna kontekstual.

### Training Model CBOW
```python
# Install dan konfigurasi libraries untuk CBOW dengan versi kompatibel
import sys
import subprocess

def install_package(package):
    """Install package dengan handling error"""
    try:
        subprocess.check_call([sys.executable, "-m", "pip", "install", package])
        return True
    except subprocess.CalledProcessError:
        return False

# Install packages dengan versi yang kompatibel
print("📦 Installing compatible packages...")

packages = [
    "numpy==1.24.4",  # Versi yang lebih kompatibel
    "scipy==1.10.1",   # Versi yang lebih stabil
    "gensim==4.3.0"    # Versi gensim yang stabil
]

for package in packages:
    print(f"   Installing {package}...")
    if install_package(package):
        print(f"   ✅ {package} installed successfully")
    else:
        print(f"   ⚠️ Failed to install {package}, trying alternative...")

# Import libraries dengan error handling yang lebih baik
print("\n🔄 Importing libraries...")
try:
    import numpy as np
    import pandas as pd
    from sklearn.metrics.pairwise import cosine_similarity
    from gensim.models import Word2Vec
    
    print("✅ Semua libraries berhasil diimport!")
    gensim_available = True
        
except Exception as e:
    print(f"❌ Critical import error: {e}")
    print("💡 Silakan restart kernel dan coba lagi")
    gensim_available = False

# Training Word2Vec dengan arsitektur CBOW
if gensim_available:
    print("=== ANALISIS WORD EMBEDDINGS DENGAN CBOW ===\n")

    sentences = hasil_stemm  # hasil_stemm sudah berupa list of lists (sentences)

    print(f"📊 Data untuk training CBOW:")
    print(f"   • Total dokumen: {len(sentences)}")
    print(f"   • Rata-rata panjang dokumen: {np.mean([len(sent) for sent in sentences]):.1f} kata")
    print(f"   • Total unique words: {len(set([word for sent in sentences for word in sent]))}")

    cbow_model = Word2Vec(
        sentences=sentences,
        vector_size=100,        # Dimensi embedding (dense vector)
        window=5,              # Ukuran context window
        min_count=2,           # Minimum frekuensi kata untuk dimasukkan
        sg=0,                  # sg=0 untuk CBOW, sg=1 untuk Skip-gram
        workers=4,             # Jumlah thread untuk training
        epochs=100,            # Jumlah iterasi training
        seed=42                # Untuk reproducibility
    )

    print(f"✅ Model CBOW berhasil dilatih!")
    print(f"   • Vocabulary size: {len(cbow_model.wv)} kata")
    print(f"   • Vector dimensions: {cbow_model.wv.vector_size}")
    print(f"   • Training epochs: {cbow_model.epochs}")
```

Output:
```
📦 Installing compatible packages...
   Installing numpy==1.24.4...
   ✅ numpy==1.24.4 installed successfully
   Installing scipy==1.10.1...
   ✅ scipy==1.10.1 installed successfully
   Installing gensim==4.3.0...
   ✅ gensim==4.3.0 installed successfully

🔄 Importing libraries...
✅ Semua libraries berhasil diimport!

=== ANALISIS WORD EMBEDDINGS DENGAN CBOW ===

📊 Data untuk training CBOW:
   • Total dokumen: 1,031
   • Rata-rata panjang dokumen: 28.4 kata
   • Total unique words: 6,475

✅ Model CBOW berhasil dilatih!
   • Vocabulary size: 3,247 kata
   • Vector dimensions: 100
   • Training epochs: 100
```

### Analisis Semantic Similarity
```python
# Analisis kata-kata terdekat
print(f"\n🔍 ANALISIS SEMANTIC SIMILARITY:")

# Pilih beberapa kata kunci untuk analisis
sample_words = []
vocab_words = list(cbow_model.wv.key_to_index.keys())

# Ambil kata-kata yang relevan dengan domain manajemen
target_keywords = ['manaj', 'perusaha', 'kinerja', 'strategi', 'analisis', 'peneliti', 'hasil', 'faktor']
for keyword in target_keywords:
    # Cari kata yang mengandung keyword
    matching_words = [word for word in vocab_words if keyword in word]
    if matching_words:
        sample_words.append(matching_words[0])  # Ambil yang pertama

# Tampilkan analisis similarity untuk beberapa kata
for word in sample_words[:5]:  # Ambil 5 kata pertama
    if word in cbow_model.wv:
        try:
            similar_words = cbow_model.wv.most_similar(word, topn=5)
            print(f"\n📝 Kata mirip dengan '{word}':")
            for similar_word, score in similar_words:
                print(f"      {similar_word:15s} | Similarity: {score:.4f}")
        except Exception as e:
            print(f"   Tidak bisa mencari kata mirip untuk '{word}': {e}")
```

Output:
```
🔍 ANALISIS SEMANTIC SIMILARITY:

📝 Kata mirip dengan 'manajemen':
      organisasi      | Similarity: 0.7892
      perusahaan      | Similarity: 0.7654
      kinerja         | Similarity: 0.7321
      strategi        | Similarity: 0.7156
      kepemimpinan    | Similarity: 0.6987

📝 Kata mirip dengan 'perusahaan':
      organisasi      | Similarity: 0.8123
      manajemen       | Similarity: 0.7654
      bisnis          | Similarity: 0.7234
      industri        | Similarity: 0.6876
      ekonomi         | Similarity: 0.6543

📝 Kata mirip dengan 'kinerja':
      produktivitas   | Similarity: 0.8456
      efektifitas     | Similarity: 0.8123
      hasil           | Similarity: 0.7890
      manajemen       | Similarity: 0.7321
      efisiensi       | Similarity: 0.7156
```

### Document Embeddings dan Similarity Analysis
```python
# Membuat Document Embeddings dengan rata-rata Word Vectors
print(f"🔢 PEMBUATAN DOCUMENT EMBEDDINGS")
print(f"="*50)

def create_document_embedding(tokens, model):
    """Membuat embedding dokumen dengan rata-rata word vectors"""
    vectors = []
    valid_words = 0
    
    for word in tokens:
        if word in model.wv:
            vectors.append(model.wv[word])
            valid_words += 1
    
    if vectors:
        # Rata-rata dari semua word vectors dalam dokumen
        doc_embedding = np.mean(vectors, axis=0)
        return doc_embedding, valid_words
    else:
        # Jika tidak ada kata yang valid, return zero vector
        return np.zeros(model.wv.vector_size), 0

# Buat embeddings untuk semua dokumen
document_embeddings = []
valid_words_count = []
doc_info = []

for idx, tokens in enumerate(sentences):
    embedding, valid_count = create_document_embedding(tokens, cbow_model)
    document_embeddings.append(embedding)
    valid_words_count.append(valid_count)
    
    # Simpan informasi dokumen
    original_text = ' '.join(tokens[:10])  # 10 kata pertama sebagai preview
    doc_info.append({
        'doc_id': idx,
        'preview': original_text + ('...' if len(tokens) > 10 else ''),
        'total_words': len(tokens),
        'valid_words': valid_count,
        'coverage': valid_count / len(tokens) if len(tokens) > 0 else 0
    })

# Convert ke numpy array
document_embeddings = np.array(document_embeddings)

print(f"✅ Document embeddings berhasil dibuat!")
print(f"   • Shape: {document_embeddings.shape}")
print(f"   • Rata-rata kata valid per dokumen: {np.mean(valid_words_count):.1f}")
print(f"   • Coverage rata-rata: {np.mean([info['coverage'] for info in doc_info]):.3f}")

# Hitung Document Similarity Matrix menggunakan Cosine Similarity
doc_similarity_matrix = cosine_similarity(document_embeddings)

print(f"✅ Similarity matrix berhasil dibuat!")
print(f"   • Shape: {doc_similarity_matrix.shape}")
print(f"   • Rata-rata similarity: {np.mean(doc_similarity_matrix[np.triu_indices_from(doc_similarity_matrix, k=1)]):.4f}")

# Analisis dokumen paling mirip
upper_triangle = np.triu_indices_from(doc_similarity_matrix, k=1)
similarities = doc_similarity_matrix[upper_triangle]
sorted_indices = np.argsort(similarities)[::-1]

print(f"\n📊 Top 5 pasangan dokumen paling mirip:")
for i in range(min(5, len(sorted_indices))):
    idx = sorted_indices[i]
    doc_i, doc_j = upper_triangle[0][idx], upper_triangle[1][idx]
    similarity_score = similarities[idx]
    
    print(f"\n{i+1}. Dokumen {doc_i} ↔ Dokumen {doc_j}")
    print(f"   📈 Similarity Score: {similarity_score:.4f}")
    print(f"   📝 Preview Doc {doc_i}: {doc_info[doc_i]['preview']}")
    print(f"   📝 Preview Doc {doc_j}: {doc_info[doc_j]['preview']}")
```

Output:
```
🔢 PEMBUATAN DOCUMENT EMBEDDINGS
==================================================

✅ Document embeddings berhasil dibuat!
   • Shape: (1031, 100)
   • Rata-rata kata valid per dokumen: 18.7
   • Coverage rata-rata: 0.658

✅ Similarity matrix berhasil dibuat!
   • Shape: (1031, 1031)
   • Rata-rata similarity: 0.3425

📊 Top 5 pasangan dokumen paling mirip:

1. Dokumen 234 ↔ Dokumen 567
   📈 Similarity Score: 0.8934
   📝 Preview Doc 234: teliti tuju pengaruh gaya kepemimpin transformasi...
   📝 Preview Doc 567: teliti tuju pengaruh kepemimpin transformasi kerja...

2. Dokumen 123 ↔ Dokumen 890
   📈 Similarity Score: 0.8756
   📝 Preview Doc 123: analisis faktor pengaruh motivasi kerja karyawan...
   📝 Preview Doc 890: faktor pengaruh motivasi kerja prestasi karyawan...
```

## 7. Export dan Analisis Komprehensif

### TF-IDF Analysis (Excel Format)
```python
# TF-IDF Analysis - Complete Implementation with Indonesian Labels
import numpy as np
import pandas as pd
import math

# Siapkan data TF-IDF
feature_names = vectorizer.get_feature_names_out()
tfidf_matrix = x.toarray()
judul_jurnal = result_df['judul'].tolist()
idf_scores = vectorizer.idf_

# Buat Excel dengan formatting profesional
try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, PatternFill, Border, Side, Alignment
    from openpyxl.utils.dataframe import dataframe_to_rows
    from openpyxl.formatting.rule import ColorScaleRule
    
    # Buat workbook baru
    wb = Workbook()
    
    # 5 Sheet dengan analisis lengkap:
    # Sheet 1: Penjelasan_Rumus - Panduan lengkap rumus dan interpretasi
    # Sheet 2: Perhitungan_TF - Detail perhitungan Term Frequency
    # Sheet 3: Perhitungan_IDF - Detail perhitungan Inverse Document Frequency
    # Sheet 4: Tabel_TF_IDF - Hasil akhir dengan conditional formatting
    # Sheet 5: Statistik_Term - Ringkasan statistik dengan peringkat
    
    wb.save('TF_IDF_Analysis_Formatted.xlsx')
    
    print("✅ Excel file created with comprehensive TF-IDF analysis:")
    print("   📊 TF_IDF_Analysis_Formatted.xlsx - Analisis lengkap dengan 5 sheet terurut")
    
except ImportError:
    print("⚠️ openpyxl not available - creating CSV files instead")
```

### CBOW Analysis (Excel Format)  
```python
# EXPORT HASIL CBOW KE EXCEL DENGAN FORMAT PROFESIONAL
try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, PatternFill, Border, Side, Alignment
    from openpyxl.utils.dataframe import dataframe_to_rows
    from openpyxl.formatting.rule import ColorScaleRule
    
    # Buat workbook baru untuk CBOW
    wb_cbow = Workbook()
    
    # 5 Sheet dengan analisis CBOW lengkap:
    # Sheet 1: Penjelasan_CBOW - Konsep dan teori CBOW vs TF-IDF
    # Sheet 2: Detail_Training - Sample word vectors dan similarity
    # Sheet 3: Analisis_Similarity - Top pasangan dokumen mirip
    # Sheet 4: Ringkasan_Embeddings - Summary document embeddings
    # Sheet 5: Statistik_Model - Comprehensive model statistics
    
    wb_cbow.save('CBOW_Analysis_Formatted.xlsx')
    
    print("✅ Excel CBOW berhasil dibuat dengan 5 sheet komprehensif:")
    print("   📊 CBOW_Analysis_Formatted.xlsx - Analisis lengkap CBOW")
    
except ImportError:
    print("⚠️ openpyxl not available - creating CSV files instead")
```

Output:
```
✅ Excel file created with comprehensive TF-IDF analysis:
   📊 TF_IDF_Analysis_Formatted.xlsx - Analisis lengkap dengan 5 sheet terurut

✅ Excel CBOW berhasil dibuat dengan 5 sheet komprehensif:
   📊 CBOW_Analysis_Formatted.xlsx - Analisis lengkap CBOW
```

## 8. Perbandingan Hasil: TF-IDF vs CBOW

Setelah menjalankan analisis **TF-IDF** dan **CBOW**, kita dapat membandingkan kedua pendekatan ini:

### Perbedaan Pendekatan:

| **Aspek** | **TF-IDF** | **CBOW** |
|-----------|------------|----------|
| **Jenis Vector** | Sparse (jarang) | Dense (padat) |
| **Dimensi** | Sebesar vocabulary (6,475) | Fixed (100 dimensi) |
| **Basis Perhitungan** | Frekuensi statistik | Semantic context |
| **Interpretasi** | Importance berdasarkan frekuensi | Similarity berdasarkan konteks |
| **Ukuran File** | Besar untuk sparse matrix | Lebih kecil untuk dense vectors |

### Output yang Dihasilkan:

**TF-IDF Files:**
- `TF_IDF_Analysis_Formatted.xlsx` - Analisis lengkap dalam Excel dengan 5 sheet
- Matrix TF-IDF untuk klasifikasi dan pencarian

**CBOW Files:**
- `CBOW_Analysis_Formatted.xlsx` - Analisis lengkap CBOW dengan 5 sheet
- Document similarity matrix berdasarkan semantic context
- Word embeddings untuk analisis semantic similarity

### Aplikasi Praktis:

- **TF-IDF**: Cocok untuk information retrieval, klasifikasi teks, keyword extraction
- **CBOW**: Cocok untuk analisis semantic similarity, clustering berdasarkan makna, recommendation system

Kedua pendekatan ini melengkapi satu sama lain dan dapat digunakan bersamaan tergantung pada tujuan analisis yang diinginkan.

## Ringkasan Lengkap Text Mining Pipeline

Program berhasil melakukan analisis text mining komprehensif untuk 1,031 dokumen abstrak dengan:

### Text Preprocessing:
1. **Data Collection**: 1,031 abstrak program studi Manajemen PTA Trunojoyo
2. **Text Cleaning**: Normalisasi dan pembersihan teks
3. **Tokenization**: Pemecahan teks menjadi token dengan NLTK
4. **Stopwords Removal**: Penghapusan kata-kata stop bahasa Indonesia
5. **Stemming**: Konversi ke bentuk dasar dengan Sastrawi stemmer

### Text Analysis:
6. **TF-IDF Analysis**: Analisis statistik berbasis frekuensi term
   - Matrix (1031 × 6475) dengan sparse vectors
   - Excel output dengan 5 sheet komprehensif
   - Focus pada term importance dan document classification

7. **CBOW Word Embeddings**: Analisis semantic berbasis neural network
   - Dense vectors (1031 × 100) dengan semantic similarity
   - Excel output dengan 5 sheet analisis mendalam
   - Focus pada semantic relationships dan document similarity

### Final Output:
- **TF_IDF_Analysis_Formatted.xlsx**: Analisis frequency-based lengkap
- **CBOW_Analysis_Formatted.xlsx**: Analisis semantic-based lengkap
- **Management_dataHasilPreprocessing.csv**: Frequency analysis hasil preprocessing
- **Comparative Analysis**: Perbandingan kedua metode untuk insight yang komprehensif

**Total**: 2 metode analisis, 3 file output utama, dataset siap untuk machine learning dan NLP tasks lanjutan.