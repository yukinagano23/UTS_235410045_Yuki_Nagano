# UTS Sistem Terdistribusi dan Terdesentralisasi

Berikut panduan lengkap membuat aplikasi FastAPI (Python) dengan load balancing nginx menggunakan Podman.
___

Stack yang digunakan
1. Framework: FastAPI (bukan Blacksheep)
2. Container engine: Podman
3. Load balancer: nginx
4. Orkestrasi: Podman pod / Podman network
___

### Struktur Proyek
```bash
myapp/
├── app/
│   ├── main.py
│   └── Containerfile
├── nginx/
│   └── nginx.conf
└── README.md
```
___

### Langkah 1 — Buat Aplikasi FastAPI
masuk ke folder myapp, lalu masuk lagi ke folder app, buat file main.py. isikan kode berikut:
```bash
import os
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    hostname = os.environ.get("HOSTNAME", "unknown")
    return {
        "message": "Hello from FastAPI!",
        "instance": hostname
    }

@app.get("/health")
def health():
    return {"status": "ok"}
```
<img width="863" height="356" alt="gambar" src="https://github.com/user-attachments/assets/8010c78a-cad0-4c24-bb2b-26e41ac91fa5" />
catatan: Field instance menampilkan hostname container — berguna untuk membuktikan bahwa nginx mendistribusikan request ke instance berbeda.

---


### Langkah 2 — Buat Containerfile Aplikasi
masuk --> app/Containerfile
```bash
FROM python:3.11-slim

WORKDIR /app

RUN pip install fastapi uvicorn

COPY main.py .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
<img width="790" height="266" alt="gambar" src="https://github.com/user-attachments/assets/394d22f1-a039-453d-8643-60462e687e38" />

---

### Langkah 3 — Konfigurasi nginx
masuk ke --> nginx/nginx.conf
```bash
events {}

http {
    upstream fastapi_cluster {
        server app1:8000;
        server app2:8000;
        server app3:8000;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://fastapi_cluster;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}

```
<img width="779" height="406" alt="gambar" src="https://github.com/user-attachments/assets/b5895ee6-88b6-4b12-9529-4e95f08198a3" />
catatan: Blok upstream mendaftarkan 3 instance aplikasi. nginx menggunakan round-robin secara default setiap request diteruskan ke instance berikutnya secara bergantian.

---

### Langkah 4 — Build Image Aplikasi
```bash
cd myapp/app
podman build -t fastapi-app .
```
<img width="963" height="421" alt="gambar" src="https://github.com/user-attachments/assets/1733d0b7-a44a-462d-b438-1e024cd8f600" />
<img width="957" height="341" alt="gambar" src="https://github.com/user-attachments/assets/040030e6-174a-4353-b97a-50bdbb168610" />

---

### Langkah 5 — Buat Podman Network
```bash
podman network create myapp-net
```
<img width="961" height="77" alt="gambar" src="https://github.com/user-attachments/assets/62fb4ec3-b06e-4407-aaa3-e44c2259603e" />
catatan: Semua container (app1, app2, app3, nginx) harus berada dalam network yang sama agar bisa saling berkomunikasi via nama container.

---

### Langkah 6 — Jalankan Instance Aplikasi
```bash
podman run -d --name app1 --network myapp-net fastapi-app
podman run -d --name app2 --network myapp-net fastapi-app
podman run -d --name app3 --network myapp-net fastapi-app
```
<img width="874" height="180" alt="gambar" src="https://github.com/user-attachments/assets/0e585c56-e018-4a35-9e60-f6f70ecacd81" />

---

### Langkah 7 — Jalankan nginx
```bash
podman run -d --name nginx-lb --network myapp-net -p 8080:80 -v "C:/Users/Thinkpad X270/myapp/nginx/nginx.conf:/etc/nginx/nginx.conf:ro" nginx:alpine
```
<img width="961" height="293" alt="gambar" src="https://github.com/user-attachments/assets/e3457d96-453e-4616-b187-dde9a78e7448" />

---

### Langkah 8 — Verifikasi Load Balancing
```bash
# Kirim beberapa request dan perhatikan field "instance"
curl http://localhost:8080/
curl http://localhost:8080/
curl http://localhost:8080/
```
<img width="961" height="122" alt="gambar" src="https://github.com/user-attachments/assets/2046c544-406c-4c76-adfa-0823c242ce74" />




