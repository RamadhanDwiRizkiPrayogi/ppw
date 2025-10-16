# UTS: Klasifikasi Teks Berita dengan LDA Feature Extraction dan SVM dan Naive Bayes

### Tugas ini bertujuan untuk melakukan Klasifikasi Teks Berita menggunakan:

LDA (Latent Dirichlet Allocation) sebagai metode ekstraksi fitur

SVM (Support Vector Machine) sebagai algoritma klasifikasi

### Tahapan Pengerjaan:
Persiapan Data Berlabel - Load dan split data training/testing

Pra-pemrosesan Teks - Data sudah di-preprocessing (kolom stemming_indo)

Ekstraksi Fitur dengan LDA - Transform teks menjadi distribusi topik

Pelatihan Model SVM - Latih classifier dengan fitur LDA

Evaluasi Hasil - Ukur performa dan bandingkan dengan TF-IDF


## 1. Imports & Setup
```python
# Import libraries untuk data manipulation
import pandas as pd
import numpy as np
import warnings
warnings.filterwarnings('ignore')

# Import libraries untuk visualisasi
import matplotlib.pyplot as plt
import seaborn as sns

# Import libraries untuk text processing
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.model_selection import train_test_split

# Import LDA
from sklearn.decomposition import LatentDirichletAllocation
# UTS: Klasifikasi Teks Berita — Terstruktur dari `code_berita.ipynb`

Dokumen ini adalah versi markdown yang disinkronkan dengan alur di `uts_ppw/code_berita.ipynb`. Saya menyalin sel-sel utama (judul, penjelasan, dan blok kode) ke dalam urutan yang sama. Untuk beberapa sel, output aktual bergantung pada data yang tersedia saat runtime — saya menyertakan output yang sudah ada (dari analisis artikel) dan menandai bagian lain supaya Anda menjalankan sel untuk melihat hasilnya.



## 1. Pendahuluan

### Tujuan
Tugas ini bertujuan melakukan klasifikasi teks berita menggunakan:
- LDA (Latent Dirichlet Allocation) sebagai metode ekstraksi fitur
- SVM (Support Vector Machine) dan Naive Bayes sebagai algoritma klasifikasi

### Tahapan Pengerjaan
- Persiapan Data Berlabel - Load dan split data training/testing
- Pra-pemrosesan Teks - Data sudah di-preprocessing (kolom stemming_indo) atau tersedia preprocessing artikel
- Ekstraksi Fitur dengan LDA - Transform teks menjadi distribusi topik
- Pelatihan Model SVM / Naive Bayes - Latih classifier dengan fitur LDA
- Evaluasi Hasil - Ukur performa dan bandingkan

---

## 2. Import Libraries
```python
# Import libraries untuk data manipulation
import pandas as pd
import numpy as np
import warnings
warnings.filterwarnings('ignore')

# Import libraries untuk visualisasi
import matplotlib.pyplot as plt
import seaborn as sns

# Import libraries untuk text processing
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.model_selection import train_test_split

# Import LDA
from sklearn.decomposition import LatentDirichletAllocation

# Import SVM dan metrics
from sklearn.svm import SVC
from sklearn.metrics import (accuracy_score, precision_score, recall_score,
                            f1_score, classification_report, confusion_matrix)

# Import untuk grid search
from sklearn.model_selection import GridSearchCV

print("✓ Libraries berhasil di-import")
```

Output (jika dijalankan):

✓ Libraries berhasil di-import

---

## 3. Load dan Eksplorasi Data
```python
# Load dataset
df = pd.read_csv('Berita.csv')

# Tampilkan info dataset
print("="*50)
print("INFORMASI DATASET")
print("="*50)
print(f"Jumlah data: {len(df)}")
print(f"Jumlah kolom: {len(df.columns)}")
print(f"\nKolom-kolom:")
print(df.columns.tolist())

# Tampilkan beberapa baris pertama
print("\n" + "="*50)
print("5 BARIS PERTAMA DATA")
print("="*50)
df.head()
```

Output: jalankan sel untuk melihat informasi dataset (tergantung file `Berita.csv` pada direktori ini).

---

## 4. Distribusi Kategori dan Visualisasi
```python
# Cek distribusi kategori berita
print("="*50)
print("DISTRIBUSI KATEGORI BERITA")
print("="*50)
kategori_dist = df['kategori'].value_counts()
print(kategori_dist)

