# highlevelContext.md — стартовий контекст для нової сесії

Прочитай цей файл повністю — після цього в тебе має бути повне розуміння,
чим займається ця "проєктна тека" і куди дивитися далі. Усі посилання нижче —
відносно `/home/denixx/projects/nautilus_things/`, якщо не вказано інше.

---

## 1. Хто, що, навіщо

Користувач — **denixx** (або **dnxb42**), на Ubuntu 26.04, Wayland, AMD Ryzen 7 7735HS + Radeon (Mesa).  
Його GNOME Files (Nautilus) **стабільно падає** при дуже швидкій навігації
(List View або Grid View) по великому дереву каталогів — у теці зі ~870 піддиректоріями і ~4500 файлами
(див. також [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297)).

Перевірка пам'яті за допомогою Memtest86+ трохи більше 4 годин **без помилок** — RAM не винен.
Креш відтворюється на:

- distro Nautilus 50.0 (Ubuntu 26.04, system GTK4 4.20.x, GLib 2.86.x);
- Flatpak Nightly `org.gnome.Nautilus.Devel//master` (Nautilus 51.alpha + GTK4 master);
- усіх `GSK_RENDERER`-ах (Vulkan / GL / Cairo) — отже, **GPU/Mesa не винні**;
- з і без `MALLOC_CHECK_`/`MALLOC_PERTURB_` — отже, MALLOC-діагностика не "створює" креш,
а лише ескалює UAF до видимого збою раніше.

Ціль користувача — допомагати GNOME-розробникам по максимуму для виправлення падінь nautilus.

---

## 2. Що уже зроблено (станом на 2026-05-02)

