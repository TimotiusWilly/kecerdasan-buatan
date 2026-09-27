# Laporan Proyek: Asisten Cerdas Berbasis Graph Retrieval-Augmented Generation (GraphRAG) untuk Manajemen Arsip Organisasi

**Kelompok 5**

**Disusun Oleh:**
1. Timotius Willy Narendra (H1D025052)
2. Adridinan Najmi Faza (H1D025059)
3. Fardizza Finda Rahman (H1D025067)
4. Kautsar Rifqi Aditya (H1D025084)
5. Haiban Nur Irvanof (H1D025085)

---

## 1. Masalah (Problem Statement)
Setiap pergantian kepengurusan di lingkungan organisasi mahasiswa atau kampus, sering kali terjadi fenomena hilangnya "memori institusi" (*institutional memory*). Dokumen-dokumen kesejarahan dan operasional penting—seperti notula rapat, Standar Operasional Prosedur (SOP), Laporan Pertanggungjawaban (LPJ), hingga proposal program kerja—sering kali tercecer di berbagai platform penyimpanan terpisah (misalnya Google Drive pribadi pengurus lama, grup WhatsApp, atau media penyimpanan lokal). 

Dampak utama dari masalah ini adalah pengurus yang baru menjabat kesulitan mencari data historis secara spesifik ketika dibutuhkan. Metode pencarian kata kunci tradisional (*keyword search*) yang biasa digunakan tidak mampu menangkap konteks kalimat dan sering kali gagal menjawab pertanyaan yang kompleks. Akibatnya, banyak waktu produktif yang terbuang untuk meriset ulang keputusan, mencari rincian anggaran yang pernah digunakan, atau bahkan mengulang kesalahan yang sama dari kepengurusan tahun sebelumnya. Ketiadaan sistem pengarsipan cerdas yang mampu menjawab pertanyaan analitis secara instan (misal: *"Kenapa tahun lalu vendor konveksi jaket diganti, dan berapa sisa budget proker X?"*) membuat proses regenerasi organisasi menjadi tidak efisien.

## 2. Dataset
Untuk membangun sistem ini dengan aman secara *resource* komputasi namun tetap menjaga privasi data riil mahasiswa, dataset yang akan digunakan difokuskan pada arsip-arsip internal organisasi kampus dengan perlakuan (pra-pemrosesan) khusus.
*   **Jenis Data**: Koleksi dokumen meliputi Laporan Pertanggungjawaban (LPJ), Proposal Program Kerja, Notula Rapat mingguan/bulanan, serta dokumen Standar Operasional Prosedur (SOP) divisi.
*   **Format Data**: Sebagian besar berupa file teks tidak terstruktur (*unstructured text*) dalam bentuk PDF (untuk laporan formal) dan format Markdown atau teks polos (untuk notula rapat).
*   **Volume dan Sumber**: Dataset akan disimulasikan menggunakan data buatan (*dummy*) yang merepresentasikan operasional himpunan mahasiswa, atau menggunakan arsip asli organisasi kampus yang telah melalui proses anonimisasi (*anonymized*). Proses anonimisasi bertujuan menghilangkan Informasi Identifikasi Pribadi (PII) yang sensitif seperti NIM, NIK, alamat, atau nomor telepon. Volume data ditargetkan mencapai puluhan hingga ratusan dokumen yang mencakup rentang waktu 2-3 tahun kepengurusan terakhir.
*   **Karakteristik**: Data kaya akan informasi kronologis dan relasional seperti daftar panitia inti, rancangan anggaran historis, evaluasi paska-kegiatan, dan justifikasi (alasan) di balik keputusan strategis/operasional sebuah acara.

## 3. Rencana Algoritma (Algorithm Plan)
Proyek ini dirancang secara realistis agar dapat diselesaikan dalam 1 semester. Oleh karena itu, pendekatan ini tidak akan melatih (*training*) *Large Language Model* (LLM) dari awal yang menuntut biaya GPU mahal. Sebagai gantinya, sistem akan menggunakan pendekatan arsitektur **Graph Retrieval-Augmented Generation (GraphRAG)** yang dipadukan dengan **Hybrid Search**. Rencana algoritma dipecah ke dalam empat fase utama:

### A. Data Ingestion & Indexing Pipeline
1.  **Parsing & Chunking**: Dokumen mentah (PDF/Markdown) dibaca menggunakan pustaka *PDF Parser* untuk mengekstraksi teks. Teks tersebut kemudian dipecah menjadi unit-unit lebih kecil (*chunks*) menggunakan strategi *semantic chunking*. Strategi semantik ini memastikan bahwa batasan antar potongan teks dibuat berdasarkan makna paragraf, sehingga konteks kalimat tidak terputus di tengah jalan.
2.  **Vector & Graph Construction**: 
    *   Setiap teks *chunk* dikonversi menjadi representasi matematis (*vector embeddings*) menggunakan model *embedding*, kemudian diindeks dan disimpan di dalam *Vector Database* (menggunakan PostgreSQL dengan ekstensi `pgvector`).
    *   Pada saat yang sama, digunakan algoritma (dibantu oleh LLM) untuk mengekstrak **Entitas** (misal: subjek "Proker Malam Keakraban", "Vendor Kaos Y", "Tahun 2023") serta **Relasinya** (misal: "diganti karena harga naik"). Entitas dan relasi ini direpresentasikan sebagai simpul (*nodes*) dan sisi (*edges*) untuk membangun **Knowledge Graph** (menggunakan *engine* SQLite yang diproses melalui pustaka NetworkX).

### B. Proses Pencarian (Retrieval Phase)
Ketika sistem menerima pertanyaan (kueri) dari pengguna menggunakan bahasa natural, algoritma akan menjalankan metode **Hybrid Search** (Pencarian Ganda):
1.  **Vector Search**: Mengukur jarak kedekatan (kemiripan kosinin/semantik) antara kueri pengguna dengan *chunks* dokumen di dalam *vector database*. Pendekatan ini mahir mencari dokumen yang memiliki kesamaan makna (secara implisit) walau kata kunci eksaknya tidak sama persis.
2.  **Graph Retrieval**: Melakukan kueri penelusuran (*traversal*) pada *Knowledge Graph* untuk menelusuri jaringan keterkaitan antar entitas. Pendekatan graf ini mengatasi keterbatasan RAG standar, karena mampu menghubungkan pecahan-pecahan informasi yang secara posisi teks berjauhan di dokumen berbeda, namun secara logika sangat berkaitan.
3.  **Re-ranking**: Kumpulan *chunks* relevan yang diperoleh dari Vektor dan Graf kemudian digabungkan, dihilangkan duplikatnya, lalu diurutkan ulang (*re-ranking*) berdasarkan tingkat relevansi tertingginya terhadap kueri pengguna.

### C. Pembuatan Jawaban (Generation & Source Citation)
Dokumen *chunks* terbaik yang berhasil diretrieve akan dirangkai dan disisipkan bersama pertanyaan pengguna ke dalam sebuah kerangka instruksi (*prompt template*). *Prompt* ini kemudian dikirim ke *engine* LLM (menggunakan mesin lokal Ollama dengan model *open-source* 8B parameter, atau melalui panggilan API *cloud* berbiaya gratis seperti Groq). 

Melalui teknik *prompt engineering* dan *guardrails*, algoritma diinstruksikan secara ketat agar (1) hanya merangkum jawaban berdasarkan konteks dokumen yang diberikan (menghindari halusinasi), dan (2) **wajib** menyertakan sitasi atau *hyperlink* yang merujuk langsung ke *chunk* dokumen arsip sumbernya sebagai alat bukti empiris.

### D. Evaluasi Kinerja (Evaluation Framework)
Akurasi dari arsitektur algoritma ini tidak hanya dinilai secara manual oleh manusia, melainkan diukur secara kuantitatif otomatis menggunakan kerangka kerja evaluasi standar industri bernama **Ragas** (Retrieval Augmented Generation Assessment). Pengujian algoritma difokuskan pada dua metrik matematis utama:
*   **Faithfulness**: Memastikan bahwa keseluruhan klaim dalam jawaban yang dikeluarkan LLM benar-benar dapat dideduksi secara logis dari *chunks* sumber dokumen yang diberikan.
*   **Answer Relevancy**: Menilai penalti jika jawaban LLM melebar atau bertele-tele, sehingga memastikan jawaban yang keluar benar-benar presisi, singkat, dan langsung memecahkan pertanyaan asli pengguna.