# Visualisasi distribusi kategori
plt.figure(figsize=(10, 6))
kategori_dist.plot(kind='bar', color='skyblue', edgecolor='black')
plt.title('Distribusi Kategori Berita', fontsize=14, fontweight='bold')
plt.xlabel('Kategori', fontsize=12)
plt.ylabel('Jumlah Berita', fontsize=12)
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y', alpha=0.3)
plt.tight_layout()
plt.show()

print(f"\nJumlah kategori: {df['kategori'].nunique()}")
```

Output: tampilkan tabel distribusi dan plot setelah menjalankan sel (tergantung `Berita.csv`).

---

## 5. Persiapan Data untuk Klasifikasi
Menggunakan kolom teks (mis. `berita` atau `stemming_indo`) sebagai corpus dan `kategori` sebagai label.

```python
# Ambil corpus dan label
corpus = df['berita'].astype(str).tolist()
labels = df['kategori'].tolist()

print("="*50)
print("PERSIAPAN DATA")
print("="*50)
print(f"Jumlah dokumen: {len(corpus)}")
print(f"Jumlah label: {len(labels)}")
print(f"\nContoh dokumen pertama:")
print(corpus[0][:200] + "...")
print(f"\nLabel pertama: {labels[0]}")
```

Output: jalankan sel untuk melihat ringkasan data.

---

## 6. Split Train/Test (80:20)
```python
X_train_text, X_test_text, y_train, y_test = train_test_split(
    corpus,
    labels,
    test_size=0.2,
    random_state=42,
    stratify=labels
)

print("="*50)
print("PEMBAGIAN DATA TRAINING DAN TESTING")
print("="*50)
print(f"Jumlah data training: {len(X_train_text)}")
print(f"Jumlah data testing: {len(X_test_text)}")
print(f"\nProporsi pembagian: {len(X_train_text)/len(corpus)*100:.1f}% training, {len(X_test_text)/len(corpus)*100:.1f}% testing")

# Cek distribusi kategori di data training dan testing
print("\n" + "="*50)
print("DISTRIBUSI KATEGORI DI DATA TRAINING")
print("="*50)
print(pd.Series(y_train).value_counts())

print("\n" + "="*50)
print("DISTRIBUSI KATEGORI DI DATA TESTING")
print("="*50)
print(pd.Series(y_test).value_counts())
```

Output: jalankan untuk melihat jumlah train/test dan distribusi kategori.

---

## 7. Ekstraksi Fitur Menggunakan LDA (via CountVectorizer + sklearn LDA)

### 7.1 Membuat Vocabulary dengan CountVectorizer
```python
from sklearn.feature_extraction.text import CountVectorizer

vectorizer = CountVectorizer(
    min_df=5,
    max_df=0.8,
    max_features=5000
)

# Fit dan transform data training
X_train_counts = vectorizer.fit_transform(X_train_text)

# Transform data testing
X_test_counts = vectorizer.transform(X_test_text)

print("="*50)
print("VOCABULARY INFORMATION")
print("="*50)
print(f"Ukuran vocabulary: {len(vectorizer.vocabulary_)}")
print(f"Bentuk matrix training: {X_train_counts.shape}")
print(f"Bentuk matrix testing: {X_test_counts.shape}")
print(f"\nContoh 10 kata pertama dalam vocabulary:")
print(list(vectorizer.vocabulary_.keys())[:10])
```

Output: jalankan sel untuk melihat ukuran vocabulary dan bentuk matriks sparse.

### 7.2 Menentukan Jumlah Topik Optimal (Perplexity)
```python
def compute_perplexity(lda_model, data):
    return lda_model.perplexity(data)

topic_range = [5, 10, 15, 20, 25, 30]
perplexity_scores = []

print("="*50)
print("MENCARI JUMLAH TOPIK OPTIMAL")
print("="*50)

for n_topics in topic_range:
    print(f"Mencoba k={n_topics} topik...")
    lda = LatentDirichletAllocation(
        n_components=n_topics,
        random_state=42,
        max_iter=150,
        learning_method='batch',
        n_jobs=-1
    )
    lda.fit(X_train_counts)
    perplexity = compute_perplexity(lda, X_train_counts)
    perplexity_scores.append(perplexity)
    print(f"  Perplexity: {perplexity:.2f}")

