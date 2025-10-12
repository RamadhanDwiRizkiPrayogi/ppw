# preprocessing, tf-idf dan cbow

## Imports & setup
```python
import requests
import string
import re

from bs4 import BeautifulSoup #untuk memisahkan teks yang ingin diambil dengan struktur html
import nltk
from nltk.corpus import stopwords

%pip install Sastrawi
from Sastrawi.Stemmer.StemmerFactory import StemmerFactory

%pip install plotly
%pip install --upgrade gensim
from gensim.models import Word2Vec, FastText

import numpy as np
```

## scraping (Kompas)
- Cell ini mengambil contoh halaman web dan mengekstrak teks mentah sebagai data input untuk pipeline.
```python
web = requests.get("https://kompas.com")
soup = BeautifulSoup(web.content, 'html.parser')
for a in soup(['script', 'style']):
    a.decompose()
text = ' '.join(soup.stripped_strings)
print(text)
```

## Penjelasan singkat
- Cell ini membersihkan teks mentah (lowercase, hapus angka/tanda baca, dan normalisasi spasi) agar siap diproses.

## Basic cleaning
```python
teks = text.lower()
teks = re.sub(r'\d+', '', teks)
teks = teks.translate(str.maketrans('', '', string.punctuation))
teks = re.sub(r'\s+', ' ', teks)
teks = teks.strip()
```

## Stemming & lemmatization
- Cell ini melakukan stemming (bahasa Indonesia) dan contoh lemmatization (bahasa Inggris) untuk menyatukan bentuk kata dasar.
```python
# Stemming
factory = StemmerFactory()
stemmer = factory.create_stemmer()
stemmed = stemmer.stem(teks)

# Lemmatization (menggunakan stemming hasil Sastrawi, karena lemmatizer khusus Indonesia belum tersedia)
output = stemmed
factory = StemmerFactory()
stemmer = factory.create_stemmer()
output = stemmer.stem(teks)

print("Hasil stemming/lemmatization (Bahasa Indonesia):", output)
```


## Tokenizing
- Cell ini memecah teks menjadi token/kata agar dapat dihitung frekuensi dan diproses lebih lanjut.
```python
#tokenizing
tokens = [t for t in output.split()]
print(tokens)
```

## Stopwords removal
- Cell ini menghapus stopword (kata umum yang kurang informatif) dari daftar token.
```python
# menghapus stopwords dari tokens
nltk.download('stopwords')
stop_words = set(stopwords.words('indonesian'))
filtered_tokens = [word for word in tokens if word not in stop_words]
print(filtered_tokens)
```

## TF-IDF (markdown explanation)
- Cell ini menjelaskan konsep TF‑IDF dan mengapa metrik tersebut berguna untuk menilai pentingnya kata dalam dokumen.
### tf-idf
TF-IDF (Term Frequency-Inverse Document Frequency): TF-IDF menghitung seberapa penting sebuah kata dalam sebuah dokumen relatif terhadap seluruh koleksi dokumen. Outputnya adalah matriks di mana setiap baris mewakili dokumen dan setiap kolom mewakili kata unik dalam seluruh koleksi. Nilai di dalam sel menunjukkan skor TF-IDF kata tersebut dalam dokumen. Karena setiap kolom mewakili sebuah kata, maka header kolomnya adalah kata-kata itu sendiri.

## TF-IDF code
- Cell ini menghitung matriks TF‑IDF dari korpus dan menyimpan hasilnya ke CSV untuk analisis lebih lanjut.
```python
from sklearn.feature_extraction.text import TfidfVectorizer
import pandas as pd

vectorizer = TfidfVectorizer()
tfidf_matrix = vectorizer.fit_transform(filtered_tokens)

# ubah ke DataFrame biar keliatan
tfidf_df = pd.DataFrame(
    tfidf_matrix.toarray(),
    columns=vectorizer.get_feature_names_out()
)

print("\nTF-IDF shape:", tfidf_df.shape)
print(tfidf_df.head())

# Simpan hasil TF-IDF ke CSV
# Jika Anda ingin menyimpan dokumen x fitur, indeks baris mewakili dokumen yang diberikan ke fit_transform
tfidf_df.to_csv('tfidf.csv', index=False, float_format='%.6f', encoding='utf-8')
```

