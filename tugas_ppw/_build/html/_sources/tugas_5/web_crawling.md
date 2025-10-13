# LDA model

## Imports & setup
```python
import requests
import string
import re
from bs4 import BeautifulSoup
import nltk
from nltk.corpus import stopwords
from Sastrawi.Stemmer.StemmerFactory import StemmerFactory
from gensim import corpora, models
from gensim.models import CoherenceModel
import numpy as np
```

## scraping berita kompas

```python
web = requests.get("https://surabaya.kompas.com/read/2025/10/13/073714878/keharuan-keluarga-saat-identitas-2-jasad-santri-ponpes-al-khoziny")
soup = BeautifulSoup(web.content, 'html.parser')
for a in soup(['script', 'style']):
    a.decompose()
text = ' '.join(soup.stripped_strings)
print("Scraping berhasil. Teks mentah siap diproses.")
```
Scraping berhasil. Teks mentah siap diproses.
## inisisalisasi preprocessing

```python
# Membuat stemmer dari Sastrawi
factory = StemmerFactory()
stemmer = factory.create_stemmer()

# Mengunduh dan menyiapkan daftar stopwords Bahasa Indonesia
nltk.download('stopwords')
stop_words = set(stopwords.words('indonesian'))

print("Stemmer dan Stopwords siap digunakan.")
```
[nltk_data] Downloading package stopwords to
[nltk_data]     C:\Users\hp\AppData\Roaming\nltk_data...
[nltk_data]   Package stopwords is already up-to-date!

## preprocessing per kalimat

```python
# Memecah teks mentah menjadi daftar kalimat (sentences)
sentences = re.split(r'[.!?]\s*', text) # Memecah berdasarkan tanda titik, tanda tanya, atau tanda seru diikuti oleh spasi
print(f"Teks berhasil dipecah menjadi {len(sentences)} kalimat.\n") # Ditambah \n
#perulangan untuk preprocessing pada setiap kalimat 
processed_docs = []
for sentence in sentences:
    # Hanya proses kalimat yang tidak terlalu pendek
    if len(sentence) > 20:
        #Case Folding & Cleaning (angka, tanda baca)
        sent = sentence.lower()
        sent = re.sub(r'\d+', '', sent)
        sent = sent.translate(str.maketrans('', '', string.punctuation))
        sent = sent.strip()

        #Stemming
        stemmed_sent = stemmer.stem(sent)

        #Tokenizing & Stopword Removal
        tokens = [word for word in stemmed_sent.split() if word not in stop_words]

        # Tambahkan hasil (daftar token) jika tidak kosong
        if tokens:
            processed_docs.append(tokens)

print(f"Preprocessing selesai. Dihasilkan {len(processed_docs)} dokumen yang siap diproses.")
print("\nContoh 3 dokumen pertama:")
for i, doc in enumerate(processed_docs[:3]):
    print(f"Dokumen {i}: {doc}")
```
Preprocessing selesai. Dihasilkan 42 dokumen yang siap diproses.

Contoh 3 dokumen pertama:

Dokumen 0: ['haru', 'keluarga', 'identitas', 'jasad', 'santri', 'ponpes', 'al', 'khoziny', 'identifikasi', 'otomatis', 'mode', 'gelap', 'mode', 'terang', 'login', 'gabung', 'kompas']