print("\n✓ Selesai menghitung perplexity untuk berbagai jumlah topik")
```

Visualisasikan dan pilih optimal topics (jalankan sel berikut):
```python
plt.figure(figsize=(10, 6))
plt.plot(topic_range, perplexity_scores, marker='o', linewidth=2, markersize=8)
plt.xlabel('Jumlah Topik (k)', fontsize=12)
plt.ylabel('Perplexity Score', fontsize=12)
plt.title('Perplexity Score vs Jumlah Topik', fontsize=14, fontweight='bold')
plt.grid(True, alpha=0.3)
plt.xticks(topic_range)
plt.tight_layout()
plt.show()

optimal_topics = topic_range[np.argmin(perplexity_scores)]
print(f"\n✓ Jumlah topik optimal (berdasarkan perplexity): {optimal_topics}")
```

---

### 7.3 Melatih Model LDA Final
```python
print("="*50)
print(f"MELATIH MODEL LDA DENGAN {optimal_topics} TOPIK")
print("="*50)
lda_model = LatentDirichletAllocation(
    n_components=optimal_topics,
    random_state=0,
    max_iter=1000,
    learning_method='batch',
    n_jobs=-1,
    verbose=1
)
lda_model.fit(X_train_counts)
print(f"\n✓ Model LDA berhasil dilatih dengan {optimal_topics} topik")
```

### 7.4 Tampilkan Top Words per Topik
```python
def display_topics(model, feature_names, n_top_words=10):
    topics = []
    for topic_idx, topic in enumerate(model.components_):
        top_words_idx = topic.argsort()[-n_top_words:][::-1]
        top_words = [feature_names[i] for i in top_words_idx]
        topics.append((topic_idx, top_words))
    return topics

feature_names = vectorizer.get_feature_names_out()
print("="*50)
print("TOP 10 KATA PER TOPIK")
print("="*50)
topics = display_topics(lda_model, feature_names, n_top_words=10)
for topic_idx, top_words in topics:
    print(f"\nTopik {topic_idx}:")
    print(", ".join(top_words))
```

---

## 8. Ekstraksi Fitur: Transformasi Dokumen ke Distribusi Topik (θ)
```python
X_train_lda = lda_model.transform(X_train_counts)
X_test_lda = lda_model.transform(X_test_counts)

print("="*50)
print("EKSTRAKSI FITUR LDA")
print("="*50)
print(f"Bentuk fitur LDA training: {X_train_lda.shape}")
print(f"Bentuk fitur LDA testing: {X_test_lda.shape}")
print(f"\nSetiap dokumen kini direpresentasikan sebagai vektor dengan {optimal_topics} dimensi")
print("(probabilitas dokumen terkait dengan setiap topik)")

print(f"\n" + "="*50)
print("CONTOH DISTRIBUSI TOPIK DOKUMEN PERTAMA")
print("="*50)
for i, prob in enumerate(X_train_lda[0]):
    print(f"Topik {i}: {prob:.4f}")

print(f"\nTotal probabilitas: {X_train_lda[0].sum():.4f} (harus = 1.0)")
```

---

## 9. Pelatihan Model Klasifikasi (SVM)
```python
print("="*50)
print("MELATIH MODEL SVM DENGAN FITUR LDA")
print("="*50)

svm_lda = SVC(kernel='rbf', random_state=42, verbose=True)
svm_lda.fit(X_train_lda, y_train)

print("\n✓ Model SVM berhasil dilatih dengan fitur LDA")
```

Prediksi dan evaluasi (jalankan sel berikut):
```python
y_pred_lda = svm_lda.predict(X_test_lda)

print("="*50)
print("PREDIKSI SELESAI")
print("="*50)
print(f"Jumlah prediksi: {len(y_pred_lda)}")
print(f"\nContoh 10 prediksi pertama:")
for i in range(min(10, len(y_pred_lda))):
    print(f"Aktual: {y_test[i]:15} | Prediksi: {y_pred_lda[i]}")

print("\n" + "="*50)
print("CLASSIFICATION REPORT (LDA + SVM)")
print("="*50)
print(classification_report(y_test, y_pred_lda, zero_division=0))

cm_lda = confusion_matrix(y_test, y_pred_lda)
plt.figure(figsize=(10, 8))
sns.heatmap(cm_lda, annot=True, fmt='d', cmap='Blues',
            xticklabels=sorted(set(y_test)),
            yticklabels=sorted(set(y_test)))
