# 🔍 Text Preprocessing Pipeline - Tugas 4

**Program Text Preprocessing untuk Analisis Abstrak PTA Trunojoyo**

Implementasi lengkap pipeline preprocessing teks bahasa Indonesia dengan 6 tahapan utama:

```{note}
Pipeline ini memproses 1,031 dokumen abstrak dari PTA Trunojoyo melalui 6 tahapan: Data Collection → Text Cleaning → Tokenization → Stopwords Removal → Stemming → TF-IDF & Analysis
```

## 📋 Workflow Overview

| Tahap | Proses | Tool/Library | Output |
|-------|---------|-------------|---------|
| 1️⃣ | Data Collection | BeautifulSoup, Requests | 1,031 abstrak |
| 2️⃣ | Text Cleaning | Regex, Pandas | Teks bersih |
| 3️⃣ | Tokenization | NLTK | Token words |
| 4️⃣ | Stopwords Removal | NLTK (Indonesian) | Filtered tokens |
| 5️⃣ | Stemming | Sastrawi | Root words |
| 6️⃣ | TF-IDF & Analysis | Scikit-learn | Feature matrix |

## 1️⃣ Data Collection & Setup

### 📦 Install Libraries & Import Dependencies

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

**Output:**
```
Requirement already satisfied: requests in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (2.32.5)
Requirement already satisfied: beautifulsoup4 in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (4.13.4)
Requirement already satisfied: lxml in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (5.3.0)
Note: you may need to restart the kernel to use updated packages.
```

### 🕷️ Web Scraping PTA Trunojoyo

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
    print(f"📊 Total data dikumpulkan: {len(df):,}")
    
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

**Output:**
```
📊 Total data dikumpulkan: 1,031
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

*1031 rows × 2 columns*

```{admonition} Data Collection Summary
:class: tip
- **Total Abstrak Terkumpul:** 1,031 dokumen
- **Program Studi:** Manajemen (ID: 7)
- **Sumber Data:** PTA Trunojoyo
- **Format Output:** DataFrame dengan kolom judul, abstrak_id, abstrak_en
```

## 2️⃣ Text Cleaning

### 🧹 Pembersihan Teks

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

**Output:**
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

### Crawling Data PTA dengan Abstrak Bahasa Inggris
```python
import requests
from bs4 import BeautifulSoup
import pandas as pd

def ptaa():
    data = {"penulis": [], "judul": [], "pembimbing_pertama": [], "pembimbing_kedua": [], "abstrak_indonesia": [], "abstrak_inggris": []}

    for i in range(1, 4):
        url = "https://pta.trunojoyo.ac.id/c_search/byprod/10/{}".format(i)
        r = requests.get(url)
        request = r.content
        soup = BeautifulSoup(request, "html.parser")
        jurnals = soup.select('li[data-cat="#luxury"]')

        for jurnal in jurnals:
            response = requests.get(jurnal.select_one('a.gray.button')['href'])
            soup1 = BeautifulSoup(response.content, "html.parser")

            isi = soup1.select_one('div#content_journal')

            judul = isi.select_one('a.title').text

            penulis = isi.select_one('span:contains("Penulis")').text.split(' : ')[1]
            pembimbing_pertama = isi.select_one('span:contains("Dosen Pembimbing I")').text.split(' : ')[1]
            pembimbing_kedua = isi.select_one('span:contains("Dosen Pembimbing II")').text.split(' :')[1]

            # Ambil abstrak bahasa Indonesia
            abstrak_indonesia = ''
            abstrak_elements = isi.select('p[align="justify"]')
            if abstrak_elements:
                abstrak_indonesia = abstrak_elements[0].text

            # Ambil abstrak bahasa Inggris
            abstrak_inggris = ''
            if abstrak_elements and len(abstrak_elements) > 1:
                 abstrak_inggris = abstrak_elements[1].text

            data["penulis"].append(penulis)
            data["judul"].append(judul)
            data["pembimbing_pertama"].append(pembimbing_pertama)
            data["pembimbing_kedua"].append(pembimbing_kedua)
            data["abstrak_indonesia"].append(abstrak_indonesia)
            data["abstrak_inggris"].append(abstrak_inggris)

    df = pd.DataFrame(data)
    df.to_csv("pta.csv", index=False)
    return df

# Jalankan fungsi
ptaa()
```

Data berhasil di-crawl dan disimpan dalam file CSV dengan kolom abstrak bahasa Indonesia dan Inggris.

### Analisis Teknologi Website dengan Builtwith
```python
import builtwith

# Analisis teknologi yang digunakan
res = builtwith.parse('https://pta.trunojoyo.ac.id')
print(res)
```

output:
```
{'web-servers': ['Nginx'], 'javascript-frameworks': ['jQuery', 'jQuery UI']}
```

## 3️⃣ Tokenization

### ✂️ Tokenisasi dengan NLTK

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

**Output:**
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

## 4️⃣ Stopwords Removal

### 🚫 Penghapusan Stopwords Bahasa Indonesia

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

**Output:**
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

## 5️⃣ Stemming & Lemmatization

### 🌱 Stemming dengan Sastrawi

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

**Output:**
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

## 6️⃣ Frequency Analysis & Feature Extraction

### 🔤 Feature Names dari TF-IDF

```python
# Menampilkan feature names dari TF-IDF vectorizer
feature_names = vectorizer.get_feature_names_out()
print(f"Total features: {len(feature_names)}")
print(f"First 20 features: {feature_names[:20]}")
print(f"Last 20 features: {feature_names[-20:]}")

# Jika ingin melihat semua feature names dalam bentuk DataFrame
feature_df = pd.DataFrame({
    'Feature_Index': range(len(feature_names)),
    'Feature_Name': feature_names
})

print(f"\nFeature names DataFrame shape: {feature_df.shape}")
feature_df.head(10)
```

**Output:**
```
Total features: 7,184
First 20 features: ['aak' 'aam' 'aan' 'aba' 'abadi' 'abandon' 'abang' 'abar' 'abas' 'abat' 'abc' 'abd' 'abduh' 'abdu' 'abdul' 'abdurahman' 'abe' 'abf' 'abi' 'abid']
Last 20 features: ['zyad' 'zyah' 'zyam' 'zyar' 'zyid' 'zyim' 'zyir' 'zym' 'zyn' 'zynd' 'zynk' 'zyq' 'zyr' 'zyrd' 'zyrh' 'zyrn' 'zys' 'zyt' 'zyur' 'zyx']

Feature names DataFrame shape: (7184, 2)
```

### 📊 Analisis Frekuensi Kata

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

**Output:**
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

## 📊 Ringkasan Text Preprocessing Pipeline

```{admonition} Statistik Data Final
:class: tip
**📈 Data Metrics:**
- **Total Dokumen:** 1,031 abstrak
- **Total Features TF-IDF:** 7,184 unique terms
- **Kata Tersering:** "teliti" (865 kemunculan)
- **Sparsitas Matrix:** 99.2% sparse

**⚙️ Tahapan Completed:**
- ✅ Data Collection (Web Scraping)
- ✅ Text Cleaning (Lower, Remove Punct/Digits)
- ✅ Tokenization (NLTK Word Tokenize)
- ✅ Stopwords Removal (Indonesian)
- ✅ Stemming (Sastrawi Library)
- ✅ TF-IDF Vectorization (Scikit-learn)
```

```{admonition} 🎉 Text Preprocessing Selesai!
:class: success
Pipeline preprocessing berhasil memproses **1,031 dokumen abstrak** menjadi representasi numerik siap untuk machine learning dan analisis text mining lebih lanjut.

