# 🛡️ Roblox Cloud Administration Suite 
> *Advanced Discord Bot Integration for Remote Community Management*

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![Discord.py](https://img.shields.io/badge/discord.py-v2.3.1-5865F2?logo=discord&logoColor=white)
![Roblox API](https://img.shields.io/badge/Roblox-Open_Cloud_API-red?logo=roblox&logoColor=white)

Proyek ini adalah sistem manajemen komunitas jarak jauh (*Remote Administration System*) yang dirancang khusus untuk mengintegrasikan server Discord dengan ekosistem game Roblox. Dibangun menggunakan Python dan `discord.py`, bot ini memanfaatkan **Roblox Open Cloud API** untuk mengelola data pemain secara *real-time* melintasi beberapa *universe* (experience) Roblox secara bersamaan, tanpa mengharuskan staf moderator untuk masuk ke dalam permainan.

> 🔒 **Catatan Privasi:** > *Source code* asli bersifat tertutup (*Private/Proprietary*) untuk melindungi hak kekayaan intelektual klien, struktur database, dan kunci keamanan. Repositori ini berfungsi sebagai representasi arsitektur dan dokumentasi teknis dari sistem yang telah di-*deploy*.

<br>

## 🏗️ Arsitektur Sistem (*System Architecture*)
---
Sistem ini beroperasi sebagai perantara (*middleware*) asinkron antara antarmuka pengguna di Discord dan basis data awan Roblox (*Global DataStores*).

* 📱 **User Interface (Discord):** Memanfaatkan fitur mutakhir *Slash Commands* (App Commands) untuk antarmuka yang intuitif, didukung oleh validasi input *real-time*.
* ⚙️ **Backend Engine (Python):** Memproses logika bisnis, manajemen memori (*caching*), manipulasi format waktu (*Unix Epoch*), dan penanganan *error*.
* 🌐 **External API Communication:** Melakukan *handshaking* yang aman dengan endpoint RESTful Roblox menggunakan header otentikasi `x-api-key`.
* 💾 **Data Persistence (Roblox Datastore):** Menyimpan status moderasi (*ban, kick*) yang akan dieksekusi oleh *script* internal di dalam game Roblox itu sendiri saat pemain mencoba bergabung.

<br>

## 🚀 Fitur Utama & Logika Implementasi (*Key Features*)
---

### 1. Multi-Universe Sync Management
Bot ini tidak terbatas pada satu game. Sistem dikonfigurasi untuk menangani manipulasi Datastore ke berbagai *universe* Roblox secara dinamis.
* **Mekanisme:** Menggunakan struktur data *Dropdown Choices* di Discord (`app_commands.Choice`), moderator dapat memilih *universe* target (misal: DKID FREEDRIVE, NEXTGEN, atau PJL SIMULATOR) dalam satu eksekusi perintah. Bot akan secara otomatis mengarahkan *payload* API ke Universe ID yang relevan.

### 2. Real-Time Player Autocomplete & Memory Caching
Salah satu tantangan terbesar dalam administrasi jarak jauh adalah kesalahan pengetikan (*typo*) nama pengguna.
* **Solusi Inovatif:** Mengimplementasikan sistem *Background Task Caching* (`@tasks.loop`) yang secara asinkron mengambil data daftar pemain aktif dari Roblox Datastore setiap 60 detik.
* **Dampak UX:** Saat moderator mengetikkan nama di Discord, bot menggunakan memori *cache* lokal untuk memberikan saran nama pemain secara instan (*Autocomplete*). Ini meminimalisir pemanggilan API (*Rate Limit Optimization*) dan mempercepat alur kerja moderasi.

### 3. Advanced Moderation Engine (Ban, Kick, Unban)
Modul moderasi dirancang untuk menangani hukuman dengan presisi tingkat tinggi.
* **Resolusi Identitas:** Sebelum hukuman dijatuhkan, bot memanggil *Roblox Users API* untuk mengonversi Username menjadi User ID yang absolut, mencegah penghindaran hukuman akibat pergantian nama profil.
* **Perhitungan Waktu Dinamis:** Modul `rban` menerima parameter satuan waktu (Menit, Jam, Hari, Bulan, Permanen). Sistem mengonversi input ini menjadi *Unix Epoch Timestamp* lalu memposting *payload* JSON ke *GlobalBanList Datastore*.
* **Flag Hukuman Spesifik:** Untuk fitur `rkick`, sistem memanipulasi *expiry time* menjadi sangat singkat (+10 detik) dan menyematkan parameter boolean `"is_kick": True` agar server game langsung mengeluarkan pemain tanpa memasukkannya ke daftar larangan permanen.

### 4. Role-Based Access Control (RBAC) & Security Guardrails
Karena bot memiliki kontrol langsung ke basis data klien, keamanan adalah prioritas mutlak.
* **Verifikasi Guild & Kepemilikan:** Bot secara *hard-coded* mengevaluasi `GUILD_ID` saat perintah dieksekusi. Jika dipanggil dari server yang tidak sah, bot akan menggugurkan proses (*Silent Drop*).
* **Pengecekan Otoritas:** Hanya pengguna dengan Role spesifik (misal: "Staff") atau Pemilik Server yang dapat melewati fungsi predikat otorisasi (`is_authorized_user()`). Akses tanpa izin akan ditolak via *Ephemeral Message*.

<br>

## 🛠️ Tech Stack
---
* **Bahasa Pemrograman:** Python 3.10+
* **Framework Bot:** `discord.py v2.3.1` *(Intents & App Commands Tree)*
* **Integrasi API Eksternal:** * Roblox Open Cloud API *(Standard Datastores v1)*
  * Roblox Web API *(Users v1)*
* **Manajemen & Keamanan:** `python-dotenv` (Enkripsi *Env Variables*), `requests` (HTTP Asinkron), `asyncio` (Event Loop).

<br>

## 🧠 Tantangan & Solusi
---

| Tantangan (*Challenge*) | Solusi yang Diimplementasikan (*Solution*) |
| :--- | :--- |
| **Discord 3-Second Interaction Timeout:** Perintah moderasi seringkali membutuhkan waktu lebih dari 3 detik untuk menyelesaikan proses *fetching* ID dan POST ke basis data Roblox. | Menerapkan metode `interaction.response.defer()` di awal fungsi untuk memberi sinyal "sedang memproses" kepada Discord, mencegah kegagalan timeout, lalu menyelesaikannya dengan `followup.send()`. |
| **Roblox API Rate Limiting:** Memanggil daftar pemain online secara langsung setiap kali command diketik akan memicu pemblokiran API (HTTP 429 Too Many Requests). | Mengembangkan mekanisme *in-memory cache* (`active_players_cache`) yang diperbarui di latar belakang. Permintaan *autocomplete* mengambil data dari memori RAM (*list* Python), bukan dari API internet, menjadikan waktu respons kurang dari 0.1 detik. |
| **Stabilitas Jaringan & Idempotensi:** Potensi kegagalan HTTP saat melakukan POST/DELETE ke Datastore Roblox. | Membangun struktur `try-except` bersarang dengan evaluasi *Status Code* HTTP. Bot dapat membedakan antara *Success* (200/204), *Not Found* (404), dan *Server Error*, lalu memberikan *feedback* spesifik kepada moderator. |

<br>

## 💻 Code Snippets
---
Meskipun repositori ini tidak memuat *source code* secara penuh, berikut adalah beberapa abstraksi dari sistem inti yang menunjukkan bagaimana fitur-fitur utama diimplementasikan:

#### 1. Background Task Caching (Optimasi API Rate Limit)
Daripada melakukan *request* ke API Roblox setiap kali perintah dieksekusi, sistem memperbarui memori lokal setiap 60 detik secara asinkron. Ini menjaga performa bot tetap kilat dan mencegah pemblokiran *Rate Limit*.

```python
from discord.ext import tasks
import requests
import json

# Cache memori lokal untuk menyimpan daftar pemain yang sedang online
active_players_cache = []

@tasks.loop(seconds=60)
async def update_player_cache():
    global active_players_cache
    temp_players = []

    # Iterasi ke setiap universe (game) yang dikelola
    for game in DAFTAR_GAME:
        universe_id = game.value
        url = f"[https://apis.roblox.com/datastores/v1/universes/](https://apis.roblox.com/datastores/v1/universes/){universe_id}/..."
        
        try:
            # Menggunakan timeout untuk mencegah bot freeze jika API Roblox down
            response = requests.get(url, headers=ROBLOX_HEADERS, params=params, timeout=5)
            if response.status_code == 200:
                # Logika parsing JSON...
                temp_players.extend(parsed_data)
        except Exception as e:
            continue # Melanjutkan loop meskipun terjadi kegagalan jaringan
            
    # Membersihkan duplikat menggunakan set()
    active_players_cache = list(set(temp_players))
