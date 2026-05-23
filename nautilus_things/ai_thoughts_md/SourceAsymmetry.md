# SourceAsymmetry.md — Асиметрія `nautilus-grid-cell.c` vs `nautilus-name-cell.c`

Доступні джерела:

- `/home/denixx/nautilus-50.0/src/` — стабільна 50.0;
- `/home/denixx/nautilus_git_1/src/` — git master / 51.alpha (Flatpak Nightly).

У обох версіях код, що нас цікавить, **ідентичний** (нумерація рядків трохи різниться).

---

## 1. `NautilusGridCell` — `_init` + `_dispose` (master)

```c
/* nautilus_git_1/src/nautilus-grid-cell.c */

/* _init() — встановлюються signal-group і binding */
static void
nautilus_grid_cell_init (NautilusGridCell *self)
{
    gtk_widget_init_template (GTK_WIDGET (self));

    self->item_signal_group = g_signal_group_new (NAUTILUS_TYPE_VIEW_ITEM);
    g_signal_group_connect_swapped (self->item_signal_group, "notify::is-cut",
                                    G_CALLBACK (on_item_is_cut_changed), self);
    g_signal_group_connect_swapped (self->item_signal_group, "file-changed",
                                    G_CALLBACK (on_file_changed), self);

    g_object_bind_property (self, "item",
                            self->item_signal_group, "target",
                            G_BINDING_SYNC_CREATE);

    /* Plus a few more g_object_bind_property() calls between self->item and child widgets. */
}

/* _dispose() — порядок звільнення */
static void
nautilus_grid_cell_dispose (GObject *object)
{
    NautilusGridCell *self = (NautilusGridCell *) object;

    gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_GRID_CELL);   /* (1) */
    g_clear_object (&self->item_signal_group);                                  /* (2) */

    G_OBJECT_CLASS (nautilus_grid_cell_parent_class)->dispose (object);
}
```

(1) звільняє auto-children template-у (тобто всі віджети, що були прив'язані через
`gtk_widget_class_bind_template_child`). Кожен auto-child може мати свої `GBinding`-и, що
посилаються на `self`, і свої `GtkPropertyExpression watch`-и.

(2) лише після цього звільняє `item_signal_group` (який forwardить `notify::is-cut` і
`file-changed` на `self->item`).

**Проблема порядку**: під час (1) GLib фіналізує дочірні віджети в певному порядку. Якщо
у проміжку якийсь з них (наприклад `GtkLabel` для назви файлу, що зв'язаний із
`self->item:display-name` через `g_object_bind_property`) у своєму dispose-у викличе
emit `notify` (що буває в `g_object_notify_queue_thaw`) — цей `notify` може дочитатися до
`self->item_signal_group`, який ще живий, але **`item_signal_group->target` уже
очищується** parent-ом, що стартував `g_object_unref` ланцюга.

---

## 2. `NautilusNameCell` — `_dispose` + `_finalize` (master)

```c
/* nautilus_git_1/src/nautilus-name-cell.c */

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
    g_clear_object (&self->item_signal_group);                                  /* пізніше! */
    g_clear_object (&self->file_path_base_location);
    G_OBJECT_CLASS (nautilus_name_cell_parent_class)->finalize (object);
}
```

Тут `g_clear_object(&self->item_signal_group)` відбувається у **finalize**, тобто
**значно пізніше**, ніж dispose дочірніх віджетів. У grid_cell — у dispose, відразу після.

---

## 3. Чому асиметрія матерія?

Дочірні віджети `*_cell` тримають callback-функції, прив'язані до `self->item` через
`g_object_bind_property` і `GtkPropertyExpression watch`. Коли `gtk_widget_dispose_template`
розриває auto-children, GLib фіналізує діти і викликає їх deregister-сигнали в `self->item`.
**`self->item` залишається валідним, бо grid_cell`/`name_cell ще не очищували `item`-власність**,
але `item_signal_group->target` міг бути перезаписаний раніше (в `gtk_widget_dispose_template`,
бо binding `(self, "item")` ↔ `(item_signal_group, "target")` обривається через
emit `notify::item` під час teardown).