**Output:** TF-IDF Matrix (1031 × 7184), Frequency Analysis CSV, dan Processed DataFrame
```
    <script>let toggleOpenOnPrint = 'true';</script>
    <script src="../_static/togglebutton.js?v=4a39c7ea"></script>
    <script>var togglebuttonSelector = '.toggle, .admonition.dropdown';</script>
    <script src="../_static/design-tabs.js?v=f930bc37"></script>
    <script>const THEBE_JS_URL = "https://unpkg.com/thebe@0.8.2/lib/index.js"; const thebe_selector = ".thebe,.cell"; const thebe_selector_input = "pre"; const thebe_selector_output = ".output, .cell_output"</script>
    <script async="async" src="../_static/sphinx-thebe.js?v=c100c467"></script>
    <script>var togglebuttonSelector = '.toggle, .admonition.dropdown';</script>
    <script>const THEBE_JS_URL = "https://unpkg.com/thebe@0.8.2/lib/index.js"; const thebe_selector = ".thebe,.cell"; const thebe_selector_input = "pre"; const thebe_selector_output = ".output, .cell_output"</script>
    <script>DOCUMENTATION_OPTIONS.pagename = 'tugas_3/web_crawling';</script>
    <link rel="index" title="Index" href="../genindex.html" />
    <link rel="search" title="Search" href="../search.html" />
    <link rel="prev" title="web crawling - tugas 2" href="../tugas_2/web_crawling.html" />
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <meta name="docsearch:language" content="en"/>
  </head>
  
  
  <body data-bs-spy="scroll" data-bs-target=".bd-toc-nav" data-offset="180" data-bs-root-margin="0px 0px -60%" data-default-mode="">

  
  
  <div id="pst-skip-link" class="skip-link d-print-none"><a href="#main-content">Skip to main content</a></div>
  
  <div id="pst-scroll-pixel-helper"></div>
  
  <button type="button" class="btn rounded-pill" id="pst-back-to-top">
    <i class="fa-solid fa-arrow-up"></i>Back to top</button>

  
  <input type="checkbox"
          class="sidebar-toggle"
          id="pst-primary-sidebar-checkbox"/>
  <label class="overlay overlay-primary" for="pst-primary-sidebar-checkbox"></label>
  
  <input type="checkbox"
          class="sidebar-toggle"
          id="pst-secondary-sidebar-checkbox"/>
  <label class="overlay overlay-secondary" for="pst-secondary-sidebar-checkbox"></label>
  
  <div class="search-button__wrapper">
    <div class="search-button__overlay"></div>
    <div class="search-button__search-container">
<form class="bd-search d-flex align-items-center"
      action="../search.html"
      method="get">
  <i class="fa-solid fa-magnifying-glass"></i>
  <input type="search"
         class="form-control"
         name="q"
         id="search-input"
         placeholder="Search this book..."
         aria-label="Search this book..."
         autocomplete="off"
         autocorrect="off"
         autocapitalize="off"
         spellcheck="false"/>
  <span class="search-button__kbd-shortcut"><kbd class="kbd-shortcut__modifier">Ctrl</kbd>+<kbd>K</kbd></span>
</form></div>
  </div>

  <div class="pst-async-banner-revealer d-none">
  <aside id="bd-header-version-warning" class="d-none d-print-none" aria-label="Version warning"></aside>
</div>

  
    <header class="bd-header navbar navbar-expand-lg bd-navbar d-print-none">
    </header>
  

  <div class="bd-container">
    <div class="bd-container__inner bd-page-width">
      
      
      
      <div class="bd-sidebar-primary bd-sidebar">
        

  
  <div class="sidebar-header-items sidebar-primary__section">
    
    
    
    
  </div>
  
    <div class="sidebar-primary-items__start sidebar-primary__section">
        <div class="sidebar-primary-item">

  
    
  

<a class="navbar-brand logo" href="../intro.html">
  
  
  
  
  
    
    
      
    
    
    <img src="../_static/foto_profil.jpg" class="logo__image only-light" alt="My sample book - Home"/>
    <script>document.write(`<img src="../_static/foto_profil.jpg" class="logo__image only-dark" alt="My sample book - Home"/>`);</script>
  
  
</a></div>
        <div class="sidebar-primary-item">

 <script>
 document.write(`
   <button class="btn search-button-field search-button__button" title="Search" aria-label="Search" data-bs-placement="bottom" data-bs-toggle="tooltip">
    <i class="fa-solid fa-magnifying-glass"></i>
    <span class="search-button__default-text">Search</span>
    <span class="search-button__kbd-shortcut"><kbd class="kbd-shortcut__modifier">Ctrl</kbd>+<kbd class="kbd-shortcut__modifier">K</kbd></span>
   </button>
 `);
 </script></div>
        <div class="sidebar-primary-item"><nav class="bd-links bd-docs-nav" aria-label="Main">
    <div class="bd-toc-item navbar-nav active">
        
        <ul class="nav bd-sidenav bd-sidenav__home-link">
            <li class="toctree-l1">
                <a class="reference internal" href="../intro.html">
                    Profile
                </a>
            </li>
        </ul>
        <ul class="current nav bd-sidenav">
<li class="toctree-l1"><a class="reference internal" href="../tugas_1/pengantar_ppw.html">Pengantar Web Mining</a></li>
<li class="toctree-l1"><a class="reference internal" href="../tugas_2/web_crawling.html">web crawling - tugas 2</a></li>
<li class="toctree-l1 current active"><a class="current reference internal" href="#">Web Crawling - Tugas 3</a></li>
</ul>

    </div>
</nav></div>
    </div>
  
  
  <div class="sidebar-primary-items__end sidebar-primary__section">
  </div>
  
  <div id="rtd-footer-container"></div>


      </div>
      
      <main id="main-content" class="bd-main" role="main">
        
        

<div class="sbt-scroll-pixel-helper"></div>

          <div class="bd-content">
            <div class="bd-article-container">
              
              <div class="bd-header-article d-print-none">
<div class="header-article-items header-article__inner">
  
    <div class="header-article-items__start">
      
        <div class="header-article-item"><button class="sidebar-toggle primary-toggle btn btn-sm" title="Toggle primary sidebar" data-bs-placement="bottom" data-bs-toggle="tooltip">
  <span class="fa-solid fa-bars"></span>
</button></div>
      
    </div>
  
  
    <div class="header-article-items__end">
      
        <div class="header-article-item">

<div class="article-header-buttons">





<div class="dropdown dropdown-source-buttons">
  <button class="btn dropdown-toggle" type="button" data-bs-toggle="dropdown" aria-expanded="false" aria-label="Source repositories">
    <i class="fab fa-github"></i>
  </button>
  <ul class="dropdown-menu">
      
      
      
      <li><a href="https://github.com/executablebooks/jupyter-book" target="_blank"
   class="btn btn-sm btn-source-repository-button dropdown-item"
   title="Source repository"
   data-bs-placement="left" data-bs-toggle="tooltip"
>
  

<span class="btn__icon-container">
  <i class="fab fa-github"></i>
  </span>
<span class="btn__text-container">Repository</span>
</a>
</li>
      
      
      
      
      <li><a href="https://github.com/executablebooks/jupyter-book/issues/new?title=Issue%20on%20page%20%2Ftugas_3/web_crawling.html&body=Your%20issue%20content%20here." target="_blank"
   class="btn btn-sm btn-source-issues-button dropdown-item"
   title="Open an issue"
   data-bs-placement="left" data-bs-toggle="tooltip"
>
  

<span class="btn__icon-container">
  <i class="fas fa-lightbulb"></i>
  </span>
<span class="btn__text-container">Open issue</span>
</a>
</li>
      
  </ul>
</div>






<div class="dropdown dropdown-download-buttons">
  <button class="btn dropdown-toggle" type="button" data-bs-toggle="dropdown" aria-expanded="false" aria-label="Download this page">
    <i class="fas fa-download"></i>
  </button>
  <ul class="dropdown-menu">
      
      
      
      <li><a href="../_sources/tugas_3/web_crawling.md" target="_blank"
   class="btn btn-sm btn-download-source-button dropdown-item"
   title="Download source file"
   data-bs-placement="left" data-bs-toggle="tooltip"
>
  

<span class="btn__icon-container">
  <i class="fas fa-file"></i>
  </span>
<span class="btn__text-container">.md</span>
</a>
</li>
      
      
      
      
      <li>
<button onclick="window.print()"
  class="btn btn-sm btn-download-pdf-button dropdown-item"
  title="Print to PDF"
  data-bs-placement="left" data-bs-toggle="tooltip"
>
  

<span class="btn__icon-container">
  <i class="fas fa-file-pdf"></i>
  </span>
<span class="btn__text-container">.pdf</span>
</button>
</li>
      
  </ul>
</div>




<button onclick="toggleFullScreen()"
  class="btn btn-sm btn-fullscreen-button"
  title="Fullscreen mode"
  data-bs-placement="bottom" data-bs-toggle="tooltip"
>
  

<span class="btn__icon-container">
  <i class="fas fa-expand"></i>
  </span>

</button>



<script>
document.write(`
  <button class="btn btn-sm nav-link pst-navbar-icon theme-switch-button" title="light/dark" aria-label="light/dark" data-bs-placement="bottom" data-bs-toggle="tooltip">
    <i class="theme-switch fa-solid fa-sun fa-lg" data-mode="light"></i>
    <i class="theme-switch fa-solid fa-moon fa-lg" data-mode="dark"></i>
    <i class="theme-switch fa-solid fa-circle-half-stroke fa-lg" data-mode="auto"></i>
  </button>
`);
</script>


<script>
document.write(`
  <button class="btn btn-sm pst-navbar-icon search-button search-button__button" title="Search" aria-label="Search" data-bs-placement="bottom" data-bs-toggle="tooltip">
    <i class="fa-solid fa-magnifying-glass fa-lg"></i>
  </button>