plt.title('Confusion Matrix (LDA + SVM)', fontsize=14, fontweight='bold')
plt.ylabel('Aktual', fontsize=12)
plt.xlabel('Prediksi', fontsize=12)
plt.tight_layout()
plt.show()
```

---

## 10. Pelatihan Model Alternatif: Naive Bayes
```python
from sklearn.naive_bayes import MultinomialNB

print("="*50)
print("MELATIH MODEL NAIVE BAYES DENGAN FITUR LDA")
print("="*50)
nb_lda = MultinomialNB()
nb_lda.fit(X_train_lda, y_train)
print("\n✓ Model Naive Bayes berhasil dilatih dengan fitur LDA")

y_pred_nb = nb_lda.predict(X_test_lda)

print("="*50)
print("PREDIKSI NAIVE BAYES SELESAI")
print("="*50)
print(f"Jumlah prediksi: {len(y_pred_nb)}")
print(f"\nContoh 10 prediksi pertama:")
for i in range(min(10, len(y_pred_nb))):
    print(f"Aktual: {y_test[i]:15} | Prediksi: {y_pred_nb[i]}")

print("\n" + "="*50)
print("CLASSIFICATION REPORT (LDA + NAIVE BAYES)")
print("="*50)
print(classification_report(y_test, y_pred_nb, zero_division=0))

cm_nb = confusion_matrix(y_test, y_pred_nb)
plt.figure(figsize=(10, 8))
sns.heatmap(cm_nb, annot=True, fmt='d', cmap='Greens',
            xticklabels=sorted(set(y_test)),
            yticklabels=sorted(set(y_test)))
plt.title('Confusion Matrix (LDA + Naive Bayes)', fontsize=14, fontweight='bold')
plt.ylabel('Aktual', fontsize=12)
plt.xlabel('Prediksi', fontsize=12)
plt.tight_layout()
plt.show()
```

---

## 11. Perbandingan Performa
```python
print("="*70)
print("PERBANDINGAN METRIK EVALUASI")
print("="*70)

accuracy_svm = accuracy_score(y_test, y_pred_lda)
precision_svm = precision_score(y_test, y_pred_lda, average='weighted', zero_division=0)
recall_svm = recall_score(y_test, y_pred_lda, average='weighted', zero_division=0)
f1_svm = f1_score(y_test, y_pred_lda, average='weighted', zero_division=0)

accuracy_nb = accuracy_score(y_test, y_pred_nb)
precision_nb = precision_score(y_test, y_pred_nb, average='weighted', zero_division=0)
recall_nb = recall_score(y_test, y_pred_nb, average='weighted', zero_division=0)
f1_nb = f1_score(y_test, y_pred_nb, average='weighted', zero_division=0)

comparison_df = pd.DataFrame({
    'Metrik': ['Accuracy', 'Precision', 'Recall', 'F1-Score'],
    'LDA + SVM': [accuracy_svm, precision_svm, recall_svm, f1_svm],
    'LDA + Naive Bayes': [accuracy_nb, precision_nb, recall_nb, f1_nb]
})

print(comparison_df.to_string(index=False))
print("="*70)

fig, ax = plt.subplots(figsize=(12, 6))
x = np.arange(len(comparison_df['Metrik']))
width = 0.35

bars1 = ax.bar(x - width/2, comparison_df['LDA + SVM'], width,
               label='LDA + SVM', color='steelblue', edgecolor='black')
bars2 = ax.bar(x + width/2, comparison_df['LDA + Naive Bayes'], width,
               label='LDA + Naive Bayes', color='seagreen', edgecolor='black')

ax.set_xlabel('Metrik Evaluasi', fontsize=12, fontweight='bold')
ax.set_ylabel('Nilai', fontsize=12, fontweight='bold')
ax.set_title('Perbandingan Performa: LDA + SVM vs LDA + Naive Bayes',
             fontsize=14, fontweight='bold')
ax.set_xticks(x)
ax.set_xticklabels(comparison_df['Metrik'])
ax.legend(fontsize=10)
ax.set_ylim(0, 1.1)
ax.grid(axis='y', alpha=0.3)

for bars in [bars1, bars2]:
    for bar in bars:
        height = bar.get_height()
        ax.text(bar.get_x() + bar.get_width()/2., height,
                f'{height:.3f}',
                ha='center', va='bottom', fontsize=9)

plt.tight_layout()
plt.show()
```

---