Dokumen 1: ['com', 'konten', 'simpan', 'konten', 'suka', 'atur', 'minat', 'ikan', 'masuk', 'langgan', 'kompas', 'one', 'news', 'nasional', 'global', 'megapolitan', 'regional', 'milu', 'hype', 'konsultasi', 'hukum', 'cek', 'fakta', 'surat', 'baca', 'indeks', 'kilas', 'daerah', 'kilas', 'korporasi', 'kilas', 'menteri', 'sorot', 'politik', 'kilas', 'badan', 'negara', 'kelana', 'indonesia', 'kalbe', 'health', 'corner', 'kilas', 'parlemen', 'kilas', 'bumn', 'nusaraya', 'sumatera', 'utara', 'sumatera', 'selatan', 'sumatera', 'barat', 'riau', 'lampung', 'banten', 'yogyakarta', 'jawa', 'barat', 'jawa', 'jawa', 'timur', 'kalimantan', 'barat', 'kalimantan', 'timur', 'sulawesi', 'selatan', 'bal', 'indeks', 'jagat', 'literasi', 'artikel', 'foto', 'video', 'rolls', 'donasi', 'stem', 'cahaya', 'aktual', 'doa', 'niat', 'jadwal', 'sholat', 'tekno', 'apps', 'os', 'gadget', 'internet', 'hardware', 'business', 'game', 'galeri', 'indeks', 'tech', 'innovation', 'kilas', 'internet', 'otomotif', 'motor', 'mobil', 'sport', 'niaga', 'komunitas', 'otopedia', 'rapah', 'ev', 'leadership', 'elektrifikasi', 'pamer', 'bola', 'timnas', 'indonesia', 'liga', 'indonesia', 'liga', 'inggris', 'liga', 'italia', 'liga', 'champions', 'klasemen', 'sports', 'motogp', 'badminton', 'indeks', 'lifestyle', 'wellness', 'fashion', 'relationship', 'parenting', 'beauty', 'grooming', 'buku', 'indeks', 'sadar', 'stunting', 'kilas', 'lifestyle', 'tren', 'lestari', 'health', 'sakit', 'az', 'kilas', 'sehat', 'money', 'ekbis', 'uang', 'syariah', 'industri', 'energi', 'karier', 'cuan', 'belanja', 'pajak', 'indeks', 'kilas', 'badan', 'kilas', 'transportasi', 'kilas', 'fintech', 'kilas', 'perban', 'kilas', 'investasi', 'transaksi', 'digital', 'jejak', 'umkm', 'properti', 'news', 'listing', 'properti', 'arsitektur', 'konstruksi', 'tips', 'properti', 'ikn', 'homey', 'indeks', 'sorot', 'properti', 'edukasi', 'sekolah', 'edu', 'news', 'guru', 'didik', 'khusus', 'beasiswa', 'literasi', 'skola', 'kilas', 'didik', 'ideaksi', 'travel', 'travel', 'news', 'travel', 'ideas', 'hotel', 'story', 'travelpedia', 'food', 'ohayo', 'jepang', 'indeks', 'video', 'parapuan', 'kolom', 'sains', 'jeo', 'foto', 'vik', 'netizen', 'warta', 'membership', 'kompas']

Dokumen 2: ['com', 'perintah', 'swasta', 'lsmfigur', 'bumn', 'umkm', 'nusatirta', 'sehat', 'hidup', 'sehat', 'sejahtera', 'air', 'bersih', 'sanitasi', 'layak', 'didik', 'didik', 'kualitas', 'lingkung', 'energi', 'bersih', 'jangkau', 'tangan', 'ubah', 'iklim', 'ekosistem', 'laut', 'ekosistem', 'darat', 'ekonomi', 'umkm', 'miskin', 'lapar', 'tara', 'gender', 'kerja', 'layak', 'tumbuh', 'ekonomi', 'industri', 'inovasi', 'infrastruktur', 'senjang', 'kota', 'mukim', 'konsumsi', 'produksi', 'bertanggungjawab', 'program', 'lestari', 'lihat', 'haru', 'keluarga', 'identitas', 'jasad', 'santri', 'ponpes', 'al', 'khoziny', 'identifikasi', 'komentar', 'baca', 'berita', 'iklan']

```python
# Membuat kamus: memetakan setiap kata unik ke sebuah ID
dictionary = corpora.Dictionary(processed_docs)

# Membuat corpus: mengubah setiap dokumen menjadi representasi BoW (ID kata, frekuensi)
corpus = [dictionary.doc2bow(doc) for doc in processed_docs]
print("\nCorpus berhasil dibuat.")
print("Contoh representasi BoW untuk dokumen pertama:")
print(corpus[0])
```