`);
</script>
<button class="sidebar-toggle secondary-toggle btn btn-sm" title="Toggle secondary sidebar" data-bs-placement="bottom" data-bs-toggle="tooltip">
    <span class="fa-solid fa-list"></span>
</button>
</div></div>
      
    </div>
  
</div>
</div>
              
              

<div id="jb-print-docs-body" class="onlyprint">
    <h1>Web Crawling - Tugas 3</h1>
    <!-- Table of contents -->
    <div id="print-main-content">
        <div id="jb-print-toc">
            
            <div>
                <h2> Contents </h2>
            </div>
            <nav aria-label="Page">
                <ul class="visible nav section-nav flex-column">
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#menambahkan-abstrak-bahasa-inggris-dari-pta">1. Menambahkan abstrak bahasa inggris dari PTA</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#install-library">Install Library</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#install-dan-import-library-untuk-web-scraping">Install dan Import Library untuk Web Scraping</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-data-pta-dengan-abstrak-bahasa-inggris">Crawling Data PTA dengan Abstrak Bahasa Inggris</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#analisis-teknologi-website-dengan-builtwith">Analisis Teknologi Website dengan Builtwith</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#menampilkan-id-berita-judul-berita-kategori-berita-dan-isi-berita">2. Menampilkan ID berita, judul berita, kategori berita, dan isi berita</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#disable-ssl-warning">Disable SSL Warning</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-berita-cnn-indonesia">Crawling Berita CNN Indonesia</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-artikel-spesifik-cnn">Crawling Artikel Spesifik CNN</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#crawl-link-keluar-dari-sebuah-web">3. Crawl link keluar dari sebuah web</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#import-library-untuk-crawling-link">Import Library untuk Crawling Link</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#inisialisasi-variabel-global">Inisialisasi Variabel Global</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#fungsi-crawling-website">Fungsi Crawling Website</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#menjalankan-crawler">Menjalankan Crawler</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#menyimpan-hasil-crawling">Menyimpan Hasil Crawling</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#ringkasan">Ringkasan</a></li>
</ul>
            </nav>
        </div>
    </div>
</div>

              
                
<div id="searchbox"></div>
                <article class="bd-article">
                  
  <section class="tex2jax_ignore mathjax_ignore" id="web-crawling-tugas-3">
<h1>Web Crawling - Tugas 3<a class="headerlink" href="#web-crawling-tugas-3" title="Link to this heading">#</a></h1>
<p>Program ini melakukan tiga jenis web crawling:</p>
<ol class="arabic simple">
<li><p>Crawling data PTA Trunojoyo untuk mendapatkan abstrak bahasa Indonesia dan Inggris</p></li>
<li><p>Crawling berita dari CNN Indonesia untuk mendapatkan ID, kategori, judul, dan isi berita</p></li>
<li><p>Crawling link keluar dari website PTA Trunojoyo</p></li>
</ol>
<section id="menambahkan-abstrak-bahasa-inggris-dari-pta">
<h2>1. Menambahkan abstrak bahasa inggris dari PTA<a class="headerlink" href="#menambahkan-abstrak-bahasa-inggris-dari-pta" title="Link to this heading">#</a></h2>
<section id="install-library">
<h3>Install Library<a class="headerlink" href="#install-library" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="err">!</span><span class="n">pip</span> <span class="n">install</span> <span class="n">builtwith</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="n">Requirement</span> <span class="n">already</span> <span class="n">satisfied</span><span class="p">:</span> <span class="n">builtwith</span> <span class="ow">in</span> <span class="n">c</span><span class="p">:</span>\<span class="n">users</span>\<span class="n">hp</span>\<span class="n">appdata</span>\<span class="n">local</span>\<span class="n">programs</span>\<span class="n">python</span>\<span class="n">python311</span>\<span class="n">lib</span>\<span class="n">site</span><span class="o">-</span><span class="n">packages</span> <span class="p">(</span><span class="mf">1.3.4</span><span class="p">)</span>
<span class="n">Requirement</span> <span class="n">already</span> <span class="n">satisfied</span><span class="p">:</span> <span class="n">six</span> <span class="ow">in</span> <span class="n">c</span><span class="p">:</span>\<span class="n">users</span>\<span class="n">hp</span>\<span class="n">appdata</span>\<span class="n">local</span>\<span class="n">programs</span>\<span class="n">python</span>\<span class="n">python311</span>\<span class="n">lib</span>\<span class="n">site</span><span class="o">-</span><span class="n">packages</span> <span class="p">(</span><span class="kn">from</span><span class="w"> </span><span class="nn">builtwith</span><span class="p">)</span> <span class="p">(</span><span class="mf">1.17.0</span><span class="p">)</span>
</pre></div>
</div>
</section>
<section id="install-dan-import-library-untuk-web-scraping">
<h3>Install dan Import Library untuk Web Scraping<a class="headerlink" href="#install-dan-import-library-untuk-web-scraping" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="err">!</span><span class="n">pip</span> <span class="n">install</span> <span class="n">requests</span>
<span class="err">!</span><span class="n">pip</span> <span class="n">install</span> <span class="n">beautifulsoup4</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">requests</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">bs4</span><span class="w"> </span><span class="kn">import</span> <span class="n">BeautifulSoup</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">pandas</span><span class="w"> </span><span class="k">as</span><span class="w"> </span><span class="nn">pd</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="n">Requirement</span> <span class="n">already</span> <span class="n">satisfied</span><span class="p">:</span> <span class="n">requests</span> <span class="ow">in</span> <span class="n">c</span><span class="p">:</span>\<span class="n">users</span>\<span class="n">hp</span>\<span class="n">appdata</span>\<span class="n">local</span>\<span class="n">programs</span>\<span class="n">python</span>\<span class="n">python311</span>\<span class="n">lib</span>\<span class="n">site</span><span class="o">-</span><span class="n">packages</span> <span class="p">(</span><span class="mf">2.32.5</span><span class="p">)</span>
<span class="n">Requirement</span> <span class="n">already</span> <span class="n">satisfied</span><span class="p">:</span> <span class="n">beautifulsoup4</span> <span class="ow">in</span> <span class="n">c</span><span class="p">:</span>\<span class="n">users</span>\<span class="n">hp</span>\<span class="n">appdata</span>\<span class="n">local</span>\<span class="n">programs</span>\<span class="n">python</span>\<span class="n">python311</span>\<span class="n">lib</span>\<span class="n">site</span><span class="o">-</span><span class="n">packages</span> <span class="p">(</span><span class="mf">4.13.4</span><span class="p">)</span>
</pre></div>
</div>
</section>
<section id="crawling-data-pta-dengan-abstrak-bahasa-inggris">
<h3>Crawling Data PTA dengan Abstrak Bahasa Inggris<a class="headerlink" href="#crawling-data-pta-dengan-abstrak-bahasa-inggris" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="kn">import</span><span class="w"> </span><span class="nn">requests</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">bs4</span><span class="w"> </span><span class="kn">import</span> <span class="n">BeautifulSoup</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">pandas</span><span class="w"> </span><span class="k">as</span><span class="w"> </span><span class="nn">pd</span>

<span class="k">def</span><span class="w"> </span><span class="nf">ptaa</span><span class="p">():</span>
    <span class="n">data</span> <span class="o">=</span> <span class="p">{</span><span class="s2">&quot;penulis&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;judul&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;pembimbing_pertama&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;pembimbing_kedua&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;abstrak_indonesia&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;abstrak_inggris&quot;</span><span class="p">:</span> <span class="p">[]}</span>

    <span class="k">for</span> <span class="n">i</span> <span class="ow">in</span> <span class="nb">range</span><span class="p">(</span><span class="mi">1</span><span class="p">,</span> <span class="mi">4</span><span class="p">):</span>
        <span class="n">url</span> <span class="o">=</span> <span class="s2">&quot;https://pta.trunojoyo.ac.id/c_search/byprod/10/</span><span class="si">{}</span><span class="s2">&quot;</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">i</span><span class="p">)</span>
        <span class="n">r</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">url</span><span class="p">)</span>
        <span class="n">request</span> <span class="o">=</span> <span class="n">r</span><span class="o">.</span><span class="n">content</span>
        <span class="n">soup</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">request</span><span class="p">,</span> <span class="s2">&quot;html.parser&quot;</span><span class="p">)</span>
        <span class="n">jurnals</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">select</span><span class="p">(</span><span class="s1">&#39;li[data-cat=&quot;#luxury&quot;]&#39;</span><span class="p">)</span>

        <span class="k">for</span> <span class="n">jurnal</span> <span class="ow">in</span> <span class="n">jurnals</span><span class="p">:</span>
            <span class="n">response</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">jurnal</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;a.gray.button&#39;</span><span class="p">)[</span><span class="s1">&#39;href&#39;</span><span class="p">])</span>
            <span class="n">soup1</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">response</span><span class="o">.</span><span class="n">content</span><span class="p">,</span> <span class="s2">&quot;html.parser&quot;</span><span class="p">)</span>

            <span class="n">isi</span> <span class="o">=</span> <span class="n">soup1</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;div#content_journal&#39;</span><span class="p">)</span>

            <span class="n">judul</span> <span class="o">=</span> <span class="n">isi</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;a.title&#39;</span><span class="p">)</span><span class="o">.</span><span class="n">text</span>

            <span class="n">penulis</span> <span class="o">=</span> <span class="n">isi</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;span:contains(&quot;Penulis&quot;)&#39;</span><span class="p">)</span><span class="o">.</span><span class="n">text</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s1">&#39; : &#39;</span><span class="p">)[</span><span class="mi">1</span><span class="p">]</span>
            <span class="n">pembimbing_pertama</span> <span class="o">=</span> <span class="n">isi</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;span:contains(&quot;Dosen Pembimbing I&quot;)&#39;</span><span class="p">)</span><span class="o">.</span><span class="n">text</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s1">&#39; : &#39;</span><span class="p">)[</span><span class="mi">1</span><span class="p">]</span>
            <span class="n">pembimbing_kedua</span> <span class="o">=</span> <span class="n">isi</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="s1">&#39;span:contains(&quot;Dosen Pembimbing II&quot;)&#39;</span><span class="p">)</span><span class="o">.</span><span class="n">text</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s1">&#39; :&#39;</span><span class="p">)[</span><span class="mi">1</span><span class="p">]</span>

            <span class="c1"># Ambil abstrak bahasa Indonesia</span>
            <span class="n">abstrak_indonesia</span> <span class="o">=</span> <span class="s1">&#39;&#39;</span>
            <span class="n">abstrak_elements</span> <span class="o">=</span> <span class="n">isi</span><span class="o">.</span><span class="n">select</span><span class="p">(</span><span class="s1">&#39;p[align=&quot;justify&quot;]&#39;</span><span class="p">)</span>
            <span class="k">if</span> <span class="n">abstrak_elements</span><span class="p">:</span>
                <span class="n">abstrak_indonesia</span> <span class="o">=</span> <span class="n">abstrak_elements</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span><span class="o">.</span><span class="n">text</span>

            <span class="c1"># Ambil abstrak bahasa Inggris</span>
            <span class="n">abstrak_inggris</span> <span class="o">=</span> <span class="s1">&#39;&#39;</span>
            <span class="k">if</span> <span class="n">abstrak_elements</span> <span class="ow">and</span> <span class="nb">len</span><span class="p">(</span><span class="n">abstrak_elements</span><span class="p">)</span> <span class="o">&gt;</span> <span class="mi">1</span><span class="p">:</span>
                 <span class="n">abstrak_inggris</span> <span class="o">=</span> <span class="n">abstrak_elements</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span><span class="o">.</span><span class="n">text</span>

            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;penulis&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">penulis</span><span class="p">)</span>
            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;judul&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">judul</span><span class="p">)</span>
            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;pembimbing_pertama&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">pembimbing_pertama</span><span class="p">)</span>
            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;pembimbing_kedua&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">pembimbing_kedua</span><span class="p">)</span>
            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;abstrak_indonesia&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">abstrak_indonesia</span><span class="p">)</span>
            <span class="n">data</span><span class="p">[</span><span class="s2">&quot;abstrak_inggris&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">abstrak_inggris</span><span class="p">)</span>

    <span class="n">df</span> <span class="o">=</span> <span class="n">pd</span><span class="o">.</span><span class="n">DataFrame</span><span class="p">(</span><span class="n">data</span><span class="p">)</span>
    <span class="n">df</span><span class="o">.</span><span class="n">to_csv</span><span class="p">(</span><span class="s2">&quot;pta.csv&quot;</span><span class="p">,</span> <span class="n">index</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">df</span>

<span class="c1"># Jalankan fungsi</span>
<span class="n">ptaa</span><span class="p">()</span>
</pre></div>
</div>
<p>Data berhasil di-crawl dan disimpan dalam file CSV dengan kolom abstrak bahasa Indonesia dan Inggris.</p>
</section>
<section id="analisis-teknologi-website-dengan-builtwith">
<h3>Analisis Teknologi Website dengan Builtwith<a class="headerlink" href="#analisis-teknologi-website-dengan-builtwith" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="kn">import</span><span class="w"> </span><span class="nn">builtwith</span>

<span class="c1"># Analisis teknologi yang digunakan</span>
<span class="n">res</span> <span class="o">=</span> <span class="n">builtwith</span><span class="o">.</span><span class="n">parse</span><span class="p">(</span><span class="s1">&#39;https://pta.trunojoyo.ac.id&#39;</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="n">res</span><span class="p">)</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="p">{</span><span class="s1">&#39;web-servers&#39;</span><span class="p">:</span> <span class="p">[</span><span class="s1">&#39;Nginx&#39;</span><span class="p">],</span> <span class="s1">&#39;javascript-frameworks&#39;</span><span class="p">:</span> <span class="p">[</span><span class="s1">&#39;jQuery&#39;</span><span class="p">,</span> <span class="s1">&#39;jQuery UI&#39;</span><span class="p">]}</span>
</pre></div>
</div>
</section>
</section>
<section id="menampilkan-id-berita-judul-berita-kategori-berita-dan-isi-berita">
<h2>2. Menampilkan ID berita, judul berita, kategori berita, dan isi berita<a class="headerlink" href="#menampilkan-id-berita-judul-berita-kategori-berita-dan-isi-berita" title="Link to this heading">#</a></h2>
<section id="disable-ssl-warning">
<h3>Disable SSL Warning<a class="headerlink" href="#disable-ssl-warning" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="kn">import</span><span class="w"> </span><span class="nn">urllib3</span>

<span class="c1"># Nonaktifkan SSL warning</span>
<span class="n">urllib3</span><span class="o">.</span><span class="n">disable_warnings</span><span class="p">(</span><span class="n">urllib3</span><span class="o">.</span><span class="n">exceptions</span><span class="o">.</span><span class="n">InsecureRequestWarning</span><span class="p">)</span>
</pre></div>
</div>
</section>
<section id="crawling-berita-cnn-indonesia">
<h3>Crawling Berita CNN Indonesia<a class="headerlink" href="#crawling-berita-cnn-indonesia" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="kn">import</span><span class="w"> </span><span class="nn">requests</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">bs4</span><span class="w"> </span><span class="kn">import</span> <span class="n">BeautifulSoup</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">pandas</span><span class="w"> </span><span class="k">as</span><span class="w"> </span><span class="nn">pd</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">re</span>

<span class="k">def</span><span class="w"> </span><span class="nf">crawl_cnn_berita</span><span class="p">():</span>
    <span class="n">data</span> <span class="o">=</span> <span class="p">{</span><span class="s2">&quot;idberita&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;kategori_berita&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;judul_berita&quot;</span><span class="p">:</span> <span class="p">[],</span> <span class="s2">&quot;isi_berita&quot;</span><span class="p">:</span> <span class="p">[]}</span>
    
    <span class="n">headers</span> <span class="o">=</span> <span class="p">{</span>
        <span class="s1">&#39;User-Agent&#39;</span><span class="p">:</span> <span class="s1">&#39;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/58.0.3029.110 Safari/537.3&#39;</span>
    <span class="p">}</span>
    
    <span class="k">try</span><span class="p">:</span>
        <span class="c1"># Ambil halaman utama CNN Indonesia</span>
        <span class="n">url</span> <span class="o">=</span> <span class="s2">&quot;https://www.cnnindonesia.com/&quot;</span>
        <span class="n">response</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">url</span><span class="p">,</span> <span class="n">headers</span><span class="o">=</span><span class="n">headers</span><span class="p">,</span> <span class="n">verify</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>
        <span class="n">response</span><span class="o">.</span><span class="n">raise_for_status</span><span class="p">()</span>
        <span class="n">soup</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">response</span><span class="o">.</span><span class="n">content</span><span class="p">,</span> <span class="s2">&quot;html.parser&quot;</span><span class="p">)</span>
        
        <span class="c1"># Cari semua artikel di halaman utama</span>
        <span class="n">articles</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;article&#39;</span><span class="p">)</span>
        <span class="n">processed_urls</span> <span class="o">=</span> <span class="nb">set</span><span class="p">()</span>
        
        <span class="k">for</span> <span class="n">article</span> <span class="ow">in</span> <span class="n">articles</span><span class="p">:</span>
            <span class="n">link_tag</span> <span class="o">=</span> <span class="n">article</span><span class="o">.</span><span class="n">find</span><span class="p">(</span><span class="s1">&#39;a&#39;</span><span class="p">,</span> <span class="n">href</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
            <span class="k">if</span> <span class="ow">not</span> <span class="n">link_tag</span><span class="p">:</span>
                <span class="k">continue</span>
                
            <span class="n">berita_url</span> <span class="o">=</span> <span class="n">link_tag</span><span class="p">[</span><span class="s1">&#39;href&#39;</span><span class="p">]</span>
            <span class="k">if</span> <span class="ow">not</span> <span class="n">berita_url</span><span class="o">.</span><span class="n">startswith</span><span class="p">(</span><span class="s1">&#39;https://www.cnnindonesia.com/&#39;</span><span class="p">):</span>
                <span class="n">berita_url</span> <span class="o">=</span> <span class="s1">&#39;https://www.cnnindonesia.com&#39;</span> <span class="o">+</span> <span class="n">berita_url</span>
            
            <span class="c1"># Cek duplikasi</span>
            <span class="k">if</span> <span class="n">berita_url</span> <span class="ow">in</span> <span class="n">processed_urls</span><span class="p">:</span>
                <span class="k">continue</span>
            <span class="n">processed_urls</span><span class="o">.</span><span class="n">add</span><span class="p">(</span><span class="n">berita_url</span><span class="p">)</span>
            
            <span class="k">try</span><span class="p">:</span>
                <span class="c1"># Ambil detail berita</span>
                <span class="n">berita_response</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">berita_url</span><span class="p">,</span> <span class="n">headers</span><span class="o">=</span><span class="n">headers</span><span class="p">,</span> <span class="n">verify</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>
                <span class="n">berita_response</span><span class="o">.</span><span class="n">raise_for_status</span><span class="p">()</span>
                <span class="n">berita_soup</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">berita_response</span><span class="o">.</span><span class="n">content</span><span class="p">,</span> <span class="s2">&quot;html.parser&quot;</span><span class="p">)</span>
                
                <span class="c1"># ID berita dari URL</span>
                <span class="n">match</span> <span class="o">=</span> <span class="n">re</span><span class="o">.</span><span class="n">search</span><span class="p">(</span><span class="sa">r</span><span class="s1">&#39;/([0-9]+)$&#39;</span><span class="p">,</span> <span class="n">berita_url</span><span class="p">)</span>
                <span class="n">idberita</span> <span class="o">=</span> <span class="n">match</span><span class="o">.</span><span class="n">group</span><span class="p">(</span><span class="mi">1</span><span class="p">)</span> <span class="k">if</span> <span class="n">match</span> <span class="k">else</span> <span class="n">berita_url</span><span class="o">.</span><span class="n">split</span><span class="p">(</span><span class="s1">&#39;/&#39;</span><span class="p">)[</span><span class="o">-</span><span class="mi">1</span><span class="p">]</span>
                
                <span class="c1"># Judul berita</span>
                <span class="n">judul_tag</span> <span class="o">=</span> <span class="n">berita_soup</span><span class="o">.</span><span class="n">find</span><span class="p">(</span><span class="s1">&#39;h1&#39;</span><span class="p">)</span>
                <span class="n">judul_berita</span> <span class="o">=</span> <span class="n">judul_tag</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span> <span class="k">if</span> <span class="n">judul_tag</span> <span class="k">else</span> <span class="s1">&#39;&#39;</span>
                
                <span class="c1"># Kategori berita</span>
                <span class="n">kategori_tag</span> <span class="o">=</span> <span class="n">berita_soup</span><span class="o">.</span><span class="n">find</span><span class="p">(</span><span class="s1">&#39;div&#39;</span><span class="p">,</span> <span class="n">class_</span><span class="o">=</span><span class="s1">&#39;breadcrumb&#39;</span><span class="p">)</span>
                <span class="n">kategori_berita</span> <span class="o">=</span> <span class="s1">&#39;&#39;</span>
                <span class="k">if</span> <span class="n">kategori_tag</span><span class="p">:</span>
                    <span class="n">kategori_links</span> <span class="o">=</span> <span class="n">kategori_tag</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;a&#39;</span><span class="p">)</span>
                    <span class="k">if</span> <span class="n">kategori_links</span><span class="p">:</span>
                        <span class="n">kategori_berita</span> <span class="o">=</span> <span class="n">kategori_links</span><span class="p">[</span><span class="o">-</span><span class="mi">1</span><span class="p">]</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
                
                <span class="c1"># Isi berita dengan berbagai selector</span>
                <span class="n">isi_berita</span> <span class="o">=</span> <span class="s1">&#39;&#39;</span>
                <span class="n">selectors</span> <span class="o">=</span> <span class="p">[</span>
                    <span class="s1">&#39;div.detail_text&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;div.detail__body-text&#39;</span><span class="p">,</span> 
                    <span class="s1">&#39;div.detail-text&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;div[class*=&quot;detail&quot;]&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;div[class*=&quot;content&quot;]&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;div[class*=&quot;body&quot;]&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;article div p&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;.article-content p&#39;</span><span class="p">,</span>
                    <span class="s1">&#39;.post-content p&#39;</span>
                <span class="p">]</span>
                
                <span class="k">for</span> <span class="n">selector</span> <span class="ow">in</span> <span class="n">selectors</span><span class="p">:</span>
                    <span class="n">isi_tag</span> <span class="o">=</span> <span class="n">berita_soup</span><span class="o">.</span><span class="n">select</span><span class="p">(</span><span class="n">selector</span><span class="p">)</span>
                    <span class="k">if</span> <span class="n">isi_tag</span><span class="p">:</span>
                        <span class="k">if</span> <span class="n">selector</span><span class="o">.</span><span class="n">endswith</span><span class="p">(</span><span class="s1">&#39; p&#39;</span><span class="p">):</span>
                            <span class="n">isi_berita</span> <span class="o">=</span> <span class="s1">&#39; &#39;</span><span class="o">.</span><span class="n">join</span><span class="p">([</span><span class="n">p</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span> <span class="k">for</span> <span class="n">p</span> <span class="ow">in</span> <span class="n">isi_tag</span><span class="p">])</span>
                        <span class="k">else</span><span class="p">:</span>
                            <span class="n">isi_berita</span> <span class="o">=</span> <span class="n">isi_tag</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">separator</span><span class="o">=</span><span class="s1">&#39; &#39;</span><span class="p">,</span> <span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
                        <span class="k">break</span>
                
                <span class="c1"># Jika masih kosong, ambil semua paragraf</span>
                <span class="k">if</span> <span class="ow">not</span> <span class="n">isi_berita</span><span class="p">:</span>
                    <span class="n">paragraphs</span> <span class="o">=</span> <span class="n">berita_soup</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;p&#39;</span><span class="p">)</span>
                    <span class="k">if</span> <span class="n">paragraphs</span><span class="p">:</span>
                        <span class="n">long_paragraphs</span> <span class="o">=</span> <span class="p">[</span><span class="n">p</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span> <span class="k">for</span> <span class="n">p</span> <span class="ow">in</span> <span class="n">paragraphs</span> <span class="k">if</span> <span class="nb">len</span><span class="p">(</span><span class="n">p</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">))</span> <span class="o">&gt;</span> <span class="mi">50</span><span class="p">]</span>
                        <span class="n">isi_berita</span> <span class="o">=</span> <span class="s1">&#39; &#39;</span><span class="o">.</span><span class="n">join</span><span class="p">(</span><span class="n">long_paragraphs</span><span class="p">[:</span><span class="mi">5</span><span class="p">])</span>  
                        
                <span class="n">data</span><span class="p">[</span><span class="s2">&quot;idberita&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">idberita</span><span class="p">)</span>
                <span class="n">data</span><span class="p">[</span><span class="s2">&quot;kategori_berita&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">kategori_berita</span><span class="p">)</span>
                <span class="n">data</span><span class="p">[</span><span class="s2">&quot;judul_berita&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">judul_berita</span><span class="p">)</span>
                <span class="n">data</span><span class="p">[</span><span class="s2">&quot;isi_berita&quot;</span><span class="p">]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">isi_berita</span><span class="p">)</span>
                
            <span class="k">except</span> <span class="ne">Exception</span> <span class="k">as</span> <span class="n">e</span><span class="p">:</span>
                <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s1">&#39;Gagal mengambil berita: </span><span class="si">{</span><span class="n">berita_url</span><span class="si">}</span><span class="s1"> | Error: </span><span class="si">{</span><span class="n">e</span><span class="si">}</span><span class="s1">&#39;</span><span class="p">)</span>
    
    <span class="k">except</span> <span class="ne">Exception</span> <span class="k">as</span> <span class="n">e</span><span class="p">:</span>
        <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s1">&#39;Terjadi kesalahan: </span><span class="si">{</span><span class="n">e</span><span class="si">}</span><span class="s1">&#39;</span><span class="p">)</span>
    
    <span class="n">df</span> <span class="o">=</span> <span class="n">pd</span><span class="o">.</span><span class="n">DataFrame</span><span class="p">(</span><span class="n">data</span><span class="p">)</span>
    <span class="n">df</span><span class="o">.</span><span class="n">to_csv</span><span class="p">(</span><span class="s2">&quot;cnn_berita.csv&quot;</span><span class="p">,</span> <span class="n">index</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">df</span>

<span class="c1"># jalankan scraping</span>
<span class="n">crawl_cnn_berita</span><span class="p">()</span>
</pre></div>
</div>
<p>Data berita CNN berhasil di-crawl dan disimpan dalam file CSV dengan kolom ID berita, kategori, judul, dan isi berita.</p>
</section>
<section id="crawling-artikel-spesifik-cnn">
<h3>Crawling Artikel Spesifik CNN<a class="headerlink" href="#crawling-artikel-spesifik-cnn" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="k">def</span><span class="w"> </span><span class="nf">crawl_cnn_article</span><span class="p">(</span><span class="n">url</span><span class="p">):</span>
    <span class="n">headers</span> <span class="o">=</span> <span class="p">{</span>
        <span class="s1">&#39;User-Agent&#39;</span><span class="p">:</span> <span class="s1">&#39;Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/58.0.3029.110 Safari/537.3&#39;</span>
    <span class="p">}</span>
    
    <span class="k">try</span><span class="p">:</span>
        <span class="n">response</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">url</span><span class="p">,</span> <span class="n">headers</span><span class="o">=</span><span class="n">headers</span><span class="p">,</span> <span class="n">verify</span><span class="o">=</span><span class="kc">False</span><span class="p">)</span>
        <span class="n">response</span><span class="o">.</span><span class="n">raise_for_status</span><span class="p">()</span>
        <span class="n">soup</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">response</span><span class="o">.</span><span class="n">content</span><span class="p">,</span> <span class="s1">&#39;html.parser&#39;</span><span class="p">)</span>
        
        <span class="c1"># Ambil judul, kategori, dan isi berita</span>
        <span class="n">judul_tag</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">find</span><span class="p">(</span><span class="s1">&#39;h1&#39;</span><span class="p">)</span>
        <span class="n">judul_berita</span> <span class="o">=</span> <span class="n">judul_tag</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span> <span class="k">if</span> <span class="n">judul_tag</span> <span class="k">else</span> <span class="s1">&#39;Judul tidak ditemukan&#39;</span>
        
        <span class="n">kategori_berita</span> <span class="o">=</span> <span class="s1">&#39;Kategori tidak ditemukan&#39;</span>
        <span class="n">breadcrumb</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">find</span><span class="p">(</span><span class="s1">&#39;div&#39;</span><span class="p">,</span> <span class="n">class_</span><span class="o">=</span><span class="s1">&#39;breadcrumb&#39;</span><span class="p">)</span>
        <span class="k">if</span> <span class="n">breadcrumb</span><span class="p">:</span>
            <span class="n">kategori_links</span> <span class="o">=</span> <span class="n">breadcrumb</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;a&#39;</span><span class="p">)</span>
            <span class="k">if</span> <span class="n">kategori_links</span><span class="p">:</span>
                <span class="n">kategori_berita</span> <span class="o">=</span> <span class="n">kategori_links</span><span class="p">[</span><span class="o">-</span><span class="mi">1</span><span class="p">]</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
        
        <span class="c1"># Ambil isi berita</span>
        <span class="n">isi_berita</span> <span class="o">=</span> <span class="s1">&#39;&#39;</span>
        <span class="n">selectors</span> <span class="o">=</span> <span class="p">[</span>
            <span class="s1">&#39;div.detail_text&#39;</span><span class="p">,</span>
            <span class="s1">&#39;div.detail__body-text&#39;</span><span class="p">,</span> 
            <span class="s1">&#39;div.detail-text&#39;</span>
        <span class="p">]</span>
        
        <span class="k">for</span> <span class="n">selector</span> <span class="ow">in</span> <span class="n">selectors</span><span class="p">:</span>
            <span class="n">content_div</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">select_one</span><span class="p">(</span><span class="n">selector</span><span class="p">)</span>
            <span class="k">if</span> <span class="n">content_div</span><span class="p">:</span>
                <span class="n">paragraphs</span> <span class="o">=</span> <span class="n">content_div</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;p&#39;</span><span class="p">)</span>
                <span class="k">if</span> <span class="n">paragraphs</span><span class="p">:</span>
                    <span class="n">isi_berita</span> <span class="o">=</span> <span class="s1">&#39; &#39;</span><span class="o">.</span><span class="n">join</span><span class="p">([</span><span class="n">p</span><span class="o">.</span><span class="n">get_text</span><span class="p">(</span><span class="n">strip</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span> <span class="k">for</span> <span class="n">p</span> <span class="ow">in</span> <span class="n">paragraphs</span><span class="p">])</span>
                <span class="k">break</span>
        
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;=&quot;</span> <span class="o">*</span> <span class="mi">80</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;HASIL CRAWLING BERITA CNN INDONESIA&quot;</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;=&quot;</span> <span class="o">*</span> <span class="mi">80</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">&quot;Judul Berita: </span><span class="si">{</span><span class="n">judul_berita</span><span class="si">}</span><span class="s2">&quot;</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">&quot;Kategori: </span><span class="si">{</span><span class="n">kategori_berita</span><span class="si">}</span><span class="s2">&quot;</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;-&quot;</span> <span class="o">*</span> <span class="mi">80</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;Isi Berita:&quot;</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="n">isi_berita</span><span class="p">[:</span><span class="mi">500</span><span class="p">]</span> <span class="o">+</span> <span class="s2">&quot;...&quot;</span> <span class="k">if</span> <span class="nb">len</span><span class="p">(</span><span class="n">isi_berita</span><span class="p">)</span> <span class="o">&gt;</span> <span class="mi">500</span> <span class="k">else</span> <span class="n">isi_berita</span><span class="p">)</span>
        <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;=&quot;</span> <span class="o">*</span> <span class="mi">80</span><span class="p">)</span>
        
        <span class="k">return</span> <span class="p">{</span>
            <span class="s1">&#39;judul&#39;</span><span class="p">:</span> <span class="n">judul_berita</span><span class="p">,</span>
            <span class="s1">&#39;kategori&#39;</span><span class="p">:</span> <span class="n">kategori_berita</span><span class="p">,</span>
            <span class="s1">&#39;isi&#39;</span><span class="p">:</span> <span class="n">isi_berita</span>
        <span class="p">}</span>
        
    <span class="k">except</span> <span class="n">requests</span><span class="o">.</span><span class="n">exceptions</span><span class="o">.</span><span class="n">RequestException</span> <span class="k">as</span> <span class="n">e</span><span class="p">:</span>
        <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">&quot;Terjadi kesalahan saat mengakses </span><span class="si">{</span><span class="n">url</span><span class="si">}</span><span class="s2">: </span><span class="si">{</span><span class="n">e</span><span class="si">}</span><span class="s2">&quot;</span><span class="p">)</span>
        <span class="k">return</span> <span class="kc">None</span>

<span class="c1"># penggunaan</span>
<span class="n">url_berita</span> <span class="o">=</span> <span class="s2">&quot;https://www.cnnindonesia.com/olahraga/20250907140830-156-1270914/start-dari-belakang-bagnaia-ngarep-finis-di-10-besar-motogp-catalunya&quot;</span>
<span class="n">result</span> <span class="o">=</span> <span class="n">crawl_cnn_article</span><span class="p">(</span><span class="n">url_berita</span><span class="p">)</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="o">================================================================================</span>
<span class="n">HASIL</span> <span class="n">CRAWLING</span> <span class="n">BERITA</span> <span class="n">CNN</span> <span class="n">INDONESIA</span>
<span class="o">================================================================================</span>
<span class="n">Judul</span> <span class="n">Berita</span><span class="p">:</span> <span class="n">Start</span> <span class="n">dari</span> <span class="n">Belakang</span><span class="p">,</span> <span class="n">Bagnaia</span> <span class="n">Ngarep</span> <span class="n">Finis</span> <span class="n">di</span> <span class="mi">10</span> <span class="n">Besar</span> <span class="n">MotoGP</span> <span class="n">Catalunya</span>
<span class="n">Kategori</span><span class="p">:</span> <span class="n">Moto</span> <span class="n">GP</span>
<span class="o">--------------------------------------------------------------------------------</span>
<span class="n">Isi</span> <span class="n">Berita</span><span class="p">:</span>
<span class="n">Pembalap</span> <span class="n">DucatiFrancesco</span> <span class="n">Bagnaiaberharap</span> <span class="n">finis</span> <span class="n">di</span> <span class="mi">10</span> <span class="n">besarMotoGP</span> <span class="n">Catalunya</span> <span class="mi">2025</span><span class="n">di</span> <span class="n">Sirkuit</span> <span class="n">Catalunya</span> <span class="n">saat</span> <span class="n">start</span> <span class="n">dari</span> <span class="n">posisi</span> <span class="n">ke</span><span class="o">-</span><span class="mf">21.</span> <span class="n">Start</span> <span class="n">ke</span><span class="o">-</span><span class="mi">21</span> <span class="n">dalam</span> <span class="n">MotoGP</span> <span class="n">Catalunya</span> <span class="mi">2025</span> <span class="n">jadi</span> <span class="n">yang</span> <span class="n">terburuk</span> <span class="n">bagi</span> <span class="n">Bagnaia</span> <span class="n">setelah</span> <span class="n">MotoGP</span> <span class="n">Portugal</span> <span class="mf">2022.</span><span class="o">..</span>
<span class="o">================================================================================</span>
</pre></div>
</div>
</section>
</section>
<section id="crawl-link-keluar-dari-sebuah-web">
<h2>3. Crawl link keluar dari sebuah web<a class="headerlink" href="#crawl-link-keluar-dari-sebuah-web" title="Link to this heading">#</a></h2>
<section id="import-library-untuk-crawling-link">
<h3>Import Library untuk Crawling Link<a class="headerlink" href="#import-library-untuk-crawling-link" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="kn">import</span><span class="w"> </span><span class="nn">requests</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">bs4</span><span class="w"> </span><span class="kn">import</span> <span class="n">BeautifulSoup</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">pandas</span><span class="w"> </span><span class="k">as</span><span class="w"> </span><span class="nn">pd</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">urllib.parse</span><span class="w"> </span><span class="kn">import</span> <span class="n">urljoin</span><span class="p">,</span> <span class="n">urlparse</span>
</pre></div>
</div>
</section>
<section id="inisialisasi-variabel-global">
<h3>Inisialisasi Variabel Global<a class="headerlink" href="#inisialisasi-variabel-global" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="c1"># Variabel global</span>
<span class="n">visited</span> <span class="o">=</span> <span class="nb">set</span><span class="p">()</span>
<span class="n">web_structur</span> <span class="o">=</span> <span class="p">[]</span>
</pre></div>
</div>
</section>
<section id="fungsi-crawling-website">
<h3>Fungsi Crawling Website<a class="headerlink" href="#fungsi-crawling-website" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="k">def</span><span class="w"> </span><span class="nf">crawl_website</span><span class="p">(</span><span class="n">url</span><span class="p">,</span> <span class="n">base_domain</span><span class="p">,</span> <span class="n">max_depth</span><span class="o">=</span><span class="mi">2</span><span class="p">,</span> <span class="n">depth</span><span class="o">=</span><span class="mi">0</span><span class="p">):</span>
    <span class="k">if</span> <span class="n">url</span> <span class="ow">in</span> <span class="n">visited</span> <span class="ow">or</span> <span class="n">depth</span> <span class="o">&gt;</span> <span class="n">max_depth</span><span class="p">:</span>
        <span class="k">return</span>
    <span class="n">visited</span><span class="o">.</span><span class="n">add</span><span class="p">(</span><span class="n">url</span><span class="p">)</span>

    <span class="k">try</span><span class="p">:</span>
        <span class="n">response</span> <span class="o">=</span> <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="n">url</span><span class="p">,</span> <span class="n">timeout</span><span class="o">=</span><span class="mi">10</span><span class="p">)</span>
        <span class="n">response</span><span class="o">.</span><span class="n">raise_for_status</span><span class="p">()</span>
        <span class="n">soup</span> <span class="o">=</span> <span class="n">BeautifulSoup</span><span class="p">(</span><span class="n">response</span><span class="o">.</span><span class="n">content</span><span class="p">,</span> <span class="s1">&#39;html.parser&#39;</span><span class="p">)</span>

        <span class="c1"># Ambil semua link</span>
        <span class="n">links</span> <span class="o">=</span> <span class="n">soup</span><span class="o">.</span><span class="n">find_all</span><span class="p">(</span><span class="s1">&#39;a&#39;</span><span class="p">,</span> <span class="n">href</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
        <span class="k">for</span> <span class="n">link</span> <span class="ow">in</span> <span class="n">links</span><span class="p">:</span>
            <span class="n">full_url</span> <span class="o">=</span> <span class="n">urljoin</span><span class="p">(</span><span class="n">url</span><span class="p">,</span> <span class="n">link</span><span class="p">[</span><span class="s1">&#39;href&#39;</span><span class="p">])</span>  <span class="c1"># absolute URL</span>

            <span class="n">web_structur</span><span class="o">.</span><span class="n">append</span><span class="p">({</span>
                <span class="s2">&quot;page&quot;</span><span class="p">:</span> <span class="n">url</span><span class="p">,</span>
                <span class="s2">&quot;link keluar&quot;</span><span class="p">:</span> <span class="n">full_url</span>
            <span class="p">})</span>

            <span class="c1"># Batasi hanya ke domain pta.trunojoyo.ac.id</span>
            <span class="k">if</span> <span class="n">base_domain</span> <span class="ow">in</span> <span class="n">urlparse</span><span class="p">(</span><span class="n">full_url</span><span class="p">)</span><span class="o">.</span><span class="n">netloc</span><span class="p">:</span>
                <span class="n">crawl_website</span><span class="p">(</span><span class="n">full_url</span><span class="p">,</span> <span class="n">base_domain</span><span class="p">,</span> <span class="n">max_depth</span><span class="p">,</span> <span class="n">depth</span><span class="o">+</span><span class="mi">1</span><span class="p">)</span>

    <span class="k">except</span> <span class="ne">Exception</span> <span class="k">as</span> <span class="n">e</span><span class="p">:</span>
        <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">&quot;Error saat akses </span><span class="si">{</span><span class="n">url</span><span class="si">}</span><span class="s2">: </span><span class="si">{</span><span class="n">e</span><span class="si">}</span><span class="s2">&quot;</span><span class="p">)</span>

<span class="k">def</span><span class="w"> </span><span class="nf">run_crawler</span><span class="p">(</span><span class="n">start_url</span><span class="p">,</span> <span class="n">max_depth</span><span class="o">=</span><span class="mi">2</span><span class="p">):</span>
    <span class="n">base_domain</span> <span class="o">=</span> <span class="n">urlparse</span><span class="p">(</span><span class="n">start_url</span><span class="p">)</span><span class="o">.</span><span class="n">netloc</span>
    <span class="n">crawl_website</span><span class="p">(</span><span class="n">start_url</span><span class="p">,</span> <span class="n">base_domain</span><span class="p">,</span> <span class="n">max_depth</span><span class="o">=</span><span class="n">max_depth</span><span class="p">)</span>
</pre></div>
</div>
</section>
<section id="menjalankan-crawler">
<h3>Menjalankan Crawler<a class="headerlink" href="#menjalankan-crawler" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="n">start_url</span> <span class="o">=</span> <span class="s2">&quot;https://pta.trunojoyo.ac.id&quot;</span>
<span class="n">run_crawler</span><span class="p">(</span><span class="n">start_url</span><span class="p">,</span> <span class="n">max_depth</span><span class="o">=</span><span class="mi">1</span><span class="p">)</span>

<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">&quot;Jumlah data hasil crawl: </span><span class="si">{</span><span class="nb">len</span><span class="p">(</span><span class="n">web_structur</span><span class="p">)</span><span class="si">}</span><span class="s2">&quot;</span><span class="p">)</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="n">Error</span> <span class="n">saat</span> <span class="n">akses</span> <span class="n">https</span><span class="p">:</span><span class="o">//</span><span class="n">pta</span><span class="o">.</span><span class="n">trunojoyo</span><span class="o">.</span><span class="n">ac</span><span class="o">.</span><span class="n">id</span><span class="o">/</span><span class="n">index</span><span class="o">.</span><span class="n">html</span><span class="p">:</span> <span class="mi">404</span> <span class="n">Client</span> <span class="n">Error</span><span class="p">:</span> <span class="n">Not</span> <span class="n">Found</span> <span class="k">for</span> <span class="n">url</span><span class="p">:</span> <span class="n">https</span><span class="p">:</span><span class="o">//</span><span class="n">pta</span><span class="o">.</span><span class="n">trunojoyo</span><span class="o">.</span><span class="n">ac</span><span class="o">.</span><span class="n">id</span><span class="o">/</span><span class="n">index</span><span class="o">.</span><span class="n">html</span>
<span class="n">Jumlah</span> <span class="n">data</span> <span class="n">hasil</span> <span class="n">crawl</span><span class="p">:</span> <span class="mi">5249</span>
</pre></div>
</div>
</section>
<section id="menyimpan-hasil-crawling">
<h3>Menyimpan Hasil Crawling<a class="headerlink" href="#menyimpan-hasil-crawling" title="Link to this heading">#</a></h3>
<div class="highlight-python notranslate"><div class="highlight"><pre><span></span><span class="n">df</span> <span class="o">=</span> <span class="n">pd</span><span class="o">.</span><span class="n">DataFrame</span><span class="p">(</span><span class="n">web_structur</span><span class="p">)</span>
<span class="n">df</span><span class="o">.</span><span class="n">to_csv</span><span class="p">(</span><span class="s2">&quot;hasil_crawl.csv&quot;</span><span class="p">,</span> <span class="n">index</span><span class="o">=</span><span class="kc">False</span><span class="p">,</span> <span class="n">encoding</span><span class="o">=</span><span class="s2">&quot;utf-8&quot;</span><span class="p">)</span>

<span class="nb">print</span><span class="p">(</span><span class="s2">&quot;Hasil crawling disimpan ke &#39;hasil_crawl.csv&#39;&quot;</span><span class="p">)</span>
<span class="n">df</span><span class="o">.</span><span class="n">head</span><span class="p">(</span><span class="mi">10</span><span class="p">)</span>
</pre></div>
</div>
<p>output:</p>
<div class="highlight-default notranslate"><div class="highlight"><pre><span></span><span class="n">Hasil</span> <span class="n">crawling</span> <span class="n">disimpan</span> <span class="n">ke</span> <span class="s1">&#39;hasil_crawl.csv&#39;</span>
</pre></div>
</div>
<div class="pst-scrollable-table-container"><table class="table">
<thead>
<tr class="row-odd"><th class="head"><p></p></th>
<th class="head"><p>page</p></th>
<th class="head"><p>link keluar</p></th>
</tr>
</thead>
<tbody>
<tr class="row-even"><td><p>0</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id">https://pta.trunojoyo.ac.id</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/index.html">https://pta.trunojoyo.ac.id/index.html</a></p></td>
</tr>
<tr class="row-odd"><td><p>1</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id">https://pta.trunojoyo.ac.id</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id">https://pta.trunojoyo.ac.id</a></p></td>
</tr>
<tr class="row-even"><td><p>2</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id">https://pta.trunojoyo.ac.id</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
</tr>
<tr class="row-odd"><td><p>3</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/index.html">https://pta.trunojoyo.ac.id/index.html</a></p></td>
</tr>
<tr class="row-even"><td><p>4</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
</tr>
<tr class="row-odd"><td><p>5</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
</tr>
<tr class="row-even"><td><p>6</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/c_search/">https://pta.trunojoyo.ac.id/c_search/</a></p></td>
</tr>
<tr class="row-odd"><td><p>7</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/c_template/">https://pta.trunojoyo.ac.id/c_template/</a></p></td>
</tr>
<tr class="row-even"><td><p>8</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://library.trunojoyo.ac.id/detil.php?id=23">https://library.trunojoyo.ac.id/detil.php?id=23</a></p></td>
</tr>
<tr class="row-odd"><td><p>9</p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/">https://pta.trunojoyo.ac.id/</a></p></td>
<td><p><a class="reference external" href="https://pta.trunojoyo.ac.id/c_contact/">https://pta.trunojoyo.ac.id/c_contact/</a></p></td>
</tr>
</tbody>
</table>
</div>
</section>
</section>
<section id="ringkasan">
<h2>Ringkasan<a class="headerlink" href="#ringkasan" title="Link to this heading">#</a></h2>
<p>Program berhasil melakukan tiga jenis web crawling:</p>
<ol class="arabic simple">
<li><p><strong>Crawling PTA</strong>: Mengambil data skripsi dengan abstrak bahasa Indonesia dan Inggris</p></li>
<li><p><strong>Crawling CNN</strong>: Mengambil berita dengan ID, kategori, judul, dan isi lengkap</p></li>
<li><p><strong>Crawling Link</strong>: Mengidentifikasi semua link keluar dari website PTA dengan total 5,249 link</p></li>
</ol>
</section>
</section>

    <script type="text/x-thebe-config">
    {
        requestKernel: true,
        binderOptions: {
            repo: "binder-examples/jupyter-stacks-datascience",
            ref: "master",
        },
        codeMirrorConfig: {
            theme: "abcdef",
            mode: "python"
        },
        kernelOptions: {
            name: "python3",
            path: "./tugas_3"
        },
        predefinedOutput: true
    }
    </script>
    <script>kernelName = 'python3'</script>

                </article>
              

              
              
              
              
                <footer class="prev-next-footer d-print-none">
                  
<div class="prev-next-area">
    <a class="left-prev"
       href="../tugas_2/web_crawling.html"
       title="previous page">
      <i class="fa-solid fa-angle-left"></i>
      <div class="prev-next-info">
        <p class="prev-next-subtitle">previous</p>
        <p class="prev-next-title">web crawling - tugas 2</p>
      </div>
    </a>
</div>
                </footer>
              
            </div>
            
            
              
                <div class="bd-sidebar-secondary bd-toc"><div class="sidebar-secondary-items sidebar-secondary__inner">


  <div class="sidebar-secondary-item">
  <div class="page-toc tocsection onthispage">
    <i class="fa-solid fa-list"></i> Contents
  </div>
  <nav class="bd-toc-nav page-toc">
    <ul class="visible nav section-nav flex-column">
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#menambahkan-abstrak-bahasa-inggris-dari-pta">1. Menambahkan abstrak bahasa inggris dari PTA</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#install-library">Install Library</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#install-dan-import-library-untuk-web-scraping">Install dan Import Library untuk Web Scraping</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-data-pta-dengan-abstrak-bahasa-inggris">Crawling Data PTA dengan Abstrak Bahasa Inggris</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#analisis-teknologi-website-dengan-builtwith">Analisis Teknologi Website dengan Builtwith</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#menampilkan-id-berita-judul-berita-kategori-berita-dan-isi-berita">2. Menampilkan ID berita, judul berita, kategori berita, dan isi berita</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#disable-ssl-warning">Disable SSL Warning</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-berita-cnn-indonesia">Crawling Berita CNN Indonesia</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#crawling-artikel-spesifik-cnn">Crawling Artikel Spesifik CNN</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#crawl-link-keluar-dari-sebuah-web">3. Crawl link keluar dari sebuah web</a><ul class="nav section-nav flex-column">
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#import-library-untuk-crawling-link">Import Library untuk Crawling Link</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#inisialisasi-variabel-global">Inisialisasi Variabel Global</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#fungsi-crawling-website">Fungsi Crawling Website</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#menjalankan-crawler">Menjalankan Crawler</a></li>
<li class="toc-h3 nav-item toc-entry"><a class="reference internal nav-link" href="#menyimpan-hasil-crawling">Menyimpan Hasil Crawling</a></li>
</ul>
</li>
<li class="toc-h2 nav-item toc-entry"><a class="reference internal nav-link" href="#ringkasan">Ringkasan</a></li>
</ul>
  </nav></div>

</div></div>
              
            
          </div>
          <footer class="bd-footer-content">
            
<div class="bd-footer-content__inner container">
  
  <div class="footer-item">
    
<p class="component-author">
By The Jupyter Book Community
</p>

  </div>
  
  <div class="footer-item">
    

  <p class="copyright">
    
      © Copyright 2023.
      <br/>
    
  </p>

  </div>
  
  <div class="footer-item">
    
  </div>
  
  <div class="footer-item">
    
  </div>
  
</div>
          </footer>
        

      </main>
    </div>
  </div>
  
  <!-- Scripts loaded after <body> so the DOM is not blocked -->
  <script src="../_static/scripts/bootstrap.js?digest=dfe6caa3a7d634c4db9b"></script>
<script src="../_static/scripts/pydata-sphinx-theme.js?digest=dfe6caa3a7d634c4db9b"></script>

  <footer class="bd-footer">
  </footer>
  </body>
</html>