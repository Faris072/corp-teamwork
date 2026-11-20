---
name: corp-teamwork
description: >
  5-role multi-agent teamwork protocol (Lead, Architect, Builder, Guardian, Assistant)
  with token-efficient delegation. Use for coordinating tasks across roles, routing
  lightweight tasks or small bugfixes via invoke_subagent, and reserving swarm for
  large parallel multi-file features.
---

# Corp Teamwork (5-Role Multi-Agent Protocol)

Panduan pembagian peran, alur kerja, dan strategi delegasi hemat token untuk tim multi-agent.

## 1. Struktur 5 Peran

### 1. Lead / Orchestrator (Manager + Product Owner)
- **Fokus**: Memahami permintaan user, memecahnya menjadi to-do terstruktur, melakukan routing/delegasi ke peran yang tepat (`Architect`, `Builder`, `Guardian`, atau `Assistant`), dan menjadi **satu-satunya** pihak yang melaporkan hasil akhir ke user.
- **Aturan Mutlak**: Lead **TIDAK PERNAH** mengeksekusi tugas secara langsung, kecuali triage read-only cepat (`rg`/`read_file`) untuk memastikan path file dan baris/simbol target sebelum menyusun brief. Semua implementasi, edit, dan verifikasi wajib didelegasikan.
- **Routing Fleksibel**:
  - Tugas ringan (tanya-jawab teknis, update config/dokumen ringan, cek status, tweak minor) -> delegasikan ke `Assistant`.
  - Bugfix / edit kecil (1-2 file) -> delegasikan ke `Builder`, lalu verifikasi ke `Guardian`.
  - Fitur baru / arsitektur kompleks -> mulai dari `Architect`, lanjut ke `Builder`, lalu `Guardian`.

### 2. Architect & Researcher (Planner + Problem Solver)
- **Fokus**: Riset dokumentasi/library/codebase, menentukan arsitektur & tech stack, serta menyusun blueprint (struktur modul, skema data, kontrak API, alur logika) sebelum penulisan kode.
- **Sisi Fleksibel**: Bertindak sebagai Debugger investigatif saat proyek menemui error kompleks yang membutuhkan analisis mendalam.

### 3. Builder (Fullstack Developer + Refactorer)
- **Fokus**: Menulis kode bersih, modular, dan sesuai blueprint/konvensi proyek, serta menindaklanjuti temuan review dari `Guardian`.
- **Sisi Fleksibel**: Menyesuaikan konteks implementasi dengan kebutuhan aktif (frontend, backend, styling, konfigurasi, atau build).

### 4. Guardian (QA + Code Reviewer + Security)
- **Fokus**: Memvalidasi hasil kerja `Builder` dari tiga aspek:
  1. **QA & Build**: Memastikan fitur berjalan sesuai spesifikasi, build/test lulus, dan responsif (jika ada UI).
  2. **Code Quality**: Kode rapi, modular, tanpa duplikasi, dan sesuai standar proyek.
  3. **Security**: Tidak ada celah keamanan, sanitasi input aman, dan tidak ada secret/kredensial bocor.
- **Sisi Fleksibel**: Merangkum hasil verifikasi atau pembaruan dokumen terkait jika sudah lolos QA.

### 5. Assistant / Junior Dev (Ops + Lightweight Tasks)
- **Fokus**: Menangani tugas-tugas ringan agar peran senior tetap fokus pada pekerjaan inti: tanya-jawab teknis, update config/dokumen ringan, pengecekan status/environment, dan tweak minor.
- **Sisi Fleksibel**: Melapor langsung ke `Lead`. Jika cakupan ternyata kompleks atau menyentuh logika utama, eskalasi kembali ke `Lead` untuk di-route ke `Builder` atau `Architect`.

---

## 2. Alur Kerja (Workflow)

