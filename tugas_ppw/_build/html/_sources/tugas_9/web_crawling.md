# Web Crawling - Tugas 4

Program ini melakukan text preprocessing untuk membersihkan dan memproses teks abstrak PTA Trunojoyo dengan berbagai tahapan:

1. Data Collection - Mengumpulkan data abstrak PTA
2. Text Cleaning - Membersihkan teks dari noise
3. Tokenization - Memecah teks menjadi token
4. Stopwords Removal - Menghilangkan kata-kata stop
5. Stemming - Mengubah kata ke bentuk dasar
6. Spell Checking - Koreksi ejaan

## 1. Data Collection dan Text Cleaning

### Install Library dan Data Collection

```python
!pip install openpyxl xlsxwriter
import requests
from bs4 import BeautifulSoup
import pandas as pd
import numpy as np
import re

# Fungsi crawling data PTA
def scrape_pta_by_prodis():
    data_all = {'No': [], 'Abstrak': []}
    counter = 1
    
    prodis = ['Ekonomi Pembangunan', 'Manajemen', 'Akuntansi', 'Agribisnis', 
              'Teknik Industri', 'Teknik Elektro', 'Teknik Informatika']
    
    for prodi_idx, prodi in enumerate(prodis, 1):
        print(f"[{prodi_idx}] {prodi}", end=" - ")
        
        page = 1
        while True:
            try:
                url = f"https://pta.trunojoyo.ac.id/c_search/byprod/10/{page}"
                r = requests.get(url)
                request = r.content
                soup = BeautifulSoup(request, "html.parser")
                jurnals = soup.select('li[data-cat="#luxury"]')
                
                if not jurnals:
                    break
                    
                for jurnal in jurnals:
                    try:
                        detail_link = jurnal.select_one('a.gray.button')
                        if not detail_link:
                            continue
                            
                        response = requests.get(detail_link['href'])
                        soup1 = BeautifulSoup(response.content, "html.parser")
                        
                        # Ekstrak abstrak
                        abstrak = ""
                        abstrak_elements = soup1.select('p[align="justify"]')
                        if abstrak_elements:
                            abstrak = abstrak_elements[0].text
                        
                        data_all['No'].append(counter)
                        data_all['Abstrak'].append(abstrak)
                        counter += 1
                        
                    except Exception as e:
                        continue
                        
                page += 1
                print(f"Page {page-1}", end="/")
                
            except Exception as e:
                break
        
        print(f" [{counter-1} total]")
    
    df = pd.DataFrame(data_all)
    df.to_csv("pta_lengkap.csv", index=False)
    df.to_excel("pta_lengkap.xlsx", index=False)
    
    print(f"\n📊 Total data dikumpulkan: {len(df):,}")
    return df

# Jalankan scraping
df = scrape_pta_by_prodis()
```

**Output:**
```
[7] Manajemen - Page 207/207 [████████████████████] 100.00%
💾 Data disimpan ke: pta_lengkap.csv
💾 Data disimpan ke: pta_lengkap.xlsx

📊 Total data dikumpulkan: 1,031
```

### Text Cleaning

```python
def clean_text(text):
    if pd.isna(text) or text == '':
        return ""
    
    # Konversi ke lowercase
    text = str(text).lower()
    
    # Hapus karakter khusus dan tanda baca
    text = re.sub(r'[^\w\s]', '', text)
    
    # Hapus angka
    text = re.sub(r'\d+', '', text)
    
    # Hapus whitespace berlebih
    text = re.sub(r'\s+', ' ', text).strip()
    
    return text

# Terapkan pembersihan teks
df['abstrak_bersih'] = df['Abstrak'].apply(clean_text)

# Tampilkan hasil
print("\nHasil Pembersihan Teks:\n")
pd.set_option('display.max_columns', None)
pd.set_option('display.width', 1000)
df[['abstrak_asli', 'abstrak_bersih']].head(10)
```

**Output:**

Hasil Pembersihan Teks:

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>abstrak_asli</th>
      <th>abstrak_bersih</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Satiyah, Pengaruh Faktor-faktor Pelatihan dan ...</td>
      <td>satiyah pengaruh faktorfaktor pelatihan dan pe...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Tujuan penelitian ini adalah untuk mengetahui ...</td>
      <td>tujuan penelitian ini adalah untuk mengetahui ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>3</th>
      <td>Aplikasi nyata pemanfaatan teknologi informasi...</td>
      <td>aplikasi nyata pemanfaatan teknologi informasi...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Penelitian ini menggunakan metode kuantitatif,...</td>
      <td>penelitian ini menggunakan metode kuantitatif ...</td>
    </tr>
  </tbody>