## Word embedding (markdown explanation)
- Cell ini menjelaskan konsep word embedding (Word2Vec) dan perbedaan representasi terhadap TF‑IDF.
## word embedding

Word2Vec: Word2Vec adalah teknik word embedding yang memetakan setiap kata ke dalam sebuah vektor numerik dalam ruang berdimensi rendah (dalam kasus Anda, 56 dimensi). Outputnya adalah sekumpulan vektor, satu untuk setiap kata. Tabel yang Anda lihat menampilkan vektor-vektor ini. Setiap baris adalah kata, dan setiap kolom adalah salah satu dari 56 dimensi vektor tersebut. Oleh karena itu, header kolomnya adalah indeks dimensi (0 hingga 55), bukan kata-kata itu sendiri.

Singkatnya, TF-IDF menggunakan kata sebagai fitur, sedangkan Word2Vec menggunakan dimensi vektor sebagai fitur untuk merepresentasikan kata.


## Prepare corpus & train Word2Vec
- Cell ini menyiapkan korpus token dan melatih model Word2Vec (CBOW) untuk mendapatkan vektor kata.
```python
corpus = []
for col in filtered_tokens:
   word_list = col.split(" ")
   corpus.append(word_list)

#show first value
corpus[0:1]

#generate vectors from corpus
model = Word2Vec(corpus, min_count=1, vector_size = 56)
print(model)
```

## MeanEmbeddingVectorizer definition
- Cell ini membuat helper/class untuk mengubah kumpulan kata per dokumen menjadi vektor dokumen dengan merata‑ratakan vektor kata.
```python
class MyTokenizer:
    def fit_transform(self, texts):
        # Tokenisasi sederhana: lowercase + split
        return [str(text).lower().split() for text in texts]

class MeanEmbeddingVectorizer:
    def __init__(self, word2vec_model):
        self.word2vec = word2vec_model
        # Perbaikan: gunakan vector_size (Gensim ≥ 4.0)
        self.dim = word2vec_model.wv.vector_size

    def fit(self, X, y=None):
        return self

    def transform(self, X):
        X_tokenized = MyTokenizer().fit_transform(X)
        embeddings = []
        for words in X_tokenized:
            # Ambil vektor hanya untuk kata yang ada di vocab
            valid_vectors = [
                self.word2vec.wv[word] for word in words
                if word in self.word2vec.wv,
            ]
            if valid_vectors:
                embeddings.append(np.mean(valid_vectors, axis=0))
            else:
                embeddings.append(np.zeros(self.dim))
        return np.array(embeddings)

    def fit_transform(self, X, y=None):
        return self.transform(X)
```


## Create mean embeddings
- Cell ini menghitung embedding dokumen (rata‑rata vektor kata) sehingga dokumen dapat dibandingkan secara numerik.
```python
mean_embedding_vectorizer = MeanEmbeddingVectorizer(model)
mean_embedded = mean_embedding_vectorizer.fit_transform([' '.join(filtered_tokens)])
print(mean_embedded)
```

## Save per-term embeddings
- Cell ini mengekspor embedding kata dan tabel vektor ke CSV agar hasil model dapat dianalisis atau dibagikan.
```python
# Ambil semua fitur (kata unik) dari model
features = list(model.wv.index_to_key)

# Ambil vektor embedding untuk setiap fitur
embedding_vectors = [model.wv[word] for word in features]

# Buat DataFrame, setiap kolom adalah dimensi vektor
embedding_df = pd.DataFrame(embedding_vectors, index=features)
embedding_df.index.name = 'fitur'
print(embedding_df)

# Simpan per-term embeddings ke CSV
embedding_df.reset_index().to_csv('cbow_terms.csv', index=False, float_format='%.6f', encoding='utf-8')
print('Saved cbow_terms.csv')
