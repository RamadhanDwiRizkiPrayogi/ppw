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

```python
# Membuat kamus: memetakan setiap kata unik ke sebuah ID
dictionary = corpora.Dictionary(processed_docs)

# Opsional: Filter kata-kata yang terlalu jarang atau terlalu sering muncul
dictionary.filter_extremes(no_below=2, no_above=0.7)
print(f"Dictionary berhasil dibuat dengan {len(dictionary)} kata unik setelah filtering.")

# Membuat corpus: mengubah setiap dokumen menjadi representasi BoW (ID kata, frekuensi)
corpus = [dictionary.doc2bow(doc) for doc in processed_docs]
print("\nCorpus berhasil dibuat.")
print("Contoh representasi BoW untuk dokumen pertama:")
print(corpus[0])
```

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