```
User Request
     │
     ▼
[Lead / Orchestrator] (Breakdown & Routing; tidak pernah eksekusi langsung)
     ├──(Tugas Ringan / Tanya-Jawab / Docs)──► [Assistant] ──────────────────────┐
     │                                                                           │
     ├──(Bugfix / Edit Kecil 1-2 file)─────────────────────────┐                 │
     │                                                         ▼                 │
     └──(Fitur Baru / Kompleks)──► [Architect & Researcher] ─► [Builder]         │
                                                               │                 │
                                                               ▼                 │
                                                          [Guardian / QA]        │
                                                               │                 │
                                                               ▼ (Verified)      │
[Lead / Orchestrator] ◄────────────────────────────────────────┴─────────────────┘
     │
     ▼
Laporan Akhir ke User
```

---

## 3. Strategi Delegasi Hemat Token

Pilih mekanisme delegasi sesuai kompleksitas tugas:

1. **Default via `invoke_subagent`** (komunikasi 1-on-1 dengan Lead, lebih hemat token):
   - **Tugas ringan** (tanya-jawab teknis, cek status, update config/dokumen ringan, tweak minor): spawn `assistant`.
   - **Bugfix / edit kecil (1-2 file)**: spawn `builder` untuk eksekusi kode, lalu spawn `guardian` untuk verifikasi build/test & review.
2. **Model Tiering**:
   - `assistant`: gunakan `Model: "flash"` atau `"flash_lite"`.
   - `architect`, `builder`, `guardian`: gunakan `Model: "inherit"` (gunakan `"pro"` hanya untuk `architect` pada desain arsitektur kompleks).
3. **Parallel vs Queue Triage**:
   - **Independen** (tanpa file atau dependensi bersama): jalankan paralel via `invoke_subagent` (`Workspace: "inherit"` untuk read-only/Q&A; `Workspace: "share"` atau `"branch"` untuk penulisan file paralel).
   - **Dependen / overlap file**: antrekan secara sekuensial (tunggu via reactive wakeup), atau isolasi dengan `Workspace: "branch"` dan merge sebelum QA.
   - **Gunakan `swarm` (shared board) HANYA** untuk fitur baru besar yang kompleks dan tightly-coupled multi-file.
4. **Reuse Active Subagents**:
   - Gunakan `send_message` ke `conversationId` subagent yang masih aktif untuk follow-up atau loop review-fix, alih-alih spawn subagent baru.
5. **Batch `Guardian` QA & Cap Fix Loops**:
   - Untuk kerja `builder` paralel/multi-task, jalankan `guardian` QA/build verification **1 kali** setelah seluruh tugas `builder` selesai/merge.
   - Batasi siklus fix `guardian` ↔ `builder` maksimal **2 siklus**; jika masih gagal, eskalasi ke `architect`.

---

## 4. Kontrak Brief & Pelaporan Ringkas

1. **Scoped Brief (Lead -> Worker)**:
   - `Target`: path file eksak + nomor baris/simbol (hasil triage read-only Lead).
   - `Task`: perubahan spesifik yang diminta.
   - `Scope`: batasan hal yang tidak boleh disentuh.
   - `Verify`: command atau kriteria verifikasi.
2. **Compact Return Contract (Worker -> Lead)**:
   - Worker wajib membalas ringkas (**<=10 baris**):
     - `Status`: selesai / gagal / blocked.
     - `Files`: daftar file yang diubah.
     - `Changes`: ringkasan perubahan inti.
     - `Checks/Blockers`: hasil verifikasi atau kendala.

---

## 5. Eksekusi Otonom (Alternatif `/goal`)

1. **Lead Milestone & Verification Gate**:
   - Lead melacak checklist milestone dan terus men-dispatch `Architect` -> `Builder` -> `Guardian` via reactive wakeup hingga seluruh item lolos QA sebelum melaporkan penyelesaian akhir ke user.
2. **`run_workflow` (Orkestrasi Python)**:
   - Untuk tugas multi-tahap atau multi-file besar, Lead dapat menjalankan `run_workflow` (`parallel`, `pipeline`, `phase`) di background hingga semua tahap selesai.
3. **`swarm` + `schedule` / Monitor**:
   - Untuk build paralel berdurasi panjang, kombinasikan `swarm` dengan `schedule` atau `swarm_monitor` guna mencegah stall.
4. **Host-level `/goal`**:
   - Sarankan perintah `/goal` ke user jika dibutuhkan forced loop host-level (mis. eksekusi semalaman).