- ☑ Зібрано 32 креш-логи + 2 Valgrind звіти + 2 journal-логи + 2 листинги тестового
дерева → усе лежить у `crashes_dir/`, `dir_listings_1/`, `follow-ups/`.
- ☑ Кластеризовано креші у 8 родин (A / A′ / B / B′ / B″ / B-jump / C / D / E).
- ☑ Перевірено source code asymmetry в `nautilus-grid-cell.c` vs `nautilus-name-cell.c`
(50.0 і master — однаково).
- ☑ Опубліковано в `[nautilus#4035](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035)`:
основний коментар + два follow-up з тарболами (`crashes_to_nautilus_1.zip`,
`follow-up-1`, `follow-up-2`).
- ☑ Опубліковано в `[gtk#7910](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910)`:
деталізований коментар з нашими crash-логами Cluster A/B/B″ (тарбол `gtk-7910-followup`).
- ☑ Відкрито нову `[gtk#8173](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173)` для
Cluster D (тарбол `gtk-cluster-d`) — згодом **закрито** після політики human-only.
- ☑ Написано повний звіт у `ai_thoughts_md/` (цю теку).
- ☑ Узгоджено з [політикою human-only GTK](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/CONTRIBUTING.md#ai-contribution-policy): коментарі в GitLab переписані / додані **вручну**; `[gtk#8173](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173)` закрита; у `[gtk#7910](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910)` є людський follow-up ([note_2744852](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910#note_2744852)).
- ☑ На [Launchpad: ubuntu / nautilus #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297): оновлено summary під **швидку навігацію** по дереву (перші описи на Launchpad помилково вказували на move/copy як головну гіпотезу), додано посилання на GitLab ([note_2744445](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744445) як «все цінне», [note_2744838](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744838) — Cairo), аттачі (`listings_dirs_files_2026-04-26_1.zip`, `follow-up-2.zip`, додаткові zip крешів), **remote bug watches** на `nautilus#4035` і `gtk#7910`; задача **wayland (Ubuntu)** переведена в **Invalid**; діалог з Daniel Tang (memtest, `GSK_RENDERER=cairo`, скрипт дерева).
- ☑ **2026-05-02**: тепер доступний повний source GTK4 master у `/home/denixx/gtk_git_1/` (раніше було лише за backtrace-ами). Це дозволяє цитувати конкретні рядки і робити мінімально-інвазивні diff-проєкти патчів.
- ☑ **2026-05-02**: оновлено вміст `/home/denixx/nautilus_git_1/` свіжими комітами master (HEAD: `7cacc72c2 bookmark-list: Fix off-by-one insertion of bookmarks`). Перевірено: `nautilus_grid_cell_dispose` / `nautilus_name_cell_dispose` всё ще ідентичні версії 50.0 — асиметрія не виправлена в апстрімі.

## Що **не** зроблено (можливі наступні кроки)

- ☐ Моніторити відповіді в GitLab і Launchpad; нові аттачі / коментарі — лише власним текстом (без AI-копіпасту в GNOME).
- ☐ За потреби — окрема GTK issue для Valgrind `GtkDragSource` UAF: `Invalid write of size 4` у `drag_timeout (gtkdragsource.c:276)` після `gtk_widget_remove_controller` (нова чернетка з посиланнями на актуальні рядки `/home/denixx/gtk_git_1/gtk/gtkdragsource.c` готується; деталі — у `UpstreamIssues.md`, лог — `crashes_dir/old_nautilus_crashes_5/nautilus-vg_2nd.log:363`).

---

## 3. Структура теки `nautilus_things/`

```
nautilus_things/
├── ai_thoughts_md/             ← НОТАТКИ (читай у такому порядку)
│   ├── highlevelContext.md     ← цей файл (старт)
│   ├── AllFindings.md          ← повний звіт (490 рядків) ← ОБОВ'ЯЗКОВО ПРОЧИТАТИ
│   ├── Clusters.md             ← таксономія A/A′/B/B′/B″/B-jump/C/D/E + top frames
│   ├── PerFile.md              ← таблиця "файл → кластер → сигнал → лиф" для всіх 40 файлів
│   ├── SourceAsymmetry.md      ← код-аналіз grid_cell vs name_cell
│   ├── ReproEnvironment.md     ← hardware/OS/env-змінні/gdb-команди
│   ├── UpstreamIssues.md       ← nautilus#4035, gtk#7910, gtk#8173, Launchpad #2150297 + дотичні
│   └── FixDirection.md         ← конкретні diff-патчі для Nautilus і GTK + експерименти
│
├── crashes_dir/                ← усі вихідні логи
│   ├── crashes_to_nautilus_1/      (3 файли — основний коментар nautilus#4035)
│   ├── old_nautilus_crashes_1/     (3 .txt + 2 journalctl-логи; найраніші)
│   ├── old_nautilus_crashes_2/     (6 файлів)
│   ├── old_nautilus_crashes_3/     (4 файли)
│   ├── old_nautilus_crashes_4/     (3 файли, серед них direct gtk#7910 hit)
│   └── old_nautilus_crashes_5/     (18 файлів — distro + Flatpak Nightly + Valgrind)
│
├── dir_listings_1/             ← фотографія тестового дерева
│   ├── dirsListing_{1,2}.txt   (872 рядки кожен — повний `ls -liaR` директорій)
│   └── filesListing_{1,2}.txt  (4510 рядків кожен — повний `ls -liaR` файлів)
│
├── follow-ups/                 ← групування файлів для тарбол-аттачментів upstream
│   ├── crashes_to_nautilus_1.zip            (тарбол для основного коментаря)
│   ├── follow-up-1/                          (тарбол follow-up #1 у nautilus#4035)
│   ├── follow-up-2/                          (тарбол follow-up #2)
│   ├── gtk-7910-followup/                    (тарбол для gtk#7910)
│   └── gtk-cluster-d/                        (тарбол для gtk#8173)
│
└── scripts/                    ← робочі сніпети (env-змінні + how-to-run)
    ├── dbg_nautilus_1.txt                   (gdb-команда для distro nautilus з MALLOC_CHECK_=3)
    ├── running_flatpak_devel_1.txt          (`crash_run` shell-функція для Flatpak Nightly з Vulkan/GL/Cairo)
    └── valgrind_nautilus_1.txt              (Valgrind Memcheck команда — джерело nautilus-vg_2nd.log)
```

> Усі файли у `follow-ups/` — копії файлів із `crashes_dir/`. **Унікальних логів там немає.**

---

## 4. Доступний source code (поза `nautilus_things/`)

- **`/home/denixx/nautilus-50.0/`** — стабільна Nautilus 50.0 (та сама, що в системі). Не змінюється.
- **`/home/denixx/nautilus_git_1/`** — git master / 51.alpha (Flatpak Nightly).
  Перепулено 2026-05-02; HEAD: `7cacc72c2 bookmark-list: Fix off-by-one insertion of bookmarks`.
- **`/home/denixx/gtk_git_1/`** — git master GTK4. **Зʼявилося 2026-05-02.**
  HEAD: `b9f135a733 Merge branch 'wip/otte/for-main' into 'main'`.

Найважливіші файли в Nautilus:

- `src/nautilus-grid-cell.c` — `nautilus_grid_cell_dispose` на рядку 238 (50.0) / 271 (master).
- `src/nautilus-name-cell.c` — `nautilus_name_cell_dispose` + `nautilus_name_cell_finalize`
на рядках 373/383 (50.0) / 372/382 (master). **Асиметрія `g_clear_object(item_signal_group)` в dispose vs finalize досі не виправлена в апстрімі (станом на 2026-05-02).**
- `src/nautilus-view-model.c`, `nautilus-list-base.c`, `nautilus-grid-view.c`,
`nautilus-list-view.c` — тут живуть `GtkSortListModel`, `GtkColumnView`, `GtkGridView`
обгортки, що генерують `items-changed`, з якого починається весь cascade.
- `src/nautilus-list-base.c:844` (`setup_cell_common`) і `src/nautilus-grid-view.c:465` (`setup_cell`) — місце, де створюється `GtkDragSource` controller для recycled cell-у (видно в Valgrind-блоці аллокації UAF-нутого об'єкта в `nautilus-vg_2nd.log:431-432`).

Найважливіші файли в GTK4:

- `gtk/gtkdragsource.c` — `drag_timeout` (line 276), `gtk_drag_source_begin` (line 289), `gtk_drag_source_finalize` (line 192). **Реальна назва поля — `source->timeout_id` (не `priv->drag_timeout`); реальна назва функції — `drag_timeout` (не `gtk_drag_source_drag_timeout_cb`)**. Старі чернетки в `UpstreamIssues.md` / `FixDirection.md` використовують неточні імена з здогадки до отримання source-у — їх слід відрефакторити.
- `gtk/gtkwidget.c:11529` (`gtk_widget_dispose_template`), `gtkwidget.c:12009` (`gtk_widget_remove_controller`).
- `gtk/gtkaccessibleattributeset.c:85` (`gtk_accessible_attribute_set_free`), `gtk/gtkaccessiblevalue.c:119` (`gtk_accessible_value_unref`).
- `gtk/gtklistitemwidget.c:157` (`gtk_list_item_widget_teardown_object`), `gtk/gtkcolumnviewrowwidget.c:161` (`gtk_column_view_row_widget_teardown_object`).
- `gtk/gtkexpression.c:1299` (`gtk_property_expression_watch_destroy_closure`).

---

## 5. Корінь bug-у — TL;DR одним абзацем

**Race / use-after-free в GTK4/GLib teardown ordering**, який виникає коли
`gtk_list_item_manager_model_items_changed_cb` (тригериться від
`GtkSortListModel`/`NautilusViewModel::items-changed`) → `gtk_list_item_change_finish`
→ `g_hash_table_remove_all` зносить **усі** recycled-items одразу. Кожен recycled-item
тримає ланцюжок зв'язків `GBinding(self, "item") ↔ child->item` + `GtkPropertyExpression watch`-и

- `GSignalGroup::item_signal_group`, що мають backref-и. Під час cascade-каскаду
`g_object_unref` GLib встигає емітити `notify::*` на ще-живий `self->item`, а handler
закритих/`watch`-ів ходить уже по частково-звільненому handler-list-у. Що саме впаде
першим (closure / accessibility ctx / Wayland proxy / heap chunk) — визначає кластер
крешу. Симптоми:
- `g_closure_unref(closure=0x555500000000)` (Cluster B);
- `assertion failed: (handler != NULL)` у `invalid_closure_notify` (Cluster A, gtk#7910);
- `gtk_accessible_value_unref(self=0x7fff00000000)` (Cluster D; було gtk#8173, **закрито** — логи лишаються в `gtk-cluster-d/`);
- `dispatch_event(proxy=0x555500000000)` у `libwayland-client.so` (Cluster E);
- `corrupted size vs. prev_size` у `unlink_chunk`/`gtk_label_finalize` (Cluster C, fallout).

---

## 6. Як швидко "ввійти у тему" в новій сесії

Послідовність читання:

1. **Цей файл** (highlevelContext.md) — 5 хв.
2. `**AllFindings.md`** — 15 хв, повна картина.
3. `**Clusters.md`** — 10 хв, top frames для кожного кластера.
4. `**PerFile.md`** — як довідник; повертайся сюди, коли потрібно подивитися на конкретний файл.
5. `**SourceAsymmetry.md`**, `**UpstreamIssues.md`**, `**FixDirection.md**` — вибірково,
  залежно від того, що буде запитано.

Перевіряти конкретні crash-логи зручно так:

```bash
# top stack для всіх логів одразу
cd /home/denixx/projects/nautilus_things/crashes_dir
for f in $(find . -type f \( -name 'crash_*.txt' -o -name 'nautilus-flatpak-*.txt' \) | sort); do
  echo "===== ${f#./} ====="
  awk '/Program received signal|Program terminated/ {print "  SIG: " $0; next}
       /^#0[ \t]/ {sub(/^/, "  F0: "); print; exit}' "$f"
done
```

Або одиничний файл — `Read /home/denixx/projects/nautilus_things/crashes_dir/.../crash_*.txt`.

---

## 7. Найважливіші трекери (стан 2026-05-02)


| Issue / bug                                                                              | Стан                                                      | Примітки                                                                                                                                                                                                            |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[nautilus#4035](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035)`                 | відкрита                                                  | Після human-only — актуальні якорі: [note_2744445](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744445), [note_2744838](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035#note_2744838) (Cairo). |
| `[gtk#7910](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910)`                           | відкрита                                                  | Людський follow-up: [note_2744852](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910#note_2744852). Тарбол `gtk-7910-followup/` без змін у `follow-ups/`.                                                            |
| `[gtk#8173](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173)`                           | **закрита**                                               | Cluster D логи — `follow-ups/gtk-cluster-d/`; див. `UpstreamIssues.md`.                                                                                                                                             |
| [Launchpad **2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297)** | **nautilus (Ubuntu): New**; **wayland (Ubuntu): Invalid** | Remote watches → GitLab #4035 та #7910; деталі — `UpstreamIssues.md` → розділ 4 (Ubuntu / Launchpad).                                                                                                               |
| **GtkDragSource UAF** (`drag_timeout` write-after-free)                                  | не подано                                                 | Підтверджений Valgrind: `nautilus-vg_2nd.log:363-431`. Тепер є GTK source — точні рядки `gtk/gtkdragsource.c:{192,276,289,493}`. Готова мінімальна-діфф-чернетка фіксу. |


---

## 8. Загальні поради при роботі з цим контекстом

- Коли користувач каже "перевір такий-то файл" — спершу **класифікуй за кластером**
(див. `Clusters.md`), і тільки потім аналізуй детальніше. Це економить час — більшість
логів — дублі вже відомих сигнатур.
- Користувач говорить переважно **українською**, відповідай теж українською.
- Користувач уважний до якості markdown — перед публікацією upstream завжди перевіряй,
чи не зламана вкладеність code-fences.
- При складанні нових тарболів — користувач сам збирає `.zip`/`.tar.gz`, ти лише
даєш список файлів для архіву.
- При згадках процесора — **AMD Ryzen 7 7735HS with Radeon Graphics** (не плутати з іншими
моделями).
- Користувач знає upstream issue-tracker GitLab і вміє цитувати posts через `note=...`
ID — використовуй `https://gitlab.gnome.org/GNOME/<project>/-/issues/<N>#note_<id>` за потребою.
- При роботі з логами — ніколи не пиши `cat`/`head`/`tail`, користуйся `Read`/`Grep`/`Glob`.
- **Не модифікуй** файли в `crashes_dir/`, `dir_listings_1/`, `follow-ups/` — це вихідні
дані. Усі думки і висновки ідуть у `ai_thoughts_md/`.

