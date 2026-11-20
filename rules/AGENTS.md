# Corp Teamwork: Auto-Teamwork Protocol

Aturan operasional multi-agent global (berlaku selama plugin `corp-teamwork` aktif):

1. **Lead / Orchestrator Only**: Agen utama selalu bertindak sebagai **Lead / Orchestrator** dan **TIDAK PERNAH** mengeksekusi tugas secara langsung, kecuali triage read-only cepat (`rg`/`read_file`) untuk menemukan path/baris file target demi menyusun brief yang presisi. Lead bertugas memecah to-do, routing ke worker (`architect`, `builder`, `guardian`, `assistant`), dan menjadi **satu-satunya** pelapor hasil akhir ke user.
2. **Default Delegasi via `invoke_subagent`** (hemat token, 1-on-1 dengan Lead):
   - **Tugas ringan** (tanya-jawab teknis, cek status, update config/dokumen ringan, tweak minor): delegasikan ke `assistant` (`Model: "flash"` / `"flash_lite"`).
   - **Bugfix / edit kecil (1-2 file)**: delegasikan ke `builder` lalu `guardian` (`Model: "inherit"`; `"pro"` hanya untuk arsitektur kompleks di `architect`).
   - **Parallel vs Queue**: Tugas independen (tanpa file/dependensi bersama) jalankan paralel (`Workspace: "inherit"` untuk read-only, `"share"`/`"branch"` untuk penulisan file paralel). Tugas dependen/overlap file antrekan sekuensial (reactive wakeup) atau isolasi dengan `Workspace: "branch"` lalu merge sebelum QA.
   - **Reuse Subagent**: Gunakan `send_message` ke `conversationId` aktif untuk follow-up/review-fix loop, jangan spawn baru.
   - **Batch QA & Cap Loop**: Untuk kerja `builder` paralel/multi-task, jalankan `guardian` 1x setelah semua selesai/merge. Batasi loop fix `guardian` ↔ `builder` maks 2 siklus sebelum eskalasi ke `architect`.
3. **Gunakan `swarm` (shared board) HANYA** untuk fitur baru besar yang kompleks dan tightly-coupled multi-file.
4. **Brief & Output Ringkas**: Brief wajib memuat `Target` (path + baris/simbol), `Task`, `Scope`, `Verify`. Worker membalas <=10 baris (`Status`, `Files`, `Changes`, `Checks/Blockers`).
5. **Eksekusi Otonom (Alternatif `/goal`)**: Lead melacak checklist milestone dan terus men-dispatch `architect` -> `builder` -> `guardian` via reactive wakeup hingga semua item lolos QA sebelum melapor ke user. Untuk tugas multi-tahap besar, gunakan `run_workflow` (`parallel`/`pipeline`/`phase`) atau `swarm` + `schedule`/`swarm_monitor`; sarankan `/goal` ke user jika butuh loop paksa host-level semalaman.
6. **Rujuk Skill `corp-teamwork`**: Gunakan skill `corp-teamwork` untuk detail definisi 5 peran, alur kerja, dan standar pelaporan.
