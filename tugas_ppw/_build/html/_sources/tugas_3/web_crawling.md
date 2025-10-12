# Web Crawling - Tugas 3

Program ini melakukan tiga jenis web crawling:
1. Crawling data PTA Trunojoyo untuk mendapatkan abstrak bahasa Indonesia dan Inggris
2. Crawling berita dari CNN Indonesia untuk mendapatkan ID, kategori, judul, dan isi berita
3. Crawling link keluar dari website PTA Trunojoyo

## 1. Menambahkan abstrak bahasa inggris dari PTA

### Install Library
```python
!pip install builtwith
```

output:
```
Requirement already satisfied: builtwith in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (1.3.4)
Requirement already satisfied: six in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (from builtwith) (1.17.0)
```

### Install dan Import Library untuk Web Scraping
```python
!pip install requests
!pip install beautifulsoup4
import requests
from bs4 import BeautifulSoup
import pandas as pd
```

output:
```
Requirement already satisfied: requests in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (2.32.5)
Requirement already satisfied: beautifulsoup4 in c:\users\hp\appdata\local\programs\python\python311\lib\site-packages (4.13.4)
```

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

## 2. Menampilkan ID berita, judul berita, kategori berita, dan isi berita

### Disable SSL Warning
```python
import urllib3

# Nonaktifkan SSL warning
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
```

### Crawling Berita CNN Indonesia
```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import re

def crawl_cnn_berita():
    data = {"idberita": [], "kategori_berita": [], "judul_berita": [], "isi_berita": []}
    
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/58.0.3029.110 Safari/537.3'
    }
    
    try:
        # Ambil halaman utama CNN Indonesia
        url = "https://www.cnnindonesia.com/"
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        soup = BeautifulSoup(response.content, "html.parser")
        
        # Cari semua artikel di halaman utama
        articles = soup.find_all('article')
        processed_urls = set()
        
        for article in articles:
            link_tag = article.find('a', href=True)
            if not link_tag:
                continue
                
            berita_url = link_tag['href']
            if not berita_url.startswith('https://www.cnnindonesia.com/'):
                berita_url = 'https://www.cnnindonesia.com' + berita_url
            
            # Cek duplikasi
            if berita_url in processed_urls:
                continue
            processed_urls.add(berita_url)
            
            try:
                # Ambil detail berita
                berita_response = requests.get(berita_url, headers=headers, verify=False)
                berita_response.raise_for_status()
                berita_soup = BeautifulSoup(berita_response.content, "html.parser")
                
                # ID berita dari URL
                match = re.search(r'/([0-9]+)$', berita_url)
                idberita = match.group(1) if match else berita_url.split('/')[-1]
                
                # Judul berita
                judul_tag = berita_soup.find('h1')
                judul_berita = judul_tag.get_text(strip=True) if judul_tag else ''
                
                # Kategori berita
                kategori_tag = berita_soup.find('div', class_='breadcrumb')
                kategori_berita = ''
                if kategori_tag:
                    kategori_links = kategori_tag.find_all('a')
                    if kategori_links:
                        kategori_berita = kategori_links[-1].get_text(strip=True)
                
                # Isi berita dengan berbagai selector
                isi_berita = ''
                selectors = [
                    'div.detail_text',
                    'div.detail__body-text', 
                    'div.detail-text',
                    'div[class*="detail"]',
                    'div[class*="content"]',
                    'div[class*="body"]',
                    'article div p',
                    '.article-content p',
                    '.post-content p'
                ]
                
                for selector in selectors:
                    isi_tag = berita_soup.select(selector)
                    if isi_tag:
                        if selector.endswith(' p'):
                            isi_berita = ' '.join([p.get_text(strip=True) for p in isi_tag])
                        else:
                            isi_berita = isi_tag[0].get_text(separator=' ', strip=True)
                        break
                
                # Jika masih kosong, ambil semua paragraf
                if not isi_berita:
                    paragraphs = berita_soup.find_all('p')
                    if paragraphs:
                        long_paragraphs = [p.get_text(strip=True) for p in paragraphs if len(p.get_text(strip=True)) > 50]
                        isi_berita = ' '.join(long_paragraphs[:5])  
                        
                data["idberita"].append(idberita)
                data["kategori_berita"].append(kategori_berita)
                data["judul_berita"].append(judul_berita)
                data["isi_berita"].append(isi_berita)
                
            except Exception as e:
                print(f'Gagal mengambil berita: {berita_url} | Error: {e}')
    
    except Exception as e:
        print(f'Terjadi kesalahan: {e}')
    
    df = pd.DataFrame(data)
    df.to_csv("cnn_berita.csv", index=False)
    return df

# jalankan scraping
crawl_cnn_berita()
```

Data berita CNN berhasil di-crawl dan disimpan dalam file CSV dengan kolom ID berita, kategori, judul, dan isi berita.

