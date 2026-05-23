# AllFindings.md — Nautilus crashes investigation (2026-04-25 / 26)

Цей файл — повний звіт по всьому, що зібрано в `/home/denixx/projects/nautilus_things/`.
Він охоплює:

- репродуктивне середовище і дані;
- per-file аналіз кожного крешу/логу;
- кластери крешів (A / B / B' / B'' / C / D / E + варіанти);
- асиметрію в `nautilus-grid-cell.c` vs `nautilus-name-cell.c` (50.0 і master);
- зв'язок з апстрім-issue;
- напрям виправлення.

Доповнюючі деталі винесені у сусідні MD-файли в цій же теці:

- [`Clusters.md`](Clusters.md) — таксономія кластерів зі стек-патернами.
- [`PerFile.md`](PerFile.md) — компактна таблиця "файл → кластер → сигнал → лиф".
- [`SourceAsymmetry.md`](SourceAsymmetry.md) — діагностика різниці між `grid_cell` і `name_cell`.
- [`ReproEnvironment.md`](ReproEnvironment.md) — середовище, дані для репро, env-змінні.
- [`UpstreamIssues.md`](UpstreamIssues.md) — список релевантних upstream issue.
- [`FixDirection.md`](FixDirection.md) — гіпотези фіксу і експерименти.

---

## 1. TL;DR

Nautilus падає при **дуже швидкій зміні поточної локації** (List View або Grid View) у великій
теці з ~870 піддиректоріями і ~4500 файлами; **достатньо лише навігації по дереву**, без
копіювання чи переміщення файлів (див. [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297) #21–22). Частина зібраних логів з’явилась під час ранніх
експериментів із масовими операціями — той самий каскад `items-changed` / teardown рядків,
але **корінь не в самому копіюванні**. Креш стабільно відтворюється на:

- Ubuntu 26.04, Nautilus 50.0 (system build, GTK4 4.20.x, GLib 2.86.x);
- Flatpak Nightly `org.gnome.Nautilus.Devel//master` (Nautilus 51.alpha, GTK4 master);
- незалежно від `GSK_RENDERER` (Vulkan, GL, Cairo);
- незалежно від `MALLOC_CHECK_` (відтворюється і без, і з `MALLOC_CHECK_=3`, `MALLOC_PERTURB_=85`).

Корінь — **race / use-after-free навколо тісно зв'язаних `GSignalGroup` + `GBinding` +
`GtkPropertyExpression watch`**, які тримають назад-посилання на закриту/звільнену
`NautilusGridCell` / `NautilusNameCell` / `NautilusViewItem`. Симптоматика на леві стека —
`g_closure_unref(0x555500000000)`, `gtk_accessible_value_unref(0x7fff00000000)`,
`gtk_widget_dispose_template`, або `closure_invoke_notifiers` з прапорцем
"uninstalled invalidation notifier"/"`(handler != NULL)`".

Memtest86+ (≥10h) не показав помилок RAM. Креш відтворюється "у будь-якому користувацькому
сценарії", де GTK list-view встигає кілька разів перебудувати/переробити recycled-items.

---

## 2. Структура `/home/denixx/projects/nautilus_things/`

```
nautilus_things/
├── ai_thoughts_md/                   ← ця тека з нотатками
│   ├── AllFindings.md                ← цей файл
│   ├── Clusters.md
│   ├── PerFile.md
│   ├── SourceAsymmetry.md
│   ├── ReproEnvironment.md
│   ├── UpstreamIssues.md
│   └── FixDirection.md
├── crashes_dir/
│   ├── crashes_to_nautilus_1/        ← 3 файли, прикріплені до nautilus#4035 в основному коментарі
│   │   ├── crash_2026-04-25_20-11_1.txt   (SIGSEGV у _int_malloc, heap fallout)
│   │   ├── crash_2026-04-25_20-13_1.txt   (SIGABRT, "corrupted size vs. prev_size", через gtk_label_finalize ← gtk_widget_dispose_template)
│   │   └── crash_2026-04-25_20-15_1.txt   (SIGTRAP, GLib-GObject-CRITICAL "object_already_finalized")
│   ├── old_nautilus_crashes_1/       ← найраніші логи, перший день репорту
│   │   ├── crash_2026-04-25_10-42_1.txt   (jump to 0x7fff00000000, gtk#7910 hit)
│   │   ├── crash_2026-04-25_10-49_1.txt   (SIGSEGV у gtk_widget_dispose_template, без debuginfo)
│   │   ├── crash_2026-04-25_10-54_1.txt   (SIGABRT, double-free)
│   │   ├── nautilus_crashing_1.log        (~1k рядків journalctl, 3 kernel-segfault у libwayland-client)
│   │   ├── nautilus_crashing_2_full_journal_2_boots.log (~20k рядків, 3 kernel-segfault у libwayland-client)
│   │   └── nautilus_crashing_2_full_journal_2_boots.tar.xz (упакована копія)
│   ├── old_nautilus_crashes_2/       ← 6 файлів від ~17:56–18:07
│   │   ├── crash_2026-04-25_17-56_1.txt   (SIGABRT, double-free)
│   │   ├── crash_2026-04-25_17-57_1.txt   (Cluster D: gtk_accessible_value_unref(self=0x7fff00000000), GtkBox parent)
│   │   ├── crash_2026-04-25_17-59_1.txt   (Cluster C: SIGSEGV у unlink_chunk, через gtk_css_node_style_cache_unref)
│   │   ├── crash_2026-04-25_18-03_1.txt   (Cluster D)
│   │   ├── crash_2026-04-25_18-05_1.txt   (Cluster B: handler_lookup_by_closure через GtkPropertyExpression watch у gtk_list_item_widget_teardown_object)
│   │   ├── crash_2026-04-25_18-07_1.txt   (Cluster B': _g_closure_supports_invoke_va(closure=0x555500000000))
│   │   └── crash_2026-04-25_2.zip
│   ├── old_nautilus_crashes_3/       ← 4 файли ~18:25–18:29
│   │   ├── crash_2026-04-25_18-25_1.txt   (Cluster D + Cluster C: gtk_accessible_attribute_set_free → unlink_chunk SIGABRT)
│   │   ├── crash_2026-04-25_18-26_1.txt   (Cluster D)
│   │   ├── crash_2026-04-25_18-28_1.txt   (Cluster C: SIGABRT, double free)
│   │   ├── crash_2026-04-25_18-29_1.txt   (Cluster B з jump до 0x7fff00000000 — GtkColumnViewRowWidget teardown)
│   │   └── crash_2026-04-25_3.zip
│   ├── old_nautilus_crashes_4/       ← 3 файли ~19:11–19:14
│   │   ├── crash_2026-04-25_19-11_1.txt   (Cluster B: g_closure_unref(0x555500000000))
│   │   ├── crash_2026-04-25_19-13_1.txt   (gtk#7910 hit: "uninstalled invalidation notifier" + assertion failed: (handler != NULL), SIGTRAP)
│   │   └── crash_2026-04-25_19-14_1.txt   (Cluster B)
│   └── old_nautilus_crashes_5/       ← найбільший набір, 18 файлів (system + Flatpak Nightly)
│       ├── crash_2026-04-25_20-07_1.txt              (Cluster B: gtk_widget_dispose_template, __inst=0x555500000000)
│       ├── crash_2026-04-25_20-09_1.txt              (Cluster B: дубль 20-07 на тій же адресі)
│       ├── crash_2026-04-25_20-10_strange_1.txt      (порожній — inferior вийшов нормально)
│       ├── crash_2026-04-25_20-11_1.txt              (Cluster C: SIGSEGV у _int_malloc)
│       ├── crash_2026-04-25_20-13_1.txt              (Cluster C: SIGABRT, дубль 20-13 з crashes_to_nautilus_1)
│       ├── crash_2026-04-25_20-15_1.txt              (Cluster A варіант: object_already_finalized SIGTRAP)
│       ├── nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt   (Cluster D, Flatpak Vulkan)
│       ├── nautilus-flatpak-vulkan-2026-04-25_22-11-59.txt   (Cluster B,  Flatpak Vulkan)
│       ├── nautilus-flatpak-vulkan-nomalloc-22-00-19.txt     (Cluster E, Flatpak Vulkan, БЕЗ MALLOC_*)
│       ├── nautilus-flatpak-gl-2026-04-25_22-13-09.txt       (Cluster B з PropertyExpression, Flatpak GL)
│       ├── nautilus-flatpak-gl-2026-04-25_22-14-22.txt       (Cluster A: assertion '(handler != NULL)' SIGABRT, Flatpak GL)
│       ├── nautilus-flatpak-cairo-2026-04-26_14-12-54.txt    (Cluster B'': signal_emit з corrupted handler-list, Flatpak Cairo, GtkTreeListRow path)
│       ├── nautilus-flatpak-cairo-2026-04-26_14-15-16.txt    (Cluster B з PropertyExpression watch teardown, Flatpak Cairo)
│       ├── nautilus-flatpak-cairo-2026-04-26_14-16-31.txt    (Cluster D SIGABRT, GtkBox parent, Flatpak Cairo)
│       ├── nautilus-flatpak-cairo-2026-04-26_14-18-14.txt    (Cluster D SIGABRT, GtkListItemWidget parent, Flatpak Cairo)
│       ├── nautilus-flatpak-crash.txt                         (Cluster E: dispatch_event(proxy=0x555500000000), Flatpak GL/Vulkan)
│       ├── nautilus-flatpak-crash-gl.txt                      (Cluster E)
│       ├── nautilus-vg_1st_report.log                         (Valgrind Memcheck #1: GtkDragSource UAF + Cluster B Invalid read/write)
│       └── nautilus-vg_2nd.log                                (Valgrind Memcheck #2)
├── dir_listings_1/
│   ├── dirsListing_1.txt   (872 рядки, повний рекурсивний листинг тек: `ls -liaR …` без `-w`)
│   ├── dirsListing_2.txt   (та сама структура, скорочений вивід)
│   ├── filesListing_1.txt  (4510 рядків — повний рекурсивний листинг файлів)
│   └── filesListing_2.txt  (4510 рядків, скорочена форма)
└── follow-ups/
    ├── crashes_to_nautilus_1.zip                  (3 файли вище, тарбол для основного коментаря nautilus#4035)
    ├── follow-up-1/                                ← тарбол follow-up #1 (4 файли, переважно Flatpak Nightly)
    │   ├── crash_2026-04-25_17-57_1.txt           (Cluster D)
    │   ├── crash_2026-04-25_18-07_1.txt           (Cluster B')
    │   ├── crash_2026-04-25_19-13_1.txt           (gtk#7910 hit)
    │   └── nautilus-flatpak-vulkan-nomalloc-22-00-19.txt (Cluster E без MALLOC_*)
    ├── follow-up-2/                                ← Cairo follow-up (4 файли)
    │   ├── nautilus-flatpak-cairo-2026-04-26_14-12-54.txt (Cluster B'')
    │   ├── nautilus-flatpak-cairo-2026-04-26_14-15-16.txt (Cluster B)
    │   ├── nautilus-flatpak-cairo-2026-04-26_14-16-31.txt (Cluster D SIGABRT)
    │   └── nautilus-flatpak-cairo-2026-04-26_14-18-14.txt (Cluster D SIGABRT)
    ├── gtk-7910-followup/                          ← тарбол для gtk#7910 (4 файли)
    │   ├── my_crash_2026-04-25_19-13_1.txt        (gtk#7910 direct hit)
    │   ├── nautilus-flatpak-cairo-2026-04-26_14-12-54.txt (Cluster B'')
    │   ├── nautilus-flatpak-cairo-2026-04-26_14-15-16.txt (Cluster B PropExpression)
    │   └── nautilus-flatpak-gl-2026-04-25_22-14-22.txt    (Cluster A)
    └── gtk-cluster-d/                              ← тарбол для нової GTK issue (Cluster D)
        ├── crash_2026-04-25_17-57_1.txt           (Cluster D primary)
        ├── nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt  (Cluster D)
        ├── nautilus-flatpak-cairo-2026-04-26_14-16-31.txt   (Cluster D SIGABRT, GtkBox parent)
        └── nautilus-flatpak-cairo-2026-04-26_14-18-14.txt   (Cluster D SIGABRT, GtkListItemWidget parent)
```

Доступний код:

- `/home/denixx/nautilus-50.0/` — стабільна 50.0;
- `/home/denixx/nautilus_git_1/` — git master / nightly (51.alpha).

---

## 3. Репродуктивне середовище

Деталі — у [`ReproEnvironment.md`](ReproEnvironment.md). Стисло:

- **OS**: Ubuntu 26.04 (kernel `7.0.0-14-generic`, x86_64).
- **Сесія**: Wayland.
- **GPU**: AMD Radeon (Mesa, `libvulkan_radeon.so`, `libEGL_mesa.so.0`).
- **CPU**: AMD Ryzen 7 7735HS with Radeon Graphics.
- **GTK4**: 4.20.x (system) і master (Flatpak Nightly).
- **GLib**: 2.86.x (system) і master.
- **Файлова система**: ext4 (за вмістом journalctl).

**Тригер репро** (мінімальний): швидка навігація по дереву каталогів усередині `toNewer_26.04_2/`
(≈872 директорій, ≈4510 файлів — переважно дрібні JSON/log/txt/png, включно з пробілами
і Cyrillic в іменах), **List View або Grid View**, без DnD / cut-paste / move to trash.
Креш зазвичай відбувається **під час** завершення списку елементів у вікні Nautilus, тобто на
черговому `items-changed` від `GtkSortListModel` →
`gtk_list_item_manager_model_items_changed_cb` →
`gtk_list_item_change_finish` → `g_hash_table_remove_all` (звільнення recycled-items).

**Env-змінні для репро/діагностики**:

```bash
G_SLICE=always-malloc
MALLOC_CHECK_=3                # ескалює UAF до SIGABRT
MALLOC_PERTURB_=85             # пише 0x55 у звільнену пам'ять → бачимо 0x555500000000-pattern
GSK_RENDERER=vulkan|gl|cairo   # креш не залежить від рендерера
G_DEBUG=fatal-criticals,fatal-warnings,gc-friendly  # перетворює CRITICAL у abort
```

Для Flatpak Nightly:

```bash
flatpak run --env=MALLOC_CHECK_=3 --env=MALLOC_PERTURB_=85 \
            --env=GSK_RENDERER=cairo \
            org.gnome.Nautilus.Devel//master
```

> Примітка: під Flatpak `G_DEBUG=fatal-*` рано вбиває процес ще до показу вікна
> (через innocuous CRITICAL у нічних білдах), тому в Flatpak ми його не вмикали.

---

## 4. Кластери крешів

Деталі — у [`Clusters.md`](Clusters.md). Тут — швидке зведення.

| ID | Назва (leaf) | Поясн. | Сигнал | MALLOC_CHECK_3 ескалація |
|----|--------------|--------|--------|--------------------------|
| **A** | `assertion failed: (handler != NULL)` у `invalid_closure_notify` | "Чисте" падіння GLib-GObject; внутрішня перевірка drained closure handler list | SIGTRAP/SIGABRT | вже саме SIGABRT |
| **A′** | `g_object_unref: assertion '!object_already_finalized' failed` | Та ж родина — UAF на `g_object_unref`-овано-вже-finalized object | SIGTRAP | те саме |
| **B** | `g_closure_unref(closure=0x555500000000)` / `_g_closure_supports_invoke_va(closure=0x555500000000)` / leaf inside `gtk_widget_dispose_template` із `__inst=0x555500000000` | UAF closure: нижня половина адреси перетерта `MALLOC_PERTURB_=0x55` | SIGSEGV | під MALLOC_CHECK_=3 далі ловиться "double free" або "corrupted size vs. prev_size" — Cluster C escalation |
| **B′** | `signal_emit_valist_unlocked(... 0x555500000000)` / `handler_lookup_by_closure(... 0x555500000000)` | Та ж сама родина, але креш на signal-emission path: emit → walk_handlers → poisoned closure | SIGSEGV | те саме |
| **B″** | `closure_invoke_notifiers(notify_type=1, closure=...)` із порушеною handler-link / або `signal_emit_valist_unlocked` із поломаним handler-list через `gtk_tree_list_model_clear_node` | Список handler-ів у `instance` має проміжний звільнений елемент; Flatpak Cairo шлях | SIGSEGV | те саме |
| **B (jump-to-poison)** | `#0 0x00007fff00000000 in ?? ()` — execution flow стрибнув у poisoned function pointer | Сектор `closure->meta_marshal` / vtable slot перезаписано на `0x7fff00000000` | SIGSEGV (no debuginfo on frame 0) | те саме |
| **C** | `unlink_chunk` / `_int_malloc` / `_int_free_chunk` у `malloc_printerr "corrupted size vs. prev_size"` / "double free" | Heap-corruption fallout. Найчастіше це наслідок попереднього UAF | SIGABRT (без MALLOC_CHECK_), SIGABRT раніше з MALLOC_CHECK_=3 | – |
| **D** | `gtk_accessible_value_unref(self=0x7fff00000000)` у `gtk_accessible_attribute_set_free` ← `gtk_at_context_finalize` | UAF в accessibility teardown: `attribute_values[i]` перезаписано patterns `0x7fff00000000` | SIGSEGV | під MALLOC_CHECK_=3 ескалює до SIGABRT з `corrupted size vs. prev_size` (бо `g_free(self->attribute_values)` далі натикається на пошкоджений heap chunk) |
| **E** | `dispatch_event(proxy=0x555500000000)` у `libwayland-client.so` | Wayland proxy об'єкт пережив свого власника на ~1 itera | SIGSEGV | те саме (kernel: segfault at `<heap_high_bits>00000028` в libwayland-client) |

Усі кластери виникають на одному й тому самому "макро-контексті":
**`gtk_list_item_manager_model_items_changed_cb` → `gtk_list_item_change_finish` →
`g_hash_table_remove_all` recycled-items → каскад `g_object_unref` на дочірніх віджетах**.
Тобто **всі кластери — варіанти однієї і тієї ж UAF в bookkeeping тимчасових list-item-ів**,
які тримають bind-тісні з `NautilusViewItem`. Що саме проявиться (B, D, E) залежить від того,
**яка структура** першою дотягнеться до вже звільненої пам'яті.

---

## 5. Per-file аналіз

Повна таблиця — у [`PerFile.md`](PerFile.md). Нижче — стиснута форма (40 файлів).

### 5.1. `crashes_dir/crashes_to_nautilus_1/` (3 файли, прикріплені до nautilus#4035)

- **`crash_2026-04-25_20-11_1.txt`** — SIGSEGV у `_int_malloc`. Cluster C (heap fallout).
  Стек впирається в `gtk_widget_dispose_template` → `g_hash_table_remove_all_nodes` під
  `gtk_list_item_change_finish`. Heap уже зіпсуто до моменту крешу.

- **`crash_2026-04-25_20-13_1.txt`** — SIGABRT, `corrupted size vs. prev_size`. Cluster B/C.
  Шлях: `gtk_label_finalize` ← `g_object_unref` ← `gtk_box_dispose` ← `g_hash_table_remove`
  (template auto-children) ← `gtk_widget_dispose_template` ← `nautilus_grid_cell_dispose`
  (`/src/nautilus-grid-cell.c:243`) ← `gtk_list_item_set_child(NULL)` ← teardown.
  **Це найчіткіший зразок шляху, що йде через `nautilus_grid_cell_dispose`** і ламає heap у
  finalize дочірнього `GtkLabel`.

- **`crash_2026-04-25_20-15_1.txt`** — SIGTRAP. Cluster A′:
  `GLib-GObject-CRITICAL: g_object_unref: assertion '!object_already_finalized' failed`.
  Той самий шлях, але GLib встигає піймати UAF до SEGV-у на читанні. Адреса `0x5555569e5930`.

### 5.2. `crashes_dir/old_nautilus_crashes_1/`

- **`crash_2026-04-25_10-42_1.txt`** — F0 = `0x7fff00000000 in ?? ()`. **Hybrid: gtk#7910 +
  jump-to-poison**. Перед SEGV — CRITICAL `unable to remove uninstalled invalidation notifier:
  0x7ffff7d80e20 (0x555556679670)` — це сигнатура gtk#7910. Потім execution стрибає на
  `0x7fff00000000` (poisoned vtable slot). Шлях: `g_closure_unref` ← `g_signal_handlers_destroy`
  ← `g_object_unref` ← `gtk_list_item_set_child` ← `gtk_signal_list_item_factory_teardown` ← ….

- **`crash_2026-04-25_10-49_1.txt`** — SIGSEGV у `gtk_widget_dispose_template` (без debuginfo
  на цій версії GTK). Cluster B path. Той самий стек.

- **`crash_2026-04-25_10-54_1.txt`** — SIGABRT, "double free or corruption". Cluster C.

- **`nautilus_crashing_1.log` / `nautilus_crashing_2_full_journal_2_boots.log`** — `journalctl`-фрагменти.
  3 kernel-level SIGSEGV у **`libwayland-client.so.0.24.0[86ca, +7000]`** з адресою
  `XX_XX00000028` (offset `+0x28` від pointer-а на proxy). Це Cluster E,
  які виникали ще до підключення MALLOC-діагностики. Підтверджують, що проблема **передує**
  будь-яким змінам env-змінних. Час: 25.04 8:53–8:58 (3 креші за 5 хвилин).

### 5.3. `crashes_dir/old_nautilus_crashes_2/`

- **`crash_2026-04-25_17-56_1.txt`** — SIGABRT, double-free. Cluster C.
- **`crash_2026-04-25_17-57_1.txt`** — `gtk_accessible_value_unref(self=0x7fff00000000)`.
  Cluster D primary. Стек: `gtk_accessible_attribute_set_free` ← `gtk_at_context_finalize`
  ← `g_object_unref(GtkATSpiContext)` ← finalize parent widget (GtkBox).
- **`crash_2026-04-25_17-59_1.txt`** — SIGSEGV у `unlink_chunk`. Cluster C (heap corruption),
  але новий шлях: `gtk_css_node_style_cache_unref` ← `g_hash_table_remove_all_nodes` —
  CSS node style cache hash-table містив об'єкт із вже звільненою інстансом.
- **`crash_2026-04-25_18-03_1.txt`** — Cluster D. Дубль 17-57.
- **`crash_2026-04-25_18-05_1.txt`** — **дуже інформативний**: SIGSEGV у `handler_lookup_by_closure`
  під час `invalid_closure_notify`. Шлях:
  `gtk_property_expression_watch_destroy_closure` ← `gtk_property_expression_watch_expr_notify_cb`
  ← `g_signal_emit("notify", detail=965)` (∼ `notify::position`) на `GtkListItem` із
  `gtk_list_item_widget_teardown_object` ← `gtk_signal_list_item_factory_teardown`. Тобто
  під час teardown **`do_notify` ще раз емітить `notify::position`**, на нього висить
  `GtkPropertyExpression` watch, watch у момент destroy_closure намагається invalidate-нути
  свою закриту — і ламається на `handler_lookup_by_closure`, бо handler list уже частково
  пошкоджений. **Це додатковий вхідний шлях у Cluster B, який не йде через
  `nautilus_grid_cell_dispose → gtk_widget_dispose_template`, а через
  `gtk_list_item_widget_teardown_object`**.
- **`crash_2026-04-25_18-07_1.txt`** — SIGSEGV у `_g_closure_supports_invoke_va(closure=0x555500000000)`.
  Cluster B′.

### 5.4. `crashes_dir/old_nautilus_crashes_3/`

- **`crash_2026-04-25_18-25_1.txt`** — SIGABRT, `corrupted size vs. prev_size` у
  `gtk_accessible_attribute_set_free` (у середині `g_free` на `attribute_values`).
  **Cluster D ескалює до Cluster C SIGABRT** (на distro-нашій 50.0, без MALLOC_CHECK_=3).
- **`crash_2026-04-25_18-26_1.txt`** — Cluster D primary.
- **`crash_2026-04-25_18-28_1.txt`** — Cluster C (double free).
- **`crash_2026-04-25_18-29_1.txt`** — F0 = `0x7fff00000000 in ?? ()`. Той самий шлях, що
  18-05, але через `gtk_column_view_row_widget_teardown_object` →
  `gtk_column_view_row_do_notify` (List View, не Grid View). Підтверджує: проблема — **не
  специфічна до Grid View**.

### 5.5. `crashes_dir/old_nautilus_crashes_4/`

- **`crash_2026-04-25_19-11_1.txt`**, **`19-14_1.txt`** — `g_closure_unref(closure=0x555500000000)`,
  Cluster B leaf.
- **`crash_2026-04-25_19-13_1.txt`** — SIGTRAP, **direct gtk#7910 hit**:
  `unable to remove uninstalled invalidation notifier`, потім assertion
  `(handler != NULL)`. Цей файл потрапив у тарбол `gtk-7910-followup`.

### 5.6. `crashes_dir/old_nautilus_crashes_5/` (Flatpak Nightly + ще трохи distro)

- **`crash_2026-04-25_20-07_1.txt`** і **`20-09_1.txt`** — SIGSEGV всередині `gtk_widget_dispose_template`,
  локальна змінна `__inst = 0x555500000000`. Cluster B з debuginfo. Стек:
  `nautilus_grid_cell_dispose:243` ← `gtk_list_item_set_child(NULL)` ← teardown.
- **`crash_2026-04-25_20-10_strange_1.txt`** — порожній файл (inferior нормально завершився).
- **`crash_2026-04-25_20-11_1.txt`** — Cluster C, SIGSEGV у `_int_malloc`. Дубль 20-11 з
  `crashes_to_nautilus_1`.
- **`crash_2026-04-25_20-13_1.txt`** — Cluster C SIGABRT. Дубль 20-13 з `crashes_to_nautilus_1`.
- **`crash_2026-04-25_20-15_1.txt`** — Cluster A′. Дубль 20-15.
- **Flatpak Vulkan**:
  - `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt` — Cluster D.
  - `nautilus-flatpak-vulkan-2026-04-25_22-11-59.txt` — Cluster B.
  - `nautilus-flatpak-vulkan-nomalloc-22-00-19.txt` — Cluster E (без MALLOC_*; **підтверджує,
    що Cluster E — не артефакт MALLOC_PERTURB_**).
- **Flatpak GL**:
  - `nautilus-flatpak-gl-2026-04-25_22-13-09.txt` — Cluster B з PropertyExpression watch
    (стек із `gtk_property_expression_watch_destroy_closure`).
  - `nautilus-flatpak-gl-2026-04-25_22-14-22.txt` — Cluster A: `assertion failed: (handler != NULL)`
    (canonical gtk#7910 hit, але вже у GTK master).
- **Flatpak Cairo** (рендерер не GPU):
  - `nautilus-flatpak-cairo-2026-04-26_14-12-54.txt` — Cluster B″: `signal_emit_valist_unlocked`
    падає з пошкодженою handler-list link, шлях через `gtk_tree_list_model_clear_node`.
  - `nautilus-flatpak-cairo-2026-04-26_14-15-16.txt` — Cluster B з PropertyExpression watch
    teardown (Cairo дублікат gl-22-13-09).
  - `nautilus-flatpak-cairo-2026-04-26_14-16-31.txt` — Cluster D SIGABRT, parent `GtkBox`.
  - `nautilus-flatpak-cairo-2026-04-26_14-18-14.txt` — Cluster D SIGABRT, parent `GtkListItemWidget`.
- **`nautilus-flatpak-crash.txt` / `nautilus-flatpak-crash-gl.txt`** — Cluster E, Wayland leaf.
- **`nautilus-vg_1st_report.log`** і **`nautilus-vg_2nd.log`** — Valgrind Memcheck:
  - `Invalid read of size 8` у `gtk_drag_source_drag_timeout_cb` ← `g_source_callback_unref` —
    **GtkDragSource UAF: timeout fires на звільненому контролері** (новий незалежний баг GTK!).
  - Numerous `Invalid read/write` навколо `g_signal_group_unbind` і `g_object_notify_queue_thaw`,
    що корелює з Cluster B.

### 5.7. `dir_listings_1/` (фотографія дерева, в якому відтворюється)

- `dirsListing_*.txt`: 872 рядки, 80–144 KB. Дерево `toNewer_26.04_2/`. Топ-рівневі піддерева:
  `.cache/google-chrome/...`, `.config/Cursor/...`, `cursor_ide/{project_1..6}`, `gpu-goodboot*`,
  `new_dmcub_1`, `old_1_working`, `.ssh`, `.cursor`. Багато глибоко-вкладених кеш-тек із
  пробілами (`Code Cache`).
- `filesListing_*.txt`: 4510 файлів, 392–724 KB. Розподіл за розширеннями:
  `.json` (383), `.log` (322), `.txt` (116), `.png` (75), `.md` (57), `.hyb` (52), `.old` (37),
  `.html` (27), `.js` (21), `.pb` (17). **Типовий "browser cache + IDE cache"-набір** з
  малими файлами і глибокими шляхами.

### 5.8. `follow-ups/`

Просто згруповані копії крешів для тарболів-аттачментів — структура файлів та їх
кластерів вище покриває всі шість тек у `follow-ups/`. Жодних унікальних логів тут немає.

---

## 6. Source code asymmetry

Деталі — у [`SourceAsymmetry.md`](SourceAsymmetry.md). Стисло:

```c
/* nautilus-50.0/src/nautilus-grid-cell.c:238–247
   AND nautilus_git_1/src/nautilus-grid-cell.c:270–279 — однаково.       */
static void
nautilus_grid_cell_dispose (GObject *object)
{
    NautilusGridCell *self = (NautilusGridCell *) object;

    gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_GRID_CELL);
    g_clear_object (&self->item_signal_group);

    G_OBJECT_CLASS (nautilus_grid_cell_parent_class)->dispose (object);
}
```

```c
/* nautilus-50.0/src/nautilus-name-cell.c:373–392
   AND nautilus_git_1/src/nautilus-name-cell.c:372–391 — однаково.       */
static void
nautilus_name_cell_dispose (GObject *object)
{
    NautilusNameCell *self = (NautilusNameCell *) object;

    gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_NAME_CELL);

    G_OBJECT_CLASS (nautilus_name_cell_parent_class)->dispose (object);
}

static void
nautilus_name_cell_finalize (GObject *object)
{
    NautilusNameCell *self = (NautilusNameCell *) object;

    g_clear_handle_id (&self->loading_timeout_id, g_source_remove);
    g_clear_object (&self->item_signal_group);
    g_clear_object (&self->file_path_base_location);
    G_OBJECT_CLASS (nautilus_name_cell_parent_class)->finalize (object);
}
```

Обидві комірки в `_init` роблять:

```c
self->item_signal_group = g_signal_group_new (NAUTILUS_TYPE_VIEW_ITEM);
g_signal_group_connect_swapped (self->item_signal_group, "notify::is-cut", …, self);
g_signal_group_connect_swapped (self->item_signal_group, "file-changed",  …, self);
g_object_bind_property (self, "item", child, "item", G_BINDING_DEFAULT);
```

(У `name_cell` додатково `notify::drag-accept`, `notify::loading`.)

**Асиметрія**:

- `grid_cell`: `g_clear_object(&self->item_signal_group)` зроблено **в dispose**, *після*
  `gtk_widget_dispose_template`.
- `name_cell`: те ж саме `g_clear_object` зроблено **в finalize**, тобто значно пізніше.

Поскільки `gtk_widget_dispose_template` всередині фіналізує дочірні віджети, а ті дочірні
зв'язані з `self` через `GBinding(self->item ↔ child->item)` і `GtkPropertyExpression watch`-и,
що тримають backref на `self->item_signal_group` — порядок розриву зв'язків критичний.
У grid_cell: спочатку убиваються діти (з GBinding-ами на `self->item`), потім
`g_clear_object(item_signal_group)`. Якщо посеред `gtk_widget_dispose_template` GBinding
встигне emit notify на `self`, що проб'ється до `item_signal_group` — а в цей момент
auto-children вже частково звільнені — UAF.

Це не значить, що "grid_cell винний" — у name_cell `item_signal_group` живе ще довше,
і той самий UAF може вистрелити пізніше. **Корінь — у GTK4 / GLib teardown ordering**.

---

## 7. Зв'язок із upstream issues

Деталі — у [`UpstreamIssues.md`](UpstreamIssues.md). Стисло:

| Issue | Статус | Релевантність |
|-------|--------|---------------|
| [`nautilus#4035`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035) | **наша основна issue** | Початковий коментар + два follow-up (тарболи як раніше). Після [human-only GTK](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy): у треді лишились **людські** оновлення; якорі: [note_2744445](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744445), [note_2744838](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744838). |
| [`gtk#7910`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910) | open | Прямий збіг із Cluster A; тарбол `gtk-7910-followup/`. Людський follow-up: [note_2744852](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910#note_2744852). |
| [`gtk#8173`](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173) | **закрита** (human-only) | Була окрема issue для Cluster D; логи лишаються в `follow-ups/gtk-cluster-d/` — за потреби цитувати в `gtk#7910` / `nautilus#4035`. |
| [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297) | New (nautilus); Invalid (wayland) | Оновлений summary, remote watches на GitLab #4035 і #7910, аттачі (zip крешів, листинги, follow-up-2); діалог з triager — див. `UpstreamIssues.md`. |
| [`gtk#7968`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7968) | closed | Стосувався іншого сценарію `GtkDropTarget` finalization — частково релевантний, але не покриває наш `GtkDragSource` `drag_timeout` UAF. |
| [`nautilus#3992`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/3992) | open | Про teardown timing в `nautilus_grid_cell` — потенційно дотичне (треба ще дочитати). |
| [`gnome-software#2452`](https://gitlab.gnome.org/GNOME/gnome-software/-/issues/2452), [`gnome-control-center#2704`](https://gitlab.gnome.org/GNOME/gnome-control-center/-/issues/2704), [`gtk#6548`](https://gitlab.gnome.org/GNOME/gtk/-/issues/6548) | різний статус | Інші проєкти бачать **той самий Cluster D leaf** у своїх креш-репортах — це сильний аргумент, що це GTK-side bug, а не Nautilus-specific. |

**Ще не подано**: окрема GTK issue для `GtkDragSource::drag_timeout` UAF, виявленого Valgrind-ом.
Чернетка готова, але не запостена.

---

## 8. Напрям виправлення

Деталі — у [`FixDirection.md`](FixDirection.md). Гіпотези:

1. **Nautilus side**: симетризувати teardown:
   - У `nautilus_grid_cell_dispose`: робити `g_clear_object(&self->item_signal_group)`
     **до** `gtk_widget_dispose_template`, або перенести у `finalize` (як у `name_cell`).
   - У `nautilus_name_cell_dispose`: явно `g_signal_group_set_target(NULL)` ще до dispose
     дочірніх — це обриває forwardні `notify::*` ланцюги до того, як діти почнуть фіналізуватися.

2. **GTK4 side** (більш фундаментально):
   - У `gtk_widget_dispose_template`: спочатку прокатати `gtk_widget_class_for_each_template_child`
     з `g_object_run_dispose` (а не повний unref), потім окремо `unref`.
   - Або: робити `g_signal_group_unbind` у `gtk_at_context_dispose` (а не в finalize),
     щоб accessibility teardown не залишав висячих watch-ів на торкнутих attribute_values.

3. **GtkDragSource UAF**: cancel timeout у `gtk_drag_source_finalize` через
   `g_clear_handle_id (&priv->drag_timeout, g_source_remove)`. Або тримати strong ref на
   контролер у `g_timeout_add_full(..., (GDestroyNotify) g_object_unref)`.

---

## 9. Open questions

- Чи допомагає вимкнути сортування `GtkSortListModel` у `NautilusViewModel`? Не перевіряли.
- Чи відтворюється на GTK 4.18 (попередній stable)? Не перевіряли — Ubuntu 26.04 уже на 4.20.
- Чи допомагає `GTK_A11Y=none` (вимкнути accessibility-back-end)? **Окрема цінна перевірка**
  для Cluster D — теоретично прибере D повністю, але B/E залишаться.
- Чи Wayland-only? Можна перевірити у `XDG_SESSION_TYPE=x11`-сесії.

---

## 10. Що зроблено / що ще можна зробити

**Зроблено:**

- ☑ 32 креш-логи проаналізовано, 8 кластерів описано;
- ☑ Перевірено реплікованість на distro 50.0 і Flatpak Nightly 51.alpha;
- ☑ Перевірено реплікованість на Vulkan/GL/Cairo;
- ☑ `nautilus#4035` — повноцінний коментар + два follow-up із тарболами; після human-only — людські якорі в треді (notes 2744445, 2744838);
- ☑ `gtk#7910` — детальний коментар із виборкою найрелевантніших crash-логів; людський follow-up [note_2744852](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910#note_2744852);
- ☑ `gtk#8173` — відкрита для Cluster D з тарболом, згодом **закрита** (політика GNOME);
- ☑ Launchpad [2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297) — синхронізовано з GitLab (summary, watches, аттачі, коментарі до #29);
- ☑ Memtest86+ ≥10h — RAM не винен;
- ☑ Source-asymmetry задокументовано.

**Не зроблено (можна):**

- ☐ Окрема GTK issue для `GtkDragSource drag_timeout` UAF (чернетка готова);
- ☐ Локальний "patched-build" Nautilus із перенесеним `g_clear_object` у `finalize` →
  перевірити, чи зникає B/B′/B″;
- ☐ Запустити Nautilus із `GTK_A11Y=none` і перевірити, чи зникає Cluster D;
- ☐ Запустити в `XDG_SESSION_TYPE=x11` і перевірити, чи зникає Cluster E.