Dictionary berhasil dibuat dengan 183 kata unik setelah filtering.

Corpus berhasil dibuat.
Contoh representasi BoW untuk dokumen pertama:
[(0, 1), (1, 1), (2, 1), (3, 1), (4, 1), (5, 1), (6, 1), (7, 1), (8, 1), (9, 1), (10, 1), (11, 2), (12, 1), (13, 1), (14, 1), (15, 1)]

## LDA model

```python
# Tentukan jumlah topik yang ingin dicari
num_topics = 5

print(f"Memulai pelatihan model LDA untuk menemukan {num_topics} topik...")
if not corpus or not dictionary:
    print("Corpus atau Dictionary kosong. Tidak dapat melatih model LDA.")
else:
    lda_model = models.LdaModel(
        corpus=corpus,
        id2word=dictionary,
        num_topics=num_topics,
        random_state=100,  # Agar hasil bisa direproduksi
        passes=15          # Jumlah iterasi pelatihan
    )
    print("Model LDA berhasil dilatih!")
```
Memulai pelatihan model LDA untuk menemukan 5 topik...
Model LDA berhasil dilatih!

```python
if 'lda_model' in locals():
    print("Topik-topik yang ditemukan oleh model LDA:")
    print("-" * 50)
    for idx, topic in lda_model.print_topics(-1):
        print(f"Topik: {idx} \nKata Kunci: {topic}\n")

    # Evaluasi dengan Coherence Score dengan c_v
    coherence_model_lda = CoherenceModel(model=lda_model, texts=processed_docs, dictionary=dictionary, coherence='c_v')
    coherence_lda = coherence_model_lda.get_coherence()
    print("-" * 50)
    print(f'Coherence Score (c_v): {coherence_lda:.4f}')
    print("(Skor Coherence yang baik biasanya mendekati 1.0)")
else:
    print("Model LDA belum dilatih. Jalankan sel sebelumnya terlebih dahulu.")
```

Topik-topik yang ditemukan oleh model LDA:
--------------------------------------------------
Topik: 0 
Kata Kunci: 0.048*"com" + 0.038*"kompas" + 0.030*"artikel" + 0.024*"news" + 0.023*"properti" + 0.020*"berita" + 0.020*"tewas" + 0.020*"travel" + 0.019*"baca" + 0.019*"indonesia"

Topik: 1 
Kata Kunci: 0.081*"jenazah" + 0.060*"dvi" + 0.057*"kantong" + 0.050*"rs" + 0.050*"bhayangkara" + 0.038*"dna" + 0.037*"tim" + 0.026*"posko" + 0.026*"polri" + 0.026*"pusdokkes"

Topik: 2 
Kata Kunci: 0.051*"bangkal" + 0.042*"nomor" + 0.040*"jenazah" + 0.032*"kabupaten" + 0.022*"santri" + 0.022*"timur" + 0.022*"proses" + 0.022*"butuh" + 0.022*"dna" + 0.022*"nama"

Topik: 3 
Kata Kunci: 0.037*"identifikasi" + 0.031*"korban" + 0.024*"com" + 0.024*"jenazah" + 0.020*"laku" + 0.018*"body" + 0.016*"kantong" + 0.014*"ruang" + 0.014*"hasil" + 0.014*"metode"

Topik: 4 
Kata Kunci: 0.067*"ponpes" + 0.063*"surabaya" + 0.049*"al" + 0.046*"wib" + 0.046*"khoziny" + 0.043*"korban" + 0.035*"identifikasi" + 0.022*"ambruk" + 0.019*"baca" + 0.019*"keluarga"

--------------------------------------------------
Coherence Score (c_v): 0.5310
(Skor Coherence yang baik biasanya mendekati 1.0)
