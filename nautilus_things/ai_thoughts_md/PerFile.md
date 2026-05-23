# PerFile.md — Файл за файлом

Колонки:

- **Файл** — шлях відносно `/home/denixx/projects/nautilus_things/`.
- **Сигнал** — SIGSEGV / SIGABRT / SIGTRAP / clean exit.
- **Кластер** — A / A′ / B / B′ / B″ / B-jump / C / D / E (див. [`Clusters.md`](Clusters.md)).
- **Лиф** — top stack frame (frame #0 / GLib message).
- **Маркер** — додаткова інформація: вид крешу, версія Nautilus, рендерер.

---

## crashes_dir/crashes_to_nautilus_1/

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_20-11_1.txt` | SIGSEGV | C | `_int_malloc(av=…, bytes=97)` | distro 50.0; heap fallout |
| `crash_2026-04-25_20-13_1.txt` | SIGABRT | C/B | `corrupted size vs. prev_size` у `gtk_label_finalize` ← `gtk_widget_dispose_template` ← `nautilus_grid_cell_dispose:243` | distro 50.0; найчіткіший Cluster B вхід |
| `crash_2026-04-25_20-15_1.txt` | SIGTRAP | A′ | `g_object_unref: assertion '!object_already_finalized' failed`, `_object=0x5555569e5930` | distro 50.0 |

## crashes_dir/old_nautilus_crashes_1/

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_10-42_1.txt` | SIGSEGV | B-jump (gtk#7910 hybrid) | `0x7fff00000000 in ?? ()`, прі CRITICAL `unable to remove uninstalled invalidation notifier: 0x7ffff7d80e20 (0x555556679670)` | distro 50.0; gtk#7910 + jump-to-poison |
| `crash_2026-04-25_10-49_1.txt` | SIGSEGV | B | `gtk_widget_dispose_template` (без debuginfo) | distro 50.0 |
| `crash_2026-04-25_10-54_1.txt` | SIGABRT | C | `__pthread_kill_implementation`, "double free or corruption" | distro 50.0 |
| `nautilus_crashing_1.log` | (journalctl) | E | 3× `kernel: nautilus[…]: segfault at XX_XX00000028 in libwayland-client.so.0.24.0[86ca, +7000]` | distro 50.0; перші зафіксовані креші ще до підключення MALLOC-діагностики |
| `nautilus_crashing_2_full_journal_2_boots.log` | (journalctl) | E | те саме, 3× `libwayland-client.so` | distro 50.0; повний журнал двох ребутів |
| `nautilus_crashing_2_full_journal_2_boots.tar.xz` | – | – | архівна копія попереднього | – |

## crashes_dir/old_nautilus_crashes_2/

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_17-56_1.txt` | SIGABRT | C | `__pthread_kill_implementation`, double-free | distro 50.0 |
| `crash_2026-04-25_17-57_1.txt` | SIGSEGV | D | `gtk_accessible_value_unref(self=0x7fff00000000)` | distro 50.0; canonical Cluster D, GtkBox parent |
| `crash_2026-04-25_17-59_1.txt` | SIGSEGV | C | `unlink_chunk(p=0x55555693c370)` ← `g_type_free_instance` ← `g_object_unref` ← `gtk_css_node_style_cache_unref` | distro 50.0; **новий шлях** через CSS node style cache |
| `crash_2026-04-25_18-03_1.txt` | SIGSEGV | D | `gtk_accessible_value_unref(self=0x7fff00000000)` | distro 50.0; дубль 17-57 |
| `crash_2026-04-25_18-05_1.txt` | SIGSEGV | B′ (PropExpression watch) | `handler_lookup_by_closure(instance=0x5555565c53c0, closure=0x55555674ec90)` ← `invalid_closure_notify` ← `gtk_property_expression_watch_destroy_closure` ← `gtk_list_item_widget_teardown_object` | distro 50.0; **дуже інформативний** — показує do_notify-emit-during-teardown шлях |
| `crash_2026-04-25_18-07_1.txt` | SIGSEGV | B′ | `_g_closure_supports_invoke_va(closure=0x555500000000)` | distro 50.0 |
| `crash_2026-04-25_2.zip` | – | – | архівна копія цієї теки | – |

## crashes_dir/old_nautilus_crashes_3/

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_18-25_1.txt` | SIGABRT | D→C | `corrupted size vs. prev_size` у `gtk_accessible_attribute_set_free:85` ← `gtk_at_context_finalize` | distro 50.0; **Cluster D ескалює до SIGABRT навіть БЕЗ MALLOC_CHECK_=3** |
| `crash_2026-04-25_18-26_1.txt` | SIGSEGV | D | `gtk_accessible_value_unref(self=0x7fff00000000)` | distro 50.0 |
| `crash_2026-04-25_18-28_1.txt` | SIGABRT | C | double-free | distro 50.0 |
| `crash_2026-04-25_18-29_1.txt` | SIGSEGV | B-jump | `0x7fff00000000 in ?? ()` ← `closure_invoke_notifiers` ← … ← `gtk_column_view_row_widget_teardown_object` ← `gtk_signal_list_item_factory_teardown` | distro 50.0; **підтверджує не-Grid View шлях (List View / ColumnView)** |
| `crash_2026-04-25_3.zip` | – | – | архівна копія | – |

## crashes_dir/old_nautilus_crashes_4/

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_19-11_1.txt` | SIGSEGV | B | `g_closure_unref(closure=0x555500000000)` | distro 50.0 |
| `crash_2026-04-25_19-13_1.txt` | SIGTRAP | A (gtk#7910 direct hit) | `_g_log_abort` після CRITICAL `unable to remove uninstalled invalidation notifier`, потім assertion `(handler != NULL)` | distro 50.0; чистий Cluster A |
| `crash_2026-04-25_19-14_1.txt` | SIGSEGV | B | `g_closure_unref(closure=0x555500000000)` | distro 50.0; дубль 19-11 |

## crashes_dir/old_nautilus_crashes_5/

### distro 50.0

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `crash_2026-04-25_20-07_1.txt` | SIGSEGV | B | `gtk_widget_dispose_template`, `__inst=0x555500000000` | distro 50.0; canonical Cluster B з debuginfo |
| `crash_2026-04-25_20-09_1.txt` | SIGSEGV | B | те саме, що 20-07 | distro 50.0; дубль |
| `crash_2026-04-25_20-10_strange_1.txt` | clean | – | (порожній; інфер вийшов нормально) | distro 50.0 |
| `crash_2026-04-25_20-11_1.txt` | SIGSEGV | C | `_int_malloc` | distro 50.0; дубль `crashes_to_nautilus_1/20-11` |
| `crash_2026-04-25_20-13_1.txt` | SIGABRT | C/B | `corrupted size vs. prev_size`, той самий стек | distro 50.0; дубль `crashes_to_nautilus_1/20-13` |
| `crash_2026-04-25_20-15_1.txt` | SIGTRAP | A′ | `object_already_finalized` | distro 50.0; дубль `crashes_to_nautilus_1/20-15` |

### Flatpak Nightly (`org.gnome.Nautilus.Devel//master`, GTK4 master)

| Файл | Сигнал | Кластер | Лиф | Маркер |
|------|--------|---------|-----|--------|
| `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt` | SIGSEGV | D | `gtk_accessible_value_unref(self=0x7fff00000000)` | Flatpak Vulkan |
| `nautilus-flatpak-vulkan-2026-04-25_22-11-59.txt` | SIGSEGV | B | `g_closure_unref(closure=0x555500000000)` | Flatpak Vulkan |
| `nautilus-flatpak-vulkan-nomalloc-22-00-19.txt` | SIGSEGV | E | `dispatch_event(proxy=0x555500000000)` | Flatpak Vulkan, **БЕЗ MALLOC_*** — підтверджує, що Cluster E реальний |
| `nautilus-flatpak-gl-2026-04-25_22-13-09.txt` | SIGSEGV | B (PropExpression) | `closure_invoke_notifiers(closure=0x555556e54c60)` ← PropertyExpression watch teardown | Flatpak GL |
| `nautilus-flatpak-gl-2026-04-25_22-14-22.txt` | SIGABRT | A | `__pthread_kill_implementation` (через `_g_log_abort` ← `(handler != NULL)`) | Flatpak GL; canonical gtk#7910 hit на GTK master |
| `nautilus-flatpak-cairo-2026-04-26_14-12-54.txt` | SIGSEGV | B″ | `signal_emit_valist_unlocked(instance=0x55555663a790, signal_id=1, detail=2688)` ← `gtk_tree_list_model_clear_node` | Flatpak Cairo; corrupted handler-list |
| `nautilus-flatpak-cairo-2026-04-26_14-15-16.txt` | SIGSEGV | B (PropExpression) | `closure_invoke_notifiers(closure=0x5555562421c0)` ← PropertyExpression watch teardown | Flatpak Cairo |
| `nautilus-flatpak-cairo-2026-04-26_14-16-31.txt` | SIGABRT | D | `__pthread_kill_implementation` (через `corrupted size vs. prev_size` у `gtk_accessible_attribute_set_free`), GtkBox parent | Flatpak Cairo |
| `nautilus-flatpak-cairo-2026-04-26_14-18-14.txt` | SIGABRT | D | те саме, GtkListItemWidget parent | Flatpak Cairo |
| `nautilus-flatpak-crash.txt` | SIGSEGV | E | `dispatch_event(proxy=0x555500000000)` | Flatpak GL, ранній лог |
| `nautilus-flatpak-crash-gl.txt` | SIGSEGV | E | `dispatch_event(proxy=0x555500000000)` | Flatpak GL |

### Valgrind

| Файл | Зміст |
|------|-------|
| `nautilus-vg_1st_report.log` | Valgrind Memcheck: `Invalid read of size 8` у `gtk_drag_source_drag_timeout_cb` ← `g_source_callback_unref` (GtkDragSource UAF: timeout не скасовується); багато `Invalid read/write` навколо `g_signal_group_unbind`, `g_object_notify_queue_thaw` (Cluster B). |
| `nautilus-vg_2nd.log` | Повтор з тих самих сигнатур, додатково `gtk_accessible_value_unref` (Cluster D) як `Invalid read`. |

---

## dir_listings_1/

| Файл | Зміст |
|------|-------|
| `dirsListing_1.txt` | 872 рядки, 144K — повний `ls -liaR` дерева `toNewer_26.04_2/`. Топ-рівневі: `.cache/`, `.config/Cursor/`, `cursor_ide/{project_1..6}/`, `gpu-goodboot{26,27}-u2604/`, `new_dmcub_1/`, `old_1_working/`, `.ssh/`, `.cursor/`. |
| `dirsListing_2.txt` | 872 рядки, 80K — скорочений вивід тих самих директорій. |
| `filesListing_1.txt` | 4510 рядків, 724K — `ls -liaR` файлів. Top extensions: `.json` (383), `.log` (322), `.txt` (116), `.png` (75), `.md` (57), `.hyb` (52), `.old` (37), `.html` (27), `.js` (21), `.pb` (17). |
| `filesListing_2.txt` | 4510 рядків, 392K — скорочена форма. |

---

## follow-ups/

Усі файли тут — копії перерахованих вище логів, групованих для тарбол-аттачментів.
**Унікальних логів немає**.

| Тека / архів | Файли | Прикріплено до |
|--------------|-------|----------------|
| `crashes_to_nautilus_1.zip` | `crash_2026-04-25_20-11_1.txt`, `_20-13_1.txt`, `_20-15_1.txt` | nautilus#4035, основний коментар |
| `follow-up-1/` | `crash_2026-04-25_17-57_1.txt` (D), `_18-07_1.txt` (B′), `_19-13_1.txt` (gtk#7910 hit), `nautilus-flatpak-vulkan-nomalloc-22-00-19.txt` (E) | nautilus#4035, follow-up #1 |
| `follow-up-2/` | 4 Flatpak Cairo логи (B″, B, D-SIGABRT×2) | nautilus#4035, follow-up #2 |
| `gtk-7910-followup/` | `my_crash_2026-04-25_19-13_1.txt` (gtk#7910 direct), `nautilus-flatpak-cairo-2026-04-26_14-12-54.txt` (B″), `nautilus-flatpak-cairo-2026-04-26_14-15-16.txt` (B PropExpr), `nautilus-flatpak-gl-2026-04-25_22-14-22.txt` (A) | gtk#7910 |
| `gtk-cluster-d/` | `crash_2026-04-25_17-57_1.txt` (D primary), `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt` (D Flatpak), `nautilus-flatpak-cairo-2026-04-26_14-16-31.txt` (D SIGABRT GtkBox), `nautilus-flatpak-cairo-2026-04-26_14-18-14.txt` (D SIGABRT GtkListItemWidget) | було gtk#8173 (закрито); цитувати в gtk#7910 / nautilus#4035 |
