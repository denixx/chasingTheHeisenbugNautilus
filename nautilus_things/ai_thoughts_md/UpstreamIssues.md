# UpstreamIssues.md — Релевантні upstream issue

## 1. Основна issue (наша)

### [`nautilus#4035`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035)

Наш основний баг-репорт. Стан:

- ☑ Початковий коментар з описом середовища, репро-кроків і трьома прикріпленими crash-логами
  (`crashes_to_nautilus_1.zip` — `crash_2026-04-25_20-11_1.txt`, `_20-13_1.txt`, `_20-15_1.txt`).
- ☑ Follow-up #1: 4 файли (`follow-up-1/`), Flatpak Nightly Vulkan/GL, демонстрація
  Cluster A/D/E і gtk#7910 hit на distro 50.0.
- ☑ Follow-up #2: 4 файли (`follow-up-2/`), Flatpak Nightly Cairo, демонстрація renderer-незалежності
  і Cluster B″ + Cluster D SIGABRT escalation.

Після [human-only](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy) у GitLab лишаються **людські** коментарі / посилання; зручні якорі:

- [note_2744445](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744445) — «все, що має цінність» (узгоджено з [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297) коментарем #24).
- [note_2744838](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744838) — відповідь Daniel щодо `GSK_RENDERER=cairo` + тести на Cairo (Launchpad #27).

---

## 2. GTK issue, яку ми поглибили

### [`gtk#7910`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910)

Прямий збіг із Cluster A. До нашого коментаря — issue була untriaged з мінімальним repro.

Що ми додали:

- наш Cluster A canonical hit (`crash_2026-04-25_19-13_1.txt`),
- Cluster B PropertyExpression watch path (Flatpak Cairo `nautilus-flatpak-cairo-2026-04-26_14-15-16.txt`),
- Cluster B″ corrupted handler-list (Flatpak Cairo `nautilus-flatpak-cairo-2026-04-26_14-12-54.txt`),
- canonical assertion `(handler != NULL)` SIGABRT на GTK master (Flatpak GL `nautilus-flatpak-gl-2026-04-25_22-14-22.txt`),
- посилання на наш `nautilus-grid-cell.c::nautilus_grid_cell_dispose` асимітричний teardown.

Тарбол: `follow-ups/gtk-7910-followup/`.

Після human-only додано **людський** follow-up з посиланням на цю політику та пропозицією допомогти з фіксом до 26.04.1: [note_2744852](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910#note_2744852) (дзеркало посилання з [Launchpad #2150297, коментар #28](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297/comments/28)).

---

## 3. GTK issue, яку ми відкрили

### [`gtk#8173`](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173) — **закрито** (2026-04-26)

Після політики [human-only у GTK](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy) issue закрита автором; **матеріали залишаються корисними** як додатковий Cluster D пакет — їх можна вручну процитувати в `gtk#7910` або `nautilus#4035`, не відкриваючи окремий тікет з AI-текстом.

Було: нова GTK issue для Cluster D — `gtk_accessible_value_unref(self=0x7fff00000000)` SIGSEGV у
`gtk_at_context_finalize` під час teardown widget-а. Ескалює до SIGABRT під `MALLOC_CHECK_=3`
з `corrupted size vs. prev_size`.

Прикріплено: `follow-ups/gtk-cluster-d/`:

- `crash_2026-04-25_17-57_1.txt` (canonical SIGSEGV, distro 50.0, GtkBox parent),
- `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt` (Flatpak Nightly Vulkan, тa сама сигнатура),
- `nautilus-flatpak-cairo-2026-04-26_14-16-31.txt` (SIGABRT escalation, GtkBox parent),
- `nautilus-flatpak-cairo-2026-04-26_14-18-14.txt` (SIGABRT escalation, GtkListItemWidget parent).

---

## 4. Дотичні / схожі issue

### [`gtk#7968`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7968) — closed

Стосувалась `GtkDropTarget` finalization crash. Для нас релевантна **тільки частково**:
не покриває `GtkDragSource::drag_timeout` UAF, який ми побачили у Valgrind.

### [`nautilus#3992`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/3992)

Про teardown timing у `nautilus_grid_cell`. Наш `SourceAsymmetry.md` дотичний — те саме
місце в коді. Можливо, варто додати cross-link.

### [`gnome-software#2452`](https://gitlab.gnome.org/GNOME/gnome-software/-/issues/2452)

Інший проект із тим самим Cluster D leaf (`gtk_accessible_value_unref(self=0x7fff…)`).
Сильний аргумент, що це GTK-side bug, не Nautilus-specific.

### [`gnome-control-center#2704`](https://gitlab.gnome.org/GNOME/gnome-control-center/-/issues/2704)

Те саме — Cluster D на іншому проекті.

### [`gtk#6548`](https://gitlab.gnome.org/GNOME/gtk/-/issues/6548)

Старіша GTK issue, яка містить часткові деталі про
`GtkAccessibleAttributeSet`/`GtkAccessibleValue` lifecycle. Її варто згадувати поруч із Cluster D у `gtk#7910` / `nautilus#4035` (gtk#8173 закрита).

### Ubuntu / Launchpad — [bug #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297)

- **Пакет nautilus (Ubuntu)**: статус **New**; **wayland (Ubuntu)**: **Invalid** (після аналізу triager-а — не окремий баг Wayland).
- **Оновлений summary** (через Launchpad UI): акцент на часті креші при **швидкій навігації** по деревах каталогів на 26.04 (раніше в заголовку фігурували move/copy як перша гіпотеза).
- **Remote bug watches**: автоматичне відстеження [nautilus#4035](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035) і [gtk#7910](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910).
- **Що додано в діалозі** (скорочено): Memtest86+ без помилок; `dpkg -V`; скрін `GSK_RENDERER=cairo`; zip з покращеними backtrace (`crash_2026-04-25_2.zip`, `_3.zip`); пояснення патерну UAF / `nautilus-grid-cell` (з AI-чернетки — користувач явно маркував переклад); репро **List View або Grid View**, стрибки по дереву **без** копіювання/переміщення; архів листингів `listings_dirs_files_2026-04-26_1.zip`; `follow-up-2.zip` для Daniel; посилання на GitLab notes вище; згадка [CONTRIBUTING.md#ai-contribution-policy](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy) і готовність допомогти з фіксом до **26.04.1** ([коментар #29](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297/comments/29)).
- **Від triager-а (Daniel Tang)**: підтвердження дослідження на gitlab.gnome.org, уточнення щодо Cairo, прохання про скрипт дерева для репро (користувач обіцяв листинги — виконано в #26).

---

## 5. Ще не подано

### `GtkDragSource::drag_timeout` UAF

Знайдено Valgrind-ом (`nautilus-vg_1st_report.log`):

```
==XXXXX== Invalid read of size 8
==XXXXX==    at ... gtk_drag_source_drag_timeout_cb ...
==XXXXX==    by ... g_source_callback_unref ...
==XXXXX==    by ... g_source_destroy_internal ...
==XXXXX==  Address 0x... is 0 bytes inside a block of size 80 free'd
==XXXXX==    at ... __GI___libc_free ...
==XXXXX==    by ... gtk_drag_source_finalize ...
```

Тобто:

1. Користувач почав drag.
2. `GtkDragSource` запускає timeout через `g_timeout_add(...)`.
3. Drag відмінений / контролер забрано до того, як timeout встиг відстрелити.
4. `gtk_drag_source_finalize` звільнив контролер.
5. Через ~N мс GLib викликає `gtk_drag_source_drag_timeout_cb` на звільненому об'єкті → SEGV.

**Фікс**: cancel timeout у `gtk_drag_source_finalize` через
`g_clear_handle_id (&priv->drag_timeout, g_source_remove)`. Або тримати strong ref на
контролер у `g_timeout_add_full(..., (GDestroyNotify) g_object_unref)`.

Чернетка GTK issue готова, не запостена. Файли для аттачменту:
`old_nautilus_crashes_5/nautilus-vg_1st_report.log`, `nautilus-vg_2nd.log`.

---

## 6. Hint для тригерування

Якщо upstream-розробник захоче відтворити:

- Wayland-сесія (X11 не тестували, але Cluster E специфічний до Wayland-proxy lifecycle).
- GSK_RENDERER=cairo, щоб виключити GPU/Mesa.
- MALLOC_CHECK_=3 + MALLOC_PERTURB_=85 — ескалює UAF до видимого збою.
- Тека з 800+ піддиректоріями і 4500+ файлами (наш `dir_listings_1/` як approx-набір).
- Швидка навігація в режимах **Grid View** та **List View** лише по дереву тек (без DnD) — див. [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297) #21–22.

---

## 7. Ключові цитати з логів для cross-reference

Якщо upstream питатиме "де у логах прямий збіг з gtk#7910":

```
(org.gnome.Nautilus:9262): GLib-GObject-CRITICAL **: 10:39:11.791:
  ../../../gobject/gclosure.c:830: unable to remove uninstalled invalidation notifier:
  0x7ffff7d80e20 (0x555556679670)
```

(`crash_2026-04-25_10-42_1.txt`, `crash_2026-04-25_19-13_1.txt`).

```
GLib-GObject-CRITICAL **: g_signal_handlers_destroy: assertion '(handler != NULL)' failed
```

— canonical asertion, видно у `nautilus-flatpak-gl-2026-04-25_22-14-22.txt`.

Для Cluster D (логи в `follow-ups/gtk-cluster-d/`; окрема issue gtk#8173 закрита):

```
#0  gtk_accessible_value_unref (self=0x7fff00000000) at gtkaccessiblevalue.c:119
#1  gtk_accessible_attribute_set_free
#2  g_atomic_rc_box_release_full
#3  gtk_accessible_attribute_set_unref
#4  gtk_at_context_finalize
```

— `crash_2026-04-25_17-57_1.txt`, `nautilus-flatpak-vulkan-2026-04-25_22-08-37.txt`.