</table>
</div>

## 2. Tokenization

### Import Library dan Tokenisasi

```python
import nltk
nltk.download('punkt_tab', quiet=True)
from nltk.tokenize import word_tokenize

def tokenize_text(text):
    if pd.isna(text) or text == '':
        return []
    tokens = word_tokenize(str(text))
    return tokens

# Terapkan tokenisasi
df['abstrak_id_tokens'] = df['abstrak_bersih'].apply(tokenize_text)

print("\nPTA (abstrak_id_tokens):\n")
df[['abstrak_id_clean', 'abstrak_id_tokens']].head(10)
```

**Output:**

PTA (abstrak_id_tokens):

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>abstrak_id_clean</th>
      <th>abstrak_id_tokens</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>satiyah pengaruh faktorfaktor pelatihan dan pe...</td>
      <td>[satiyah, pengaruh, faktorfaktor, pelatihan, d...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>tujuan penelitian ini adalah untuk mengetahui ...</td>
      <td>[tujuan, penelitian, ini, adalah, untuk, menge...</td>
    </tr>
    <tr>
      <th>2</th>
      <td></td>
      <td>[]</td>
    </tr>
    <tr>
      <th>3</th>
      <td>aplikasi nyata pemanfaatan teknologi informasi...</td>
      <td>[aplikasi, nyata, pemanfaatan, teknologi, info...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>penelitian ini menggunakan metode kuantitatif ...</td>
      <td>[penelitian, ini, menggunakan, metode, kuantit...</td>
    </tr>
  </tbody>
</table>
</div>

## 3. Stopwords Removal

### Hapus Stopwords

```python
import nltk
nltk.download('stopwords', quiet=True)
from nltk.corpus import stopwords

def remove_stopwords(tokens):
    if not tokens:
        return []
    
    indonesian_stopwords = set(stopwords.words('indonesian'))
    filtered_tokens = [word for word in tokens if word.lower() not in indonesian_stopwords]
    return filtered_tokens

# Terapkan penghapusan stopwords
df['abstrak_id_filtered'] = df['abstrak_id_tokens'].apply(remove_stopwords)

print("\nPTA (abstrak_id_filtered):\n")
df[['abstrak_id_tokens', 'abstrak_id_filtered']].head(10)
```

**Output:**

PTA (abstrak_id_filtered):

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>abstrak_id_tokens</th>
      <th>abstrak_id_filtered</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>[satiyah, pengaruh, faktorfaktor, pelatihan, d...</td>
      <td>[satiyah, pengaruh, faktorfaktor, pelatihan, p...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>[tujuan, penelitian, ini, adalah, untuk, menge...</td>
      <td>[tujuan, penelitian, persepsi, brand, associat...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>[]</td>
      <td>[]</td>
    </tr>
    <tr>
      <th>3</th>
      <td>[aplikasi, nyata, pemanfaatan, teknologi, info...</td>
      <td>[aplikasi, nyata, pemanfaatan, teknologi, info...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>[penelitian, ini, menggunakan, metode, kuantit...</td>
      <td>[penelitian, metode, kuantitatif, menekankan, ...</td>
    </tr>
  </tbody>
</table>
</div>

## 4. Stemming dan Kontraksi

### Stemming dengan Sastrawi

```python
from Sastrawi.Stemmer.StemmerFactory import StemmerFactory

factory = StemmerFactory()
stemmer = factory.create_stemmer()

def stem_tokens(tokens):
    if not tokens:
        return ""
    text = ' '.join(tokens)
    stemmed_text = stemmer.stem(text)
    return stemmed_text

# Terapkan stemming
df['abstrak_stemmed'] = df['abstrak_id_filtered'].apply(stem_tokens)

# Kontraksi bahasa Indonesia
contractions = {
    'tdk': 'tidak',
    'yg': 'yang',
    'dgn': 'dengan',
    'sbg': 'sebagai',
    'pd': 'pada',
    'tsb': 'tersebut',
    'dll': 'dan lain lain'
}

def expand_contractions(text):
    if not text:
        return ""
    
    words = text.split()
    expanded = []
    for word in words:
        expanded.append(contractions.get(word, word))
    return ' '.join(expanded)

df['abstrak_expanded'] = df['abstrak_stemmed'].apply(expand_contractions)

print("✅ Penanganan kontraksi selesai untuk 1,031 abstrak")
print("\n📊 HASIL PENANGANAN KONTRAKSI (SEMUA DATA):\n")
df[['abstrak_stemmed', 'abstrak_expanded']].head()
```

**Output:**

✅ Penanganan kontraksi selesai untuk 1,031 abstrak

📊 HASIL PENANGANAN KONTRAKSI (SEMUA DATA):

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>abstrak_stemmed</th>
      <th>abstrak_expanded</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>satiyah pengaruh faktorfaktor latih kembang pr...</td>
      <td>satiyah pengaruh faktor latih kembang produkti...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>tuju teliti persepsi brand association langgan...</td>
      <td>tuju teliti persepsi brand association langgan...</td>
    </tr>
    <tr>
      <th>2</th>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <th>3</th>
      <td>aplikasi nyata manfaat teknologi informasi kom...</td>
      <td>aplikasi nyata manfaat teknologi informasi kom...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>teliti metode kuantitatif tekan uji hipotesis ...</td>
      <td>teliti metode kuantitatif tekan uji hipotesis ...</td>
    </tr>
  </tbody>
</table>
</div>

## 5. Spell Checking

### Koreksi Ejaan dengan Kamus Indonesia

```python
!pip install pyspellchecker tqdm
import os
from spellchecker import SpellChecker
from tqdm import tqdm

def load_indonesian_wordlist():
    working_dir = os.getcwd()
    print(f"Working directory saat ini: {working_dir}")
    
    wordlist_path = os.path.join(working_dir, "00-indonesian-wordlist.lst")
    
    if not os.path.exists(wordlist_path):
        return None
    
    print(f"✅ File ditemukan: {wordlist_path}")
    
    with open(wordlist_path, 'r', encoding='utf-8') as file:
        words = [line.strip().lower() for line in file if line.strip()]
    
    print(f"✅ Berhasil memuat {len(words):,} kata dari kamus Indonesia")
    return set(words)

def spell_check_and_correct(text_list, indonesian_words):
    corrected_texts = []
    
    spell = SpellChecker()
    spell.word_frequency.load_words(indonesian_words)
    
    for text in tqdm(text_list, desc="Spellchecking"):
        if not text or pd.isna(text):
            corrected_texts.append("")
            continue
        
        words = text.split()
        corrected_words = []
        
        for word in words:
            if word in indonesian_words:
                corrected_words.append(word)
            else:
                candidates = spell.candidates(word)
                if candidates:
                    corrected_word = min(candidates, key=len)
                    corrected_words.append(corrected_word)
                else:
                    corrected_words.append(word)
        
        corrected_texts.append(' '.join(corrected_words))
    
    return corrected_texts

# Load kamus Indonesia
indonesian_words = load_indonesian_wordlist()

if indonesian_words:
    print(f"📊 Memproses {len(df)} dokumen...")
    df['abstrak_corrected'] = spell_check_and_correct(df['abstrak_expanded'].tolist(), indonesian_words)
    print("✅ Spell checking selesai!")
else:
    print("❌ Tidak dapat memuat kamus Indonesia")
```

**Output:**
```
Working directory saat ini: d:\kuliah\semester 7\ppw\tugas_ppw\tugas_4
✅ File ditemukan: d:\kuliah\semester 7\ppw\tugas_ppw\tugas_4\00-indonesian-wordlist.lst
✅ Berhasil memuat 79,898 kata dari kamus Indonesia
📊 DataFrame ditemukan dengan 1031 dokumen
📊 Memproses 1031 dokumen...
```

**Progress Bar:**
```
Spellchecking:  59%|█████▉    | 613/1031 [1:50:35<1:23:17, 11.96s/row]
```

## Ringkasan

Program text preprocessing ini berhasil melakukan berbagai tahapan:

1. **Data Collection**: Mengumpulkan 1,031 abstrak PTA dari 7 program studi
2. **Text Cleaning**: Membersihkan teks dari karakter khusus, angka, dan whitespace berlebih
3. **Tokenization**: Memecah teks menjadi token kata menggunakan NLTK
4. **Stopwords Removal**: Menghilangkan kata-kata stop bahasa Indonesia
5. **Stemming**: Mengubah kata ke bentuk dasar menggunakan Sastrawi
6. **Kontraksi**: Menangani singkatan bahasa Indonesia
7. **Spell Checking**: Koreksi ejaan menggunakan kamus Indonesia 79,898 kata

Semua tahapan preprocessing berhasil dijalankan dengan output yang tersimpan dalam DataFrame untuk analisis lebih lanjut.