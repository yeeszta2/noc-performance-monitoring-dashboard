# noc-performance-monitoring-dashboard

Repository tugas mingguan **Performance Monitoring Dashboard Design** (Cloud & Infrastructure Operations, Monitoring Governance)
Isinya adalah implementasi *dashboard-as-code* dari laporan rancangan: dashboard Grafana, aturan Prometheus, routing Alertmanager, dan runbook penanganan

## Ringkasan rancangan
Berikut Aspek dan Keputusan yang diambil:
- Platform : Grafana + Prometheus + Alertmanager (metrik server dari node_exporter)
- Layout : 3 baris, grid 24 kolom, 11 panel: **Top** status makro, **Middle** resource utama (USE), **Bottom** detail investigasi
- Metodologi : USE (Utilization, Saturation, Errors) untuk CPU, memori, storage, dan jaringan
- Refresh dan rentang waktu : Scrape 15 detik, dashboard refresh 15 detik, default 1 jam terakhir
- Threshold : Hijau, kuning, merah per metrik (tabel di bawah)
- Alarm : Berbasis durasi (`for: 3m` atau `5m`) agar lonjakan sesaat tidak membangunkan On-Call
- Notifikasi : Warning ke chat tim, Critical ke chat dan paging On-Call, eskalasi diatur di tool paging
- Runbook : Setiap panel kritis punya tautan ke runbook di folder `runbooks/` |


## Struktur repository

```
.
├── docker-compose.yml                     # lab: Prometheus, Alertmanager, Grafana, node_exporter
├── prometheus/
│   ├── prometheus.yml                     # scrape 15s, rule files, Alertmanager
│   └── rules/
│       ├── recording.yml                  # query USE yang dihitung sekali (ringan untuk dashboard)
│       └── alerts.yml                     # alert Warning/Critical + alert gabungan StorageIOSaturation
├── alertmanager/alertmanager.yml          # routing notifikasi + inhibit rule
├── grafana/
│   ├── provisioning/                      # datasource dan dashboard otomatis ter-load
│   └── dashboards/noc-performance-dashboard.json
├── runbooks/                              # RB-01 sampai RB-05
├── scripts/
│   ├── build_dashboard.py                 # generator JSON dashboard (dashboard-as-code)
│   ├── validate.py                        # cek konsistensi repo
│   └── simulate-disk-load.sh              # uji alarm dengan beban disk (hanya di lab)
├── tests/alerts_test.yml                  # unit test aturan alert (promtool)
└── workflows/validate.yml                 # CI: validasi otomatis
```


Sebelum dipakai, ganti `OWNER` pada URL runbook (di `prometheus/rules/alerts.yml` dan `scripts/build_dashboard.py`) dengan username GitHub, lalu jalankan `python3 scripts/build_dashboard.py` untuk membuat ulang JSON dashboard. URL webhook di `alertmanager/alertmanager.yml` juga masih placeholder

Panel Error Rate dan Response Time membutuhkan aplikasi yang mengekspos `http_requests_total` dan `http_request_duration_seconds_bucket`. Aktifkan job `app` di `prometheus/prometheus.yml`. Tanpa itu panel menampilkan "No data" dan alert `ErrorRateCritical` tidak aktif

## Pengujian

```bash
pip install pyyaml
python3 scripts/validate.py                        # struktur dashboard, YAML, tautan runbook

promtool check rules prometheus/rules/*.yml        # sintaks aturan Prometheus
promtool test rules tests/alerts_test.yml          # unit test alert
amtool check-config alertmanager/alertmanager.yml  # sintaks Alertmanager
```

Unit test membuktikan tiga hal: beban berkelanjutan memicu alert setelah durasi `for`, lonjakan 1 menit **tidak** memicu alert, dan CPU tinggi saja **tidak** memicu alert `StorageIOSaturation`. Untuk menguji warna panel dan alarm secara langsung, jalankan `scripts/simulate-disk-load.sh` di mesin lab (butuh `fio`)

### Status verifikasi
- Sudah dijalankan: `scripts/validate.py`, `promtool check rules`, `promtool check config`, `promtool test rules`, dan `amtool check-config` (Prometheus 2.55.1, Alertmanager 0.27.0).
- Belum diuji di Grafana yang berjalan: tampilan JSON dashboard (terutama panel tabel dan data link). Impor atau jalankan lab lalu sesuaikan bila ada panel yang berbeda.

