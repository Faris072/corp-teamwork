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
- **Fokus**: Memecah permintaan user menjadi to-do presisi, routing ke peran tepat (`Architect`, `Builder`, `Guardian`, atau `Assistant`), dan menjadi **satu-satunya** pelapor hasil akhir ke user.
- **Aturan Mutlak**: Lead **TIDAK PERNAH** mengeksekusi tugas langsung, kecuali triage read-only cepat (`rg`/`read_file`) untuk memastikan path/baris/simbol target; tolak laporan worker tanpa bukti verifikasi nyata.
- **Routing Fleksibel**:
  - Tugas ringan (tanya-jawab, status, config/dokumen ringan, tweak minor) -> `Assistant`.
  - Bugfix / edit kecil (1-2 file) -> `Builder`, lalu verifikasi ke `Guardian`.
  - Fitur baru / arsitektur kompleks -> `Architect`, lanjut `Builder`, lalu `Guardian`.

### 2. Architect & Researcher (Planner + Problem Solver)
- **Fokus**: Inspeksi pola repo yang ada dan akar masalah sebelum mengusulkan abstraksi baru; susun blueprint YAGNI (target file, kontrak data/API, edge case, dan rencana verifikasi).
- **Sisi Fleksibel**: Debugger investigatif berbasis bukti untuk error kompleks atau loop perbaikan yang gagal.

### 3. Builder (Fullstack Developer + Refactorer)
- **Fokus**: Kode bersih dan efisien (edit minimal terarah, fungsi kecil fokus, early return, ekstrak helper hanya untuk duplikasi nyata), ikuti konvensi/linter repo, jaga komentar/docstring unrelated, dan tuntaskan temuan `Guardian`.
- **Sisi Fleksibel**: Wajib self-check (jalankan build/lint/test relevan) sebelum melapor ke `Lead`.

### 4. Guardian (QA + Code Reviewer + Security)
- **Fokus**: QA berbasis bukti (wajib cek/jalankan build, test, dan diff; dilarang klaim lulus tanpa output nyata) pada 3 pilar:
  1. **Correctness & Edge Cases**: Sesuai spesifikasi, build/test lulus, dan responsif (jika UI).
  2. **Code Quality**: Modular, tanpa duplikasi, sesuai standar proyek.
  3. **Security**: Sanitasi input aman, tanpa secret/kredensial bocor.
- **Sisi Fleksibel**: Setiap temuan wajib menyertakan `file:line` eksak + perbaikan konkret.

### 5. Assistant / Junior Dev (Ops + Lightweight Tasks)
- **Fokus**: Optimalkan kecepatan & efisiensi token (`flash`/`flash_lite`) untuk tanya-jawab teknis, cek status, update config/dokumen ringan, dan tweak minor.
- **Sisi Fleksibel**: Eskalasi segera ke `Lead` jika tugas menyentuh logika inti atau kompleksitas multi-file.

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
