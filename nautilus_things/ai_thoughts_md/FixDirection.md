# FixDirection.md — Гіпотези фіксу

## 1. Nautilus-side (швидкі експерименти)

### 1.1. Перенести `g_clear_object(&self->item_signal_group)` *до* `gtk_widget_dispose_template`

```diff
 static void
 nautilus_grid_cell_dispose (GObject *object)
 {
     NautilusGridCell *self = (NautilusGridCell *) object;

-    gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_GRID_CELL);
     g_clear_object (&self->item_signal_group);
+    gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_GRID_CELL);

     G_OBJECT_CLASS (nautilus_grid_cell_parent_class)->dispose (object);
 }
```

**Очікувано прибирає**: Cluster B, B′ через `nautilus_grid_cell_dispose` шлях.
**Не зачіпає**: Cluster D (accessibility), Cluster E (Wayland), Cluster B′ через
`gtk_list_item_widget_teardown_object` (бо там teardown — на стороні GTK, не Nautilus).

### 1.2. Альтернатива — `g_signal_group_set_target(NULL)` перед dispose_template

```diff
 static void
 nautilus_grid_cell_dispose (GObject *object)
 {
     NautilusGridCell *self = (NautilusGridCell *) object;

+    if (self->item_signal_group != NULL)
+        g_signal_group_set_target (self->item_signal_group, NULL);
     gtk_widget_dispose_template (GTK_WIDGET (self), NAUTILUS_TYPE_GRID_CELL);
     g_clear_object (&self->item_signal_group);

     G_OBJECT_CLASS (nautilus_grid_cell_parent_class)->dispose (object);
 }
```

М'якший варіант: signal_group залишається живий, але вже не forwardить нічого. Якщо
причина — emit `notify::item` через signal_group — це обриває race.

### 1.3. У `name_cell` — паралельно перенести у `dispose`

(Якщо 1.1/1.2 спрацює, симетрично змінити `nautilus_name_cell_dispose`.)

---

## 2. GTK-side (фундаментальніші)

### 2.1. `gtk_widget_dispose_template` — двофазний teardown

```c
/* Phase 1: run_dispose() on all auto-children, але без unref */
gtk_widget_class_for_each_template_child (..., g_object_run_dispose);

/* Phase 2: unref all auto-children */
gtk_widget_class_for_each_template_child (..., g_object_unref);
```

Це гарантує, що ВСІ діти спочатку розіб'ють свої bindings/expressions і signal-handlers,
і тільки потім фіналізуються. Race усувається на рівні GTK.

### 2.2. `gtk_at_context_dispose` (а не finalize)

```c
static void
gtk_at_context_dispose (GObject *gobject)
{
  GtkATContext *self = GTK_AT_CONTEXT (gobject);

  /* Очистити attribute_set-и до finalize, поки інші системи ще живі. */
  g_clear_pointer (&self->properties, gtk_accessible_attribute_set_unref);
  g_clear_pointer (&self->relations,  gtk_accessible_attribute_set_unref);
  g_clear_pointer (&self->states,     gtk_accessible_attribute_set_unref);

  G_OBJECT_CLASS (gtk_at_context_parent_class)->dispose (gobject);
}
```

Або, що більш правильно, у `gtk_accessible_attribute_set_free` додати safety check
проти poisoned value pointers:

```c
for (int i = 0; i < self->n_attributes; i++)
  {
    GtkAccessibleValue *v = self->attribute_values[i];
    if (v == NULL || ((uintptr_t) v < 0x10000) || ((uintptr_t) v >> 48) != 0)
      continue;  /* skip poisoned slots */
    gtk_accessible_value_unref (v);
  }
```

(Не "правильний" фікс, а defensive guard — корінь треба шукати в тому, **хто** записує
0x7fff00000000 у `attribute_values[i]`.)

### 2.3. `GtkDragSource::drag_timeout` — cancel у finalize

```diff
 static void
 gtk_drag_source_finalize (GObject *object)
 {
   GtkDragSource *self = GTK_DRAG_SOURCE (object);

+  g_clear_handle_id (&self->drag_timeout, g_source_remove);
   g_clear_object (&self->content);
   ...
 }
```

Або, що більш hands-off, при `g_timeout_add` тримати strong ref:

```diff
-  self->drag_timeout = g_timeout_add (DRAG_TIMEOUT_MS, drag_timeout_cb, self);
+  self->drag_timeout = g_timeout_add_full (G_PRIORITY_DEFAULT,
+                                           DRAG_TIMEOUT_MS,
+                                           drag_timeout_cb,
+                                           g_object_ref (self),
+                                           g_object_unref);
```

---

## 3. Експерименти, що можна провести локально

| # | Експеримент | Очікуваний результат |
|---|-------------|----------------------|
| 1 | `GTK_A11Y=none` для distro Nautilus | Cluster D зникає; B/E залишаються |
| 2 | `XDG_SESSION_TYPE=x11` | Cluster E зникає; B/D залишаються |
| 3 | Patched-build Nautilus з 1.1 | Cluster B (через grid_cell) зникає; B (через name_cell) залишається |
| 4 | Patched-build з 1.1 + 1.3 | Cluster B (повністю через cells) зникає; залишається B′ через `gtk_list_item_widget_teardown_object` |
| 5 | Patched GTK4 з 2.1 | Cluster A/B/B′/B″ зникають; D і E залишаються |
| 6 | Patched GTK4 з 2.2 | Cluster D зникає (або defensive guard блокує SEGV) |
| 7 | Patched GTK4 з 2.3 | `GtkDragSource` UAF зникає (видно по Valgrind) |

---

## 4. Що варто **уникати**

- ☒ "Просто додати `g_object_run_dispose` у `nautilus_grid_cell_dispose`" — це лише
  переносить проблему на крок раніше.
- ☒ Defensive `if (closure->ref_count > 0)` перевірки в Nautilus — це не Nautilus-side
  проблема, обхід відверне діагностику.
- ☒ Виключати accessibility через downstream patches — Cluster D треба фіксити в GTK.

---

## 5. Робочі гіпотези про root-cause

1. **Race у emit `notify::*` між `gtk_widget_dispose_template` і `g_clear_object(item_signal_group)`**.
   Якщо це підтвердиться (експеримент #3), nautilus-side fix 1.1 закриває Cluster B/B′.

2. **Bug у `gtk_at_context_finalize` ordering**: attribute_set-и фіналізуються коли деякі
   GtkAccessibleValue-and parent widget уже звільнив references. Потрібен GTK-side fix.

3. **Bug у `GtkSignalListItemFactory::teardown` вікна**: `gtk_list_item_widget_teardown_object`
   викликає `gtk_list_item_do_notify` (емітить `notify::position`/`notify::selected`) на
   item, який уже частково deconstructed (через попередній `clear_factory`). Race
   `do_notify` ↔ teardown.

4. **GtkDragSource standalone bug** — незалежна, але того ж класу UAF.

5. **GtkPropertyExpression watch lifetime** — race між `expr_notify_cb` і
   `watch_destroy_closure`. Можливо, треба робити invalidation atomic-ом, а не
   "first call invalidate, потім notify-er".