### Crawling Artikel Spesifik CNN
```python
def crawl_cnn_article(url):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/58.0.3029.110 Safari/537.3'
    }
    
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        soup = BeautifulSoup(response.content, 'html.parser')
        
        # Ambil judul, kategori, dan isi berita
        judul_tag = soup.find('h1')
        judul_berita = judul_tag.get_text(strip=True) if judul_tag else 'Judul tidak ditemukan'
        
        kategori_berita = 'Kategori tidak ditemukan'
        breadcrumb = soup.find('div', class_='breadcrumb')
        if breadcrumb:
            kategori_links = breadcrumb.find_all('a')
            if kategori_links:
                kategori_berita = kategori_links[-1].get_text(strip=True)
        
        # Ambil isi berita
        isi_berita = ''
        selectors = [
            'div.detail_text',
            'div.detail__body-text', 
            'div.detail-text'
        ]
        
        for selector in selectors:
            content_div = soup.select_one(selector)
            if content_div:
                paragraphs = content_div.find_all('p')
                if paragraphs:
                    isi_berita = ' '.join([p.get_text(strip=True) for p in paragraphs])
                break
        
        print("=" * 80)
        print("HASIL CRAWLING BERITA CNN INDONESIA")
        print("=" * 80)
        print(f"Judul Berita: {judul_berita}")
        print(f"Kategori: {kategori_berita}")
        print("-" * 80)
        print("Isi Berita:")
        print(isi_berita[:500] + "..." if len(isi_berita) > 500 else isi_berita)
        print("=" * 80)
        
        return {
            'judul': judul_berita,
            'kategori': kategori_berita,
            'isi': isi_berita
        }
        
    except requests.exceptions.RequestException as e:
        print(f"Terjadi kesalahan saat mengakses {url}: {e}")
        return None

# penggunaan
url_berita = "https://www.cnnindonesia.com/olahraga/20250907140830-156-1270914/start-dari-belakang-bagnaia-ngarep-finis-di-10-besar-motogp-catalunya"
result = crawl_cnn_article(url_berita)
```

output:
```
================================================================================
HASIL CRAWLING BERITA CNN INDONESIA
================================================================================
Judul Berita: Start dari Belakang, Bagnaia Ngarep Finis di 10 Besar MotoGP Catalunya
Kategori: Moto GP
--------------------------------------------------------------------------------
Isi Berita:
Pembalap DucatiFrancesco Bagnaiaberharap finis di 10 besarMotoGP Catalunya 2025di Sirkuit Catalunya saat start dari posisi ke-21. Start ke-21 dalam MotoGP Catalunya 2025 jadi yang terburuk bagi Bagnaia setelah MotoGP Portugal 2022...
================================================================================
```

## 3. Crawl link keluar dari sebuah web

### Import Library untuk Crawling Link
```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
from urllib.parse import urljoin, urlparse
```

### Inisialisasi Variabel Global
```python
# Variabel global
visited = set()
web_structur = []
```

### Fungsi Crawling Website
```python
def crawl_website(url, base_domain, max_depth=2, depth=0):
    if url in visited or depth > max_depth:
        return
    visited.add(url)

    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        soup = BeautifulSoup(response.content, 'html.parser')

        # Ambil semua link
        links = soup.find_all('a', href=True)
        for link in links:
            full_url = urljoin(url, link['href'])  # absolute URL

            web_structur.append({
                "page": url,
                "link keluar": full_url
            })

            # Batasi hanya ke domain pta.trunojoyo.ac.id
            if base_domain in urlparse(full_url).netloc:
                crawl_website(full_url, base_domain, max_depth, depth+1)

    except Exception as e:
        print(f"Error saat akses {url}: {e}")

def run_crawler(start_url, max_depth=2):
    base_domain = urlparse(start_url).netloc
    crawl_website(start_url, base_domain, max_depth=max_depth)
```

### Menjalankan Crawler
```python
start_url = "https://pta.trunojoyo.ac.id"
run_crawler(start_url, max_depth=1)

print(f"Jumlah data hasil crawl: {len(web_structur)}")
```

output:
```
Error saat akses https://pta.trunojoyo.ac.id/index.html: 404 Client Error: Not Found for url: https://pta.trunojoyo.ac.id/index.html
Jumlah data hasil crawl: 5249
```

### Menyimpan Hasil Crawling
```python
df = pd.DataFrame(web_structur)
df.to_csv("hasil_crawl.csv", index=False, encoding="utf-8")

print("Hasil crawling disimpan ke 'hasil_crawl.csv'")
df.head(10)
```

output:
```
Hasil crawling disimpan ke 'hasil_crawl.csv'
```

| | page | link keluar |
|---|---|---|
| 0 | https://pta.trunojoyo.ac.id | https://pta.trunojoyo.ac.id/index.html |
| 1 | https://pta.trunojoyo.ac.id | https://pta.trunojoyo.ac.id |
| 2 | https://pta.trunojoyo.ac.id | https://pta.trunojoyo.ac.id/ |
| 3 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/index.html |
| 4 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/ |
| 5 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/ |
| 6 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/c_search/ |
| 7 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/c_template/ |
| 8 | https://pta.trunojoyo.ac.id/ | https://library.trunojoyo.ac.id/detil.php?id=23 |
| 9 | https://pta.trunojoyo.ac.id/ | https://pta.trunojoyo.ac.id/c_contact/ |

## Ringkasan

Program berhasil melakukan tiga jenis web crawling:
1. **Crawling PTA**: Mengambil data skripsi dengan abstrak bahasa Indonesia dan Inggris
2. **Crawling CNN**: Mengambil berita dengan ID, kategori, judul, dan isi lengkap
3. **Crawling Link**: Mengidentifikasi semua link keluar dari website PTA dengan total 5,249 link
