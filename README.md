# Ansible Hermes Agent Installer

Ansible role untuk instalasi otomatis Hermes Agent (Telegram AI gateway) pada server Linux/macOS venv terisolasi, konfigurasi provider AI, dan systemd service, semuanya idempotent.

Role ini dibuat untuk menggantikan proses install manual (copy-paste script bash + isi form interaktif satu-satu) dengan satu playbook yang bisa dijalankan ulang tanpa efek samping, dan aman disimpan di Git (secrets tidak pernah ke commit).

## Supported OS

Role ini sudah diuji pada:

| OS | Status | Catatan |
| --- | --- | --- |
| Ubuntu 22.04 / 24.04 | via `apt` |
| Debian 12 | via `apt` |
| Rocky Linux 9 | via `dnf` |
| macOS (Apple Silicon & Intel) | via Homebrew Homebrew harus sudah terinstall lebih dulu |

> Catatan: Windows tidak didukung role ini. Ansible mengelola Windows lewat WinRM dengan cara yang cukup berbeda dari SSH-based Linux/macOS.

## Main Features

- Auto-detect OS family (`ansible_os_family`) dan pilih package manager yang sesuai (apt/dnf/brew).
- Install Python 3.11+ kalau belum ada, dan gagal cepat dengan pesan jelas kalau versi kurang.
- Install `hermes-agent` ke **virtualenv terisolasi** menghindari error `externally-managed-environment` di Ubuntu 23.04+/Debian 12+.
- Render `~/.hermes/.env` dari template Jinja2 (bukan `>>` append) aman di-run ulang, tidak duplikat, permission `0600`.
- Setup provider AI: DeepSeek, OpenRouter, atau Ollama (pilih salah satu).
- Untuk Ollama: install binary + pull model secara idempotent (cek dulu sebelum install/pull ulang).
- Setup systemd service untuk gateway Telegram (auto-restart, jalan terus walau SSH ditutup) khusus Linux.
- Assert di awal play: gagal cepat kalau token/secrets belum diisi, daripada gagal di tengah proses.

## Role Structure

```
ansible-hermes-agent/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install_python.yml
│   ├── install_hermes.yml
│   ├── configure_env.yml
│   ├── setup_provider.yml
│   └── setup_service.yml
├── templates/
│   ├── env.j2
│   └── hermes-agent.service.j2
├── group_vars/
│   └── all.yml.example
├── .gitignore
├── ansible.cfg
├── inventory.example.ini
├── site.yml
├── README.md
└── LICENSE
```

## Requirements

Control node:

- Ansible 2.14+
- Collection `community.general` (untuk modul Homebrew di macOS):
  ```
  ansible-galaxy collection install community.general
  ```

Target server:

- Akses SSH (key-based direkomendasikan), user dengan privilege escalation (sudo) untuk Linux
- Untuk macOS: Homebrew sudah terinstall (installer Homebrew butuh prompt interaktif jadi sengaja tidak diotomatisasi role ini)
- Untuk provider `ollama`: RAM minimal ~8GB untuk model default `llama3.2:3b`
- Bot Telegram sudah dibuat via @BotFather, dan User ID sudah didapat via @userinfobot

## Required Variables

Contoh variable minimal di `group_vars/all.yml` (copy dari `group_vars/all.yml.example`):

```yaml
telegram_bot_token: "1234567890:ABCdefGHIjklmNOPqrstUVwxyz"
telegram_allowed_users: "6902977355"
hermes_provider: "deepseek"
deepseek_api_key: "sk-xxxxxxxx"
```

Penjelasan:

| Variable | Description |
| --- | --- |
| `telegram_bot_token` | Token dari @BotFather |
| `telegram_allowed_users` | User ID Telegram kamu, dari @userinfobot |
| `hermes_provider` | `deepseek`, `openrouter`, atau `ollama` |
| `deepseek_api_key` | Wajib diisi kalau `hermes_provider: deepseek` |
| `openrouter_api_key` | Wajib diisi kalau `hermes_provider: openrouter` |
| `hermes_ollama_model` | Model Ollama, default `llama3.2:3b` |
| `hermes_service_enabled` | `true`/`false` setup systemd service (Linux) |

## Inventory Example

Contoh `inventory.ini` (copy dari `inventory.example.ini`):

```ini
[hermes_servers]
myserver ansible_host=192.168.1.10 ansible_user=ubuntu
```

## Playbook Example

`site.yml` sudah jadi entry point langsung tidak perlu file playbook tambahan:

```yaml
---
- name: Install and configure Hermes Agent
  hosts: hermes_servers
  tasks:
    - name: Run hermes-agent install tasks
      ansible.builtin.import_tasks: tasks/main.yml
```

## Usage

Cek syntax:

```bash
ansible-playbook -i inventory.ini site.yml --syntax-check
```

Jalankan (secrets di `group_vars/all.yml` biasa):

```bash
ansible-playbook -i inventory.ini site.yml
```

Jalankan dengan Ansible Vault (direkomendasikan untuk tim/production):

```bash
ansible-vault encrypt group_vars/all.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

## Post Install Check

Login ke server target:

```bash
sudo systemctl status hermes-agent
sudo journalctl -u hermes-agent -n 50
```

Buka Telegram, cari bot kamu, kirim `Halo` kalau dibalas, instalasi berhasil.

## Notes

- Role ini ditujukan untuk single-server deployment.
- `.env` dan secrets lain tidak pernah masuk Git — dicek lewat `.gitignore` dan `assert` di awal `site.yml`.
- Ubah `hermes_provider` kapan saja lalu re-run playbook — `.env` akan di-render ulang otomatis dan service di-restart.
- Untuk environment production, tambahkan firewall, monitoring, dan backup `~/.hermes/.env` secara terpisah.

## License

MIT License. Lihat file [LICENSE](LICENSE).
