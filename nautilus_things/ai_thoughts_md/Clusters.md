# Clusters.md — Таксономія крешів

Усі описані тут кластери — варіанти однієї і тієї самої гонки під час
**`gtk_list_item_manager_model_items_changed_cb` → `gtk_list_item_change_finish` →
`g_hash_table_remove_all` recycled-items**. Що саме проявиться, залежить від того, яка
структура (closure / accessibility ctx / Wayland proxy) першою натикається на UAF.

Скорочення:

- `0x555500000000` — pattern, який лишає `MALLOC_PERTURB_=85` (0x55) у звільненій пам'яті
  (нижні 32 біти затерті 0x55-байтами, верхні залишені);
- `0x7fff00000000` — pattern, типовий для пере-затертих стек-адрес або internal poison
  pattern в GLib heap.

---

## Cluster A — `assertion failed: (handler != NULL)` у `invalid_closure_notify`

**Природа**: GLib-GObject piming sanity check. У момент `g_closure_invalidate` GLib
ходить по handler-list інстансу, шукає handler з відповідним closure pointer; не знаходить
(handler_lookup_by_closure повертає NULL), асертує і викликає `_g_log_abort`.

**Top frames (canonical)**:

```
#0  _g_log_abort (breakpoint=1)
#1  g_logv
#2  g_log
#3  g_assertion_message
#4  invalid_closure_notify (instance=…, closure=…)        ← assertion (handler != NULL)
#5  closure_invoke_notifiers (notify_type=1, closure=…)
#6  closure_invalidate_internal
#7  g_closure_invalidate
…
```

**Файли**: `nautilus-flatpak-gl-2026-04-25_22-14-22.txt` (canonical), почасти
`crash_2026-04-25_19-13_1.txt` (тут із попередньою CRITICAL "uninstalled invalidation notifier").

**Сигнал**: SIGTRAP / SIGABRT.

**Прямий збіг із**: [`gtk#7910`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910).

---

## Cluster A′ — `g_object_unref: assertion '!object_already_finalized' failed`

**Природа**: GLib's `g_object_unref` рано ловить UAF: на `_object` уже стоїть прапорець
`object_already_finalized` (тобто його вже фіналізовано раніше). Це той самий double-unref,
але GLib pre-empts SEGV.

**Top frames**:

```
#0  _g_log_abort
…
#3  g_return_if_fail_warning(... "g_object_unref")
#4  g_object_unref(_object=0x5555569e5930)
#5  gtk_widget_dispose(0x555556641350)
#6  g_object_unref ← g_hash_table_remove_all_nodes ← g_hash_table_destroy
#7  gtk_list_item_change_finish
#8  gtk_list_item_manager_model_items_changed_cb
…
```

**Файл**: `crashes_to_nautilus_1/crash_2026-04-25_20-15_1.txt`,
`old_nautilus_crashes_5/crash_2026-04-25_20-15_1.txt` (дубль).

**Сигнал**: SIGTRAP.

---

## Cluster B — closure (data) UAF

**Природа**: ходячи по `g_object_unref → g_signal_handlers_destroy` або через
`gtk_widget_dispose_template`, GLib читає закриту, що вже звільнена. Нижні 32 біти адреси —
`0x00000000`, верхні — `0x5555` (наслідок `MALLOC_PERTURB_=0x55`).

**Top frames варіанти**:

```
# B-leaf-1 (closure_unref):
#0  g_closure_unref (closure=0x555500000000)
#1  g_signal_handlers_destroy
#2  g_object_unref
…

# B-leaf-2 (через dispose_template):
#0  gtk_widget_dispose_template (widget=0x555556..., __inst=0x555500000000)
#1  nautilus_grid_cell_dispose (object=…) at nautilus-grid-cell.c:243
#2  g_object_unref
#3  gtk_list_item_set_child (self=…, child=0x0)        ← teardown clearing child
#4  gtk_signal_list_item_factory_teardown
#5  gtk_list_factory_widget_teardown_factory
#6  gtk_list_factory_widget_clear_factory
#7  gtk_list_factory_widget_dispose
#8  g_object_unref ← g_hash_table_remove_all
#9  gtk_list_item_change_finish
#10 gtk_list_item_manager_model_items_changed_cb
```