Тобто у проміжку `dispose_template` ↔ `g_clear_object(item_signal_group)` (для grid_cell —
ну майже жодного), `self->item_signal_group` має `target=NULL`, але сам signal_group ще
підписаний на handler-list-и в `item`. Якщо в цей момент відбувається emit `notify` через
`g_object_notify_queue_thaw` (внутрішній фрейм у dispose_template), invalidation walking
по handler-list-у може зустріти already-invalidated closure → Cluster A/B/B′.

Crash-логи `crash_2026-04-25_18-05_1.txt` (Cluster B′ у `gtk_list_item_widget_teardown_object`)
прямо демонструють цей шлях (`signal_emit_unlocked_R(notify, detail=965 ≈ "position")` ←
`gtk_list_item_do_notify` ← `gtk_list_item_widget_teardown_object`).

---

## 4. Дочірні bindings (master)

```c
/* grid_cell, master, ~рядки 510–530 */
g_object_bind_property (self, "item",
                        self->fixed_height_box, "item",
                        G_BINDING_SYNC_CREATE);
g_object_bind_property (self, "item",
                        self->snapshot_box, "item",
                        G_BINDING_SYNC_CREATE);
```

(У реальному коді там кілька десятків рядків; деталі див. у `nautilus-grid-cell.c:506+`.)

```c
/* name_cell, master, ~рядки 367–372 */
g_object_bind_property (self, "item",
                        self->fixed_height_box, "item",
                        G_BINDING_SYNC_CREATE);
```

Тобто кожна `*_cell` тримає кілька `GBinding`-ів, що прив'язують власну `item` до `item`
дочірніх. У момент teardown це означає каскад:

1. `gtk_widget_dispose_template(self)` — починає dispose-ити auto-children.
2. Auto-child`#i` фіналізується, викликає `g_object_unref(child->item)` (бо binding впав).
3. `child->item` — це той самий `self->item`, але з ref +1 від binding. Один ref тепер мінус.
4. Coreзалогований handler `notify::item` на `self->item` емітить `notify` (вирівнювання).
5. На `notify::item` слухає... що? `self->item_signal_group` через bind_property
   `(self, "item") → (item_signal_group, "target")`. Він уже `target=NULL` (binding порвав
   зв'язок раніше або дисконнектиться зараз).
6. Якщо crud invalidation race trigger-ить — Cluster A/B/B′.

Це гіпотетично: щоб довести, треба patched-build з логуванням переходу `item_signal_group->target`
між dispose-template ↔ g_clear_object.

---

## 5. Висновки і пропоновані експерименти

1. **Перенести `g_clear_object(&self->item_signal_group)` *до* `gtk_widget_dispose_template`** —
   щоб `item_signal_group` був гарантовано звільнений ще до того, як діти стартують
   teardown ланцюжок. Або:

2. **`g_signal_group_set_target(self->item_signal_group, NULL)`** у `dispose()` *перед*
   `gtk_widget_dispose_template`. Це обриває forward-ні сигнали без знесення signal_group.

3. **Перенести clear у `finalize`** (як зроблено у `name_cell`) — але це не вирішує проблему,
   бо `name_cell` теж падає у тих самих кластерах.

Пункти 1+2 потенційно прибирають Cluster B/B′ для grid_cell. Не вирішують Cluster D і E.
Cluster D — це accessibility (GtkAtSpiContext) UAF, який не зачіпається `item_signal_group`.
Cluster E — Wayland, незв'язано з cell teardown (хоча таймінг той самий, бо викликається
з того ж items-changed).

---

## 6. Що ще треба перевірити в коді

- [ ] `NautilusViewItem.dispose/finalize` — як саме він звільняє свої внутрішні signals?
- [ ] `NautilusViewModel::items-changed` емітер — чи він тримає ref на старі items до
  завершення емісії, чи ні?
- [ ] `gtk_list_item_change_finish` — там стоїть hash_table_destroy, який знищує всі
  recycled items одразу. Якщо у проміжку GTK ще обробляє pending notifies — race.
- [ ] У `nautilus-list-view.c`/`nautilus-grid-view.c` — як ініціалізується
  `GtkColumnView`/`GtkGridView` з тим самим model? Чи правильно cancel-иться попередній?
