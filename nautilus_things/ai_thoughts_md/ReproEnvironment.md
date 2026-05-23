# ReproEnvironment.md — Середовище і кроки репро

## 1. Хост

| Параметр | Значення |
|----------|----------|
| OS | Ubuntu 26.04 |
| Kernel | `7.0.0-14-generic` (PREEMPT_DYNAMIC, x86_64) |
| Сесія | Wayland |
| GPU | AMD Radeon (Mesa, `libvulkan_radeon.so`, `libEGL_mesa.so.0`) |
| CPU | AMD Ryzen 7 7735HS with Radeon Graphics |
| RAM | (Memtest86+ ≥10h — без помилок) |
| Файлова система | ext4 |

## 2. Тести Nautilus

| Конфіг | Версія | GTK | Note |
|--------|--------|-----|------|
| Distro | Nautilus 50.0 | GTK4 4.20.x | system `/usr/bin/nautilus` |
| Flatpak Nightly | Nautilus 51.alpha | GTK4 master | `org.gnome.Nautilus.Devel//master` |

## 3. Дані для репро

Тека `toNewer_26.04_2/`:

- ~872 директорії (з пробілами і Cyrillic в іменах);
- ~4510 файлів;
- розподіл за розширеннями:

| Розширення | К-сть |
|------------|-------|
| `.json` | 383 |
| `.log`  | 322 |
| `.txt`  | 116 |
| `.png`  |  75 |
| `.md`   |  57 |
| `.hyb`  |  52 |
| `.old`  |  37 |
| `.html` |  27 |
| `.js`   |  21 |
| `.pb`   |  17 |

(Повні листинги — у `/home/denixx/projects/nautilus_things/dir_listings_1/`.)

Топ-рівневі піддерева у тестовій теці:

- `.cache/google-chrome/Default/Cache/...`, `.cache/google-chrome/Default/Code Cache/...`
- `.config/Cursor/...`, `.cursor/...`
- `cursor_ide/{project_1, project_2, project_3, project_4, project_6}/...`
- `gpu-goodboot{26,27}-u2604/`
- `new_dmcub_1/`
- `old_1_working/`
- `.ssh/`

## 4. Тригер

1. **Лише навігація** (зафіксовано на [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297),
   коментарі #21–22): **List View** або **Grid View** — швидкі стрибки по дереву каталогів у
   тій самій великій структурі, **без** копіювання чи переміщення файлів. Це відповідає
   оновленому summary бага на Ubuntu.

2. **Архів логів**: частина файлів у `crashes_dir/` / `follow-ups/` зібрана під час ранніх
   спроб із DnD / cut-paste / «Move to» — той самий шлях оновлення моделі й teardown рядків,
   але для репро й опису upstream **достатньо** п.1.

Креш фіксується **під час** або одразу після фіналізації списку елементів в активному
вікні, тобто на черговому `items-changed` від `GtkSortListModel` →
`gtk_list_item_manager_model_items_changed_cb`.

## 5. Env-змінні (distro Nautilus 50.0)

```bash
# Перед запуском
nautilus -q                    # коректне завершення наявного інстансу
pkill -x nautilus              # переконатися, що процесу нема

export G_SLICE=always-malloc
export MALLOC_CHECK_=3
export MALLOC_PERTURB_=85
export GSK_RENDERER=cairo      # (або gl, vulkan)
export G_DEBUG=fatal-criticals,fatal-warnings,gc-friendly

gdb -ex 'set logging enabled on' \
    -ex 'set logging file /tmp/nautilus-gdb.log' \
    -ex 'run' \
    -ex 'thread apply all bt full' \
    -ex 'bt full' \
    --args /usr/bin/nautilus
```

Після крешу:

```bash
gdb> info shared
gdb> bt full
gdb> thread apply all bt
gdb> quit
```

## 6. Env-змінні (Flatpak Nightly)

```bash
flatpak install -y flathub-beta org.gnome.Nautilus.Devel//master
flatpak install -y flathub-beta org.gnome.Sdk.Debug//master org.gnome.Platform//master

flatpak kill org.gnome.Nautilus.Devel || true

flatpak run --command=bash --devel \
  --env=G_SLICE=always-malloc \
  --env=MALLOC_CHECK_=3 \
  --env=MALLOC_PERTURB_=85 \
  --env=GSK_RENDERER=cairo \
  org.gnome.Nautilus.Devel//master <<'EOSH'
gdb -ex 'set logging enabled on' \
    -ex 'set logging file /tmp/nautilus-flatpak.log' \
    -ex 'run' \
    --args /app/bin/org.gnome.Nautilus.Devel
EOSH
```

> Примітка: під Flatpak `G_DEBUG=fatal-criticals,fatal-warnings` забиває старт ще
> до показу вікна (через innocuous CRITICAL у nightly). Тому в Flatpak ми його не вмикали.

> Примітка 2: `GSK_RENDERER=cairo` повністю прибирає GPU-stack (Vulkan/GL/Mesa) з гри.
> Креш все одно відтворюється — це сильний доказ, що Mesa/GPU не винні.

## 7. Чому Cluster E видно і **без** MALLOC_*

Файл `nautilus-flatpak-vulkan-nomalloc-22-00-19.txt` — Flatpak Vulkan, без `MALLOC_CHECK_`,
без `MALLOC_PERTURB_`. Креш — Cluster E (`dispatch_event(proxy=0x555500000000)`). Тут
`0x555500000000` — це **реальний** heap-адрес у glibc-shaped арені (старша частина
завжди `0x55XX_XXXX_XXXX` для PIE+ASLR), просто молодші 4 байти випадково = 0. Це не
наш poison, а **валідний** UAF-симптом (Wayland proxy реально був звільнений і його heap chunk
повторно використано glibc-ом, але з нульовим payload-ом).

Кернельний slogan `kernel: nautilus[…]: segfault at <heap_high>00000028 in libwayland-client.so` зі
старих journal-логів (до підключення MALLOC-діагностики) теж підтверджує, що Cluster E
з'явився ще до того, як ми торкнулися env-змінних.

## 8. Що пробували, що **не** допомогло

- ☑ Memtest86+ ≥10h — RAM не винен.
- ☑ Зміна рендерера: Vulkan → GL → Cairo — **не допомагає**.
- ☑ Перехід на Flatpak Nightly (свіжий GTK4 master) — **не допомагає**, креш стабільно.
- ☑ Запуск Nautilus у "List view" замість "Grid view" — **не допомагає** (див.
  `crash_2026-04-25_18-29_1.txt` із `gtk_column_view_row_widget_teardown_object`).

## 9. Не пробували (можна)

- [ ] `GTK_A11Y=none` — відключити accessibility, перевірити Cluster D.
- [ ] `XDG_SESSION_TYPE=x11` — перевірити Cluster E.
- [ ] Disable `GtkSortListModel` сортування — перевірити, чи зникає B′ через PropertyExpression.
- [ ] Patched-build Nautilus з перенесеним `g_clear_object(item_signal_group)` у finalize.