**Файли**: `crash_2026-04-25_19-11_1.txt`, `19-14_1.txt`,
`nautilus-flatpak-vulkan-2026-04-25_22-11-59.txt`,
`old_nautilus_crashes_5/crash_2026-04-25_20-07_1.txt`, `20-09_1.txt`.

**Сигнал**: SIGSEGV.

---

## Cluster B′ — closure UAF на signal-emission path

**Природа**: те саме, але креш у `signal_emit_*` / `_g_closure_supports_invoke_va` /
`handler_lookup_by_closure` — тобто signal_emit ходить по handler-list, потрапляє на
звільнений closure pointer.

**Top frames**:

```
#0  _g_closure_supports_invoke_va (closure=0x555500000000)
#1  signal_emit_valist_unlocked (instance=…, signal_id=…, detail=…)
#2  g_signal_emit_valist
#3  g_signal_emit
…
```

або:

```
#0  handler_lookup_by_closure (instance=…, closure=…)            ← повертає NULL
#1  invalid_closure_notify
#2  closure_invoke_notifiers
#3  closure_invalidate_internal
#4  g_closure_invalidate
#5  gtk_property_expression_watch_destroy_closure
#6  gtk_property_expression_watch_expr_notify_cb
#7  g_closure_invoke ← signal_emit_valist_unlocked (notify, detail=965 ≈ "position")
#8  gtk_list_item_do_notify
#9  gtk_list_item_widget_teardown_object
#10 gtk_signal_list_item_factory_teardown
…
```

**Файли**: `old_nautilus_crashes_2/crash_2026-04-25_18-05_1.txt`,
`crash_2026-04-25_18-07_1.txt`,
`nautilus-flatpak-cairo-2026-04-26_14-15-16.txt`,
`nautilus-flatpak-gl-2026-04-25_22-13-09.txt`.

**Особливо інформативний**: `18-05_1.txt` — показує, що під час teardown емітиться
`notify::position`, на нього висить `GtkPropertyExpression watch`, який намагається
invalidate-нути свою closure — і ламається на handler-lookup-у.

---

## Cluster B″ — corrupted handler-list link (signal-emit на пошкодженому списку)

**Природа**: під час emit, GLib проходить по handler-link (`handler->next`), і `next`
показує на свободну пам'ять. Симптом — SEGV всередині `signal_emit_valist_unlocked` при
дереференсі handler-pointer-а, але закривий вказівник сам валідний.

**Top frames**:

```
#0  signal_emit_valist_unlocked (instance=0x55555663a790, signal_id=1, detail=2688)
#1  g_signal_emit_valist
#2  g_signal_emit
#3  g_object_dispatch_properties_changed
…
#K  gtk_tree_list_model_clear_node (model=…, node=…)            ← у проміжних кадрах
…
```

**Файл**: `nautilus-flatpak-cairo-2026-04-26_14-12-54.txt`.

**Сигнал**: SIGSEGV.

---

## Cluster B (jump-to-poison)

**Природа**: closure->meta_marshal або marshal slot перезаписано
адресою `0x7fff00000000`, GLib викликає її як функцію, RIP=0x7fff00000000.

**Top frames**:

```
#0  0x00007fff00000000 in ?? ()                                  ← jumped to poison
#1  closure_invoke_notifiers (closure=…, notify_type=1)
#2  closure_invalidate_internal
#3  g_closure_invalidate
#4  gtk_property_expression_watch_destroy_closure
…
#15 gtk_column_view_row_do_notify  (АБО  gtk_list_item_do_notify)
#16 gtk_column_view_row_widget_teardown_object  (АБО  gtk_list_item_widget_teardown_object)
…
```

**Файли**: `old_nautilus_crashes_1/crash_2026-04-25_10-42_1.txt` (gtk#7910 hybrid),
`old_nautilus_crashes_3/crash_2026-04-25_18-29_1.txt` (List View, не Grid View).

---

## Cluster C — heap corruption fallout

**Природа**: до моменту крешу heap уже зіпсуто попереднім UAF. Крешить либиць malloc на
наступному `_int_malloc` / `unlink_chunk` / `_int_free_chunk`. Інколи з повідомленням
`corrupted size vs. prev_size`, `double free or corruption (!prev)`, `free(): invalid size`.

**Top frames варіанти**:

```
# C-leaf SIGSEGV:
#0  _int_malloc (av=…, bytes=97)
#1  __GI___libc_malloc
…

# C-leaf SIGABRT through malloc_printerr:
#0  __pthread_kill_implementation (signo=6)
…
#7  malloc_printerr (str="corrupted size vs. prev_size")
#8  unlink_chunk (p=…, av=…)
#9  _int_free_create_chunk
…
#K  gtk_label_finalize (object=…) at gtk/gtklabel.c:1612          ← finalize ламає heap
#K+1 g_object_unref ← gtk_box_dispose ← g_hash_table_remove (template auto-children)
#K+2 gtk_widget_dispose_template
#K+3 nautilus_grid_cell_dispose                                    ← Cluster B вхід
```

**Файли**: `crashes_to_nautilus_1/crash_2026-04-25_20-11_1.txt`, `20-13_1.txt`,
`old_nautilus_crashes_1/crash_2026-04-25_10-54_1.txt`,
`old_nautilus_crashes_2/crash_2026-04-25_17-56_1.txt`, `17-59_1.txt` (через CSS node cache),
`old_nautilus_crashes_3/crash_2026-04-25_18-25_1.txt` (Cluster D escalation),
`crash_2026-04-25_18-28_1.txt`,
`old_nautilus_crashes_5/crash_2026-04-25_20-11_1.txt`, `20-13_1.txt`.

**Спостереження**: під `MALLOC_CHECK_=3` Cluster B/D ескалює до Cluster C SIGABRT раніше,
ніж би SEGV-нув без перевірки. Без `MALLOC_CHECK_` Cluster C трапляється у складніших шляхах
(через `gtk_label_finalize` чи `gtk_css_node_style_cache_unref`).

---

## Cluster D — `gtk_accessible_value_unref(self=0x7fff00000000)`

**Природа**: під час teardown widget-а парент-widget викликає `gtk_at_context_finalize`
свого AT-context-а; той `unref`-ить три набори attribute_set-ів; кожен набір ходить по
своєму `attribute_values[i]` і unref-ить кожен `GtkAccessibleValue`. У певний момент
`attribute_values[i]` показує на адресу `0x7fff00000000` (або інший poison), і
`gtk_accessible_value_unref` робить `self->ref_count -= 1` → SEGV.

**Top frames**:

```
#0  gtk_accessible_value_unref (self=0x7fff00000000) at gtkaccessiblevalue.c:119
#1  gtk_accessible_attribute_set_free (data=…)
#2  g_atomic_rc_box_release_full
#3  g_rc_box_release_full
#4  gtk_accessible_attribute_set_unref
#5  gtk_at_context_finalize (gobject=…)
#6  g_object_unref (parent: GtkATSpiContext)
…
#K  finalize parent widget (GtkBox / GtkListItemWidget / GtkLabel / …)
#K+1 g_hash_table_remove_all_nodes
#K+2 gtk_list_item_change_finish
#K+3 gtk_list_item_manager_model_items_changed_cb
```

**Сигнал**: SIGSEGV.
**Під MALLOC_CHECK_=3** Cluster D ескалює до SIGABRT з `corrupted size vs. prev_size`
у `gtk_accessible_attribute_set_free` (цикл успішно прокатується, але потім `g_free` на
`self->attribute_values` потрапляє на пошкоджений heap chunk).

**Файли**:
- `old_nautilus_crashes_2/crash_2026-04-25_17-57_1.txt` (canonical SIGSEGV, distro 50.0),
- `old_nautilus_crashes_2/crash_2026-04-25_18-03_1.txt`,
- `old_nautilus_crashes_3/crash_2026-04-25_18-25_1.txt` (SIGABRT escalation, distro 50.0),
- `old_nautilus_crashes_3/crash_2026-04-25_18-26_1.txt`,
- `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt` (Flatpak Nightly),
- `nautilus-flatpak-cairo-2026-04-26_14-16-31.txt` (SIGABRT, GtkBox parent),
- `nautilus-flatpak-cairo-2026-04-26_14-18-14.txt` (SIGABRT, GtkListItemWidget parent).

**Подано як**: раніше окрема [`gtk#8173`](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173) (закрита після [human-only](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy)); ті самі логи — `follow-ups/gtk-cluster-d/`, далі цитувати в [`gtk#7910`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910) / [`nautilus#4035`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035).

---

## Cluster E — `dispatch_event(proxy=0x555500000000)` у `libwayland-client.so`

**Природа**: Wayland клієнтська бібліотека отримує event від compositor-а і шукає proxy
об'єкт за ID; proxy на `wl_object` уже звільнений Mesa/GTK слідом за teardown-ом
поверхні / surface-attached resource.

**Top frames**:

```
#0  dispatch_event (display=0x55555..., queue=0x55555...) at wayland-client.c:1686
#1  dispatch_queue                ← libwayland-client.so
#2  wl_display_dispatch_queue
…
```

Або **kernel-level**: `nautilus[<pid>]: segfault at <heap_high>00000028 ip … in libwayland-client.so.0.24.0[86ca, +7000]` — fault address `<proxy>+0x28`, тобто `proxy = <heap_high>00000000`.

**Сигнал**: SIGSEGV.

**Файли**:
- `old_nautilus_crashes_5/nautilus-flatpak-crash.txt` (Flatpak GL),
- `nautilus-flatpak-crash-gl.txt` (Flatpak GL),
- `nautilus-flatpak-vulkan-nomalloc-22-00-19.txt` (Flatpak Vulkan, **без MALLOC_***!),
- `old_nautilus_crashes_1/nautilus_crashing_*.log` (3 окремі kernel-segfault у журналі).

**Важливо**: Cluster E відтворюється і **без** `MALLOC_PERTURB_`, тобто `0x555500000000`
тут — не наш artifact, а валідний у-фінал-зруйнований heap pointer (libc-shaped).
Це означає, що Cluster E — реальний UAF поверх Wayland proxy, не симптом MALLOC-pollution.

---

## Зведена таблиця "сигнал × MALLOC_CHECK_"

|  Cluster  | Без MALLOC_CHECK_  | MALLOC_CHECK_=3  | MALLOC_PERTURB_=85 ефект  |
|-----------|---------------------|-------------------|---------------------------|
| A         | SIGTRAP             | SIGABRT (раніше)  | без впливу                |
| A′        | SIGTRAP             | SIGTRAP           | без впливу                |
| B/B′      | SIGSEGV (після UAF) | SIGABRT (раніше)  | замінює UAF-pointer на 0x555500000000 |
| B″        | SIGSEGV             | SIGABRT (раніше)  | замінює handler-link payload          |
| B (jump)  | SIGSEGV (RIP=0x7fff..) | SIGABRT (раніше)  | замінює function pointer slot          |
| C         | SIGABRT             | SIGABRT           | прискорює виявлення                    |
| D         | SIGSEGV             | SIGABRT (раніше)  | замінює attribute_value pointer        |
| E         | SIGSEGV             | SIGSEGV           | без впливу (відтворюється і без)       |
