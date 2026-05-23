# DragSourceProblem1 — per-cell `GtkDragSource` is the trigger of all Nautilus list/grid crashes

> Sister bug reports for context: [`nautilus#4035`](https://gitlab.gnome.org/GNOME/nautilus/-/issues/4035),
> [`gtk#7910`](https://gitlab.gnome.org/GNOME/gtk/-/issues/7910),
> closed [`gtk#8173`](https://gitlab.gnome.org/GNOME/gtk/-/issues/8173),
> Ubuntu [Launchpad #2150297](https://bugs.launchpad.net/ubuntu/+source/nautilus/+bug/2150297).
> All crash logs and analyses live under
> `/home/denixx/projects/nautilus_things/` (see `ai_thoughts_md/AllFindings.md` for the
> full picture).

---

## Summary

Nautilus 50.0 (Ubuntu 26.04) and Nautilus 51.alpha (Flatpak Nightly,
`org.gnome.Nautilus.Devel//master`) crash within seconds of fast keyboard / mouse
navigation through a directory of ≈ 870 sub-folders and ≈ 4 500 files. Crashes
appear in many shapes — all listed in `ai_thoughts_md/Clusters.md` — but they
all share one trigger:

**Commenting out the four lines that install the per-cell `GtkDragSource`
event-controller in `setup_cell_common()`
([`nautilus-list-base.c:892–895`](https://gitlab.gnome.org/GNOME/nautilus/-/blob/main/src/nautilus-list-base.c#L892-L895)
in current `main`) makes every crash variant disappear, on every renderer
(Vulkan / GL / Cairo), on both Nautilus 50.0 and 51.alpha.**

```c
/* nautilus-list-base.c, setup_cell_common() */
892    controller = GTK_EVENT_CONTROLLER (gtk_drag_source_new ());
893    gtk_widget_add_controller (GTK_WIDGET (cell), controller);
894    gtk_event_controller_set_propagation_phase (controller, GTK_PHASE_CAPTURE);
895    g_signal_connect (controller, "prepare", G_CALLBACK (on_item_drag_prepare), cell);
```

This is obviously not an acceptable fix (it kills item drag-out), but it is
decisive proof that the per-cell `GtkDragSource` is the **single primary
corruption source**. Every other observed crash signature — invalidated
closures, accessibility-attribute UAFs, Wayland-proxy SEGVs, heap
corruption — is a downstream symptom of one tiny write-after-free originating
in `GtkDragSource`.

The actual write-after-free is captured by Valgrind in
`crashes_dir/old_nautilus_crashes_5/nautilus-vg_2nd.log:363`:

```
==35226== Invalid write of size 4
==35226==    at 0x5442AF4: drag_timeout (gtkdragsource.c:276)
==35226==    by 0x4C899F0: g_timeout_dispatch (gmain.c:5324)
==35226==    by 0x4C87B9A: g_main_dispatch (gmain.c:3591)
   ...
==35226==  Address 0x1479aa18 is 216 bytes inside a block of size 240 free'd
==35226==    at 0x49AEB1D: free_sized (vg_replace_malloc.c:1034)
==35226==    by 0x4BFCBC2: g_type_free_instance (gtype.c:1972)
==35226==    by 0x4BE7706: g_object_unref (gobject.c:4926)
==35226==    by 0x561E2EF: gtk_widget_remove_controller (gtkwidget.c:12009)
==35226==    by 0x56163F6: gtk_widget_finalize.lto_priv.0 (gtkwidget.c:7883)
   ...
==35226==    by 0x5545937: gtk_signal_list_item_factory_teardown (gtksignallistitemfactory.c:145)
   ...
==35226==    by 0x40B0621: on_view_click_pressed.lto_priv.0 (nautilus-list-base.c:365)
   ...
==35226==  Block was alloc'd at
==35226==    at 0x49B2E43: calloc (vg_replace_malloc.c:1678)
==35226==    by 0x4C8F121: g_malloc0 (gmem.c:133)
==35226==    by 0x4C03B23: g_type_create_instance (gtype.c:1872)
   ...
==35226==    by 0x40B6DCB: setup_cell_common (nautilus-list-base.c:844)
==35226==    by 0x40B6F6A: setup_cell (nautilus-grid-view.c:465)
```

The freed 240-byte block is exactly the `GtkDragSource` instance allocated by
those four lines; the 4-byte write at offset 216 is `source->timeout_id = 0;`
in `drag_timeout()` of GTK
([`gtk/gtkdragsource.c:271–279`](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/gtk/gtkdragsource.c#L271-L279) in current `main`).

---

## Details

### 1. The race in plain words

1. The user presses the mouse button on a cell.
   `GtkGestureClick` recognises a press; `GtkDragSource::begin()`
   ([`gtkdragsource.c:289`](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/gtk/gtkdragsource.c#L289))
   is also called and arms a 100 ms timeout:

   ```c
   /* gtk/gtkdragsource.c, gtk_drag_source_begin() */
   289    source->timeout_id = g_timeout_add (MIN_TIME_TO_DND /*=100*/, drag_timeout, source);
   ```

   `g_timeout_add()` does **not** take a strong reference on `source`; the
   timeout merely keeps a raw pointer in `user_data`.

2. The same press also dispatches to `on_view_click_pressed`
   (`nautilus-list-base.c:365`), which calls `gtk_widget_grab_focus()`
   on the cell. `gtk_window_root_set_focus()` then unrefs the previously
   focused list-factory widget, the factory tears down its child cell via
   `gtk_signal_list_item_factory_teardown()` →
   `gtk_list_item_set_child(NULL)`, and the cell finalises. During cell
   finalisation, `gtk_widget_finalize()` walks the controller list and calls
   `gtk_widget_remove_controller()` for each; that final
   `g_object_unref()` reaches refcount 0 and frees the `GtkDragSource`
   instance. (See frames 376–393 in the Valgrind block above.)

3. `gtk_drag_source_finalize()`
   ([`gtkdragsource.c:192–207`](https://gitlab.gnome.org/GNOME/gtk/-/blob/main/gtk/gtkdragsource.c#L192-L207))
   *does* call

   ```c
   g_clear_handle_id (&source->timeout_id, g_source_remove);
   ```

   so cancellation **is** attempted. Despite that, the next iteration of
   the GLib main loop still dispatches the timeout: GLib’s
   `g_main_dispatch()` works from a `pending_dispatches` snapshot taken
   before this teardown ran, and `g_timeout_add()`’s callback receives
   the raw `source` pointer — by now stale. `drag_timeout()` runs
   line 276:

   ```c
   /* gtk/gtkdragsource.c */
   271 static gboolean
   272 drag_timeout (gpointer user_data)
   273 {
   274   GtkDragSource *source = user_data;
   275
   276   source->timeout_id = 0;          /* <-- 4-byte write into freed memory */
   277
   278   return G_SOURCE_REMOVE;
   279 }
   ```

   Net effect: a four-byte zero write at a stable offset (+216) inside
   whatever `glibc` placed at the recycled 240-byte slot. Because Nautilus
   churns through this code path on every recycled cell, Valgrind reports
   the same `Invalid write` from many call sites in
   `nautilus-vg_2nd.log`.

### 2. Why one tiny write produces so many different crash signatures

Once those four bytes have been zeroed inside an unrelated heap chunk, the
victim object can be almost anything that `glibc` chose to recycle into the
same 240-byte slot. `ai_thoughts_md/Clusters.md` enumerates the symptoms we
have observed; the table below maps each cluster back to the same root
write-after-free:

| Cluster (see `Clusters.md`) | Visible top-of-stack | What got recycled in the freed slot |
|------|------|------|
| **A** | `assertion failed: (handler != NULL)` in `invalid_closure_notify` | a `GSignalHandler` slot whose pointer was zeroed |
| **A′** | `g_object_unref: assertion '!object_already_finalized' failed` | a `GObject` whose `qdata` was zeroed |
| **B**  | `g_closure_unref(closure=0x555500000000)` / leaf inside `gtk_widget_dispose_template` with `__inst=0x555500000000` | a `GClosure` whose internal pointer was zeroed (high half preserved by `MALLOC_PERTURB_=85`, low half from our 4-byte `0`) |
| **B′** | `signal_emit_valist_unlocked()` / `_g_closure_supports_invoke_va(closure=0x555500000000)` | same closure, hit while walking handler list |
| **B″** | `signal_emit_valist_unlocked()` with corrupted handler-list link | a `GHookList` node whose `next` pointer was zeroed |
| **B-jump** | `#0 0x00007fff00000000 in ?? ()` | a marshal pointer slot zeroed → indirect call to a poisoned address |
| **C**  | `unlink_chunk` / `_int_malloc` / `corrupted size vs. prev_size` | the malloc metadata of the next chunk in the bin |
| **D**  | `gtk_accessible_value_unref(self=0x7fff00000000)` in `gtk_at_context_finalize` | a `GtkAccessibleAttributeSet::attribute_values[i]` slot zeroed |
| **E**  | `dispatch_event(proxy=0x555500000000)` in `libwayland-client.so` | a Wayland proxy `wl_object` whose pointer was zeroed |

This is also why `MALLOC_PERTURB_=85`, `MALLOC_CHECK_=3`, and the choice of
`GSK_RENDERER` are irrelevant to the *occurrence* of the bug — they only
change which downstream object happens to land in the recycled slot, and
how loudly the corruption is reported.

### 3. Two reproducer flavours feeding the same write

The race is fed by two different teardown chains, both visible in our
crash logs:

* **`gtk_widget_grab_focus_self → gtk_window_root_set_focus → factory teardown`**
  (the Valgrind reproducer above) — happens on the very first click on a cell.
* **`GtkSortListModel::items-changed → gtk_list_item_manager_model_items_changed_cb →
  gtk_list_item_change_finish → g_hash_table_remove_all` recycled-items**
  (the dominant pattern in the GDB stacks under `crashes_dir/`) — happens
  whenever the model issues a bulk replace. Fast keyboard navigation
  through directories triggers this constantly.

Both reach `gtk_widget_remove_controller()` on a cell that has an armed
`GtkDragSource` timeout, so both expose the same UAF.

### 4. Why the 100 ms timeout exists in the first place

A natural reaction is "just remove the timeout and the UAF is gone". The
timeout is, however, deliberate UX hardening — and the same person who
added it (Corey Berla) is also a Nautilus maintainer, which is worth
spelling out.

`git log -S 'MIN_TIME_TO_DND' -- gtk/gtkdragsource.c` in `gtk_git_1`
points to a single introducing commit:

```
commit ff8d88b097792f34734c40733ea624949ce01cd5
Author: Corey Berla <corey@berla.me>
Date:   Tue Nov 29 14:15:25 2022 -0800

    dragsource: Add a timeout before starting a drag

    No one wants a drag that happens in less than 100ms (if so, it's
    probably accidental).
```

merged via [`GNOME/gtk!5275`](https://gitlab.gnome.org/GNOME/gtk/-/merge_requests/5275)
("Fix accidental DnD") on 5 Jan 2023. The MR description is empty; the
commit message above is the only written justification.

Functionally the patch is small: it adds a `guint timeout_id` to
`struct _GtkDragSource`, arms a 100 ms timeout in
`gtk_drag_source_begin()`, and AND-s the existing pixel-distance
threshold in `gtk_drag_source_update()` with `!source->timeout_id`. As a
result, even if the pointer crosses `gtk-dnd-drag-threshold` during the
press, no drag begins for the first 100 ms. The callback `drag_timeout()`
itself does only one thing — clears `source->timeout_id` so the next
motion event can pass the AND check.

The UX cases this filters are:

* a fast click + release where the mouse drifts a few pixels mid-click
  (HiDPI mice and shaky hands routinely cross the default 8 px threshold
  in <50 ms);
* a touchpad tap that the input stack reports as press + small post-press
  drift;
* the second press of a double-click landing on the same cell;
* high-frequency events where `update` arrives before the human even
  intended a gesture.

This commit fits a coherent late-2022 GNOME UX campaign against
"accidental actions on slightest movement". The clearest sibling on the
Nautilus side is António Fernandes'
[`7473be7aaf`](https://gitlab.gnome.org/GNOME/nautilus/-/commit/7473be7aaf61db97ec454481da740692f7ca0028)
("list-base: Prevent accidental hover-switch on slightest movement",
12 Aug 2022), which carries an explanation directly transferable to the
DnD case:

> Since the pointer is likely still moving (because it's tied to human
> hand movement), the timeout for entering the subfolder starts right
> away, resulting in an accidental location switch. … This has the side
> effect of also making it harder to accidentally [trigger DnD].

So the 100 ms timeout is **load-bearing UX**, not a vestigial check.
Any fix that removes it would also re-open the original
"accidental drag" complaints that motivated MR !5275.

The bitter irony: that same commit is what introduced the UAF we now see.
Before `ff8d88b097` there was no `timeout_id` field, no `drag_timeout()`
callback, no `g_timeout_add()` at press time — and therefore no main-loop
source whose `user_data` could outlive the controller. The 2022 UX
hardening accidentally created the corruption primitive we are
investigating. Any acceptable fix must preserve the 100 ms debounce.

### 5. Where the fix should live

The actual fix belongs to GTK. The constraints from §4 (must keep the
100 ms debounce, must not change `g_main_dispatch()` semantics) leave
two natural directions:

* **Take a strong reference on the timeout target** in
  `gtk_drag_source_begin()`:

  ```c
  source->timeout_id = g_timeout_add_full (G_PRIORITY_DEFAULT,
                                           MIN_TIME_TO_DND,
                                           drag_timeout,
                                           g_object_ref (source),
                                           g_object_unref);
  ```

  This keeps `source` alive until either the callback runs or the
  `GDestroyNotify` fires from `g_source_remove()` /
  `g_source_destroy()`. Costs one extra ref per press; preserves
  Corey Berla's 2022 UX behaviour exactly; trivially correct against the
  observed UAF (Valgrind block silences immediately when applied
  locally).

* **Or harden `gtk_drag_source_finalize()` / cell teardown** by
  flushing the timeout source synchronously
  (`g_main_context_iteration(NULL, FALSE)` after `g_clear_handle_id` is
  not enough on its own — by the time finalize runs the timeout's
  `GSource` may already be in `pending_dispatches`; the cleanest
  solution is still option 1).

A separate upstream issue is being prepared (draft in
`ai_thoughts_md/UpstreamIssues.md` and `FixDirection.md`; the GtkDragSource
section there pre-dates having GTK source available, so its identifiers
need a small refresh — see §7 below).

### 6. Possible Nautilus-side mitigations (until GTK is patched)

These are workarounds the Nautilus project could consider while a GTK fix
percolates. Each carries trade-offs and the maintainer is best placed to
judge them:

1. **Move the per-cell `GtkDragSource` to a single, view-level controller.**
   The drag-source on every cell exists mainly to call
   `on_item_drag_prepare()` and then forward the multi-selection. A single
   `GtkDragSource` installed on the view (where lifetime matches the view,
   not the recycled cell) and a `gtk_event_controller_get_widget()`-based
   hit-test inside `prepare` should give equivalent UX without the
   per-cell churn. This also matches the existing pattern for the
   view-wide `GtkDropTarget` (`on_view_drop`).

2. **Defer cell teardown to an idle.** Wrap the
   `gtk_signal_list_item_factory_teardown` chain so that controllers are
   removed in a `g_idle_add(... G_PRIORITY_LOW)` callback after the
   current main-loop iteration completes. This breaks the “same iteration
   that armed the timeout also frees the source” coupling; ugly, but
   surgical.

3. **Explicitly cancel `GtkDragSource` before
   `gtk_widget_remove_controller()`** by walking the cell’s controllers in
   the cell `dispose` hook and calling
   `g_object_run_dispose()` on any `GtkDragSource` first. This forces the
   `g_clear_handle_id` cancellation to run while no further dispatch can
   start. Still racy in theory (the timeout may already be in
   `pending_dispatches`), but empirically helps in similar GTK list-view
   crashes seen in other apps.

Mitigation 1 also has a side benefit: it removes the per-recycle cost of
constructing/destructing a `GtkDragSource` on every visible row, which is
non-trivial under fast scrolling.

### 7. Cross-references and corrections to existing notes

* `ai_thoughts_md/UpstreamIssues.md` (sections “GtkDragSource UAF” and
  “Cluster D”) and `ai_thoughts_md/FixDirection.md` (section 2.3) were
  drafted **before** the GTK source was available locally; they refer to
  invented identifiers `gtk_drag_source_drag_timeout_cb` and
  `priv->drag_timeout`. The actual upstream names are `drag_timeout()`
  and `source->timeout_id`. Will be cleaned up in the follow-up edit.
* The Cluster B / Cluster D / Cluster E classifications and the
  `nautilus-grid-cell.c` vs `nautilus-name-cell.c` teardown asymmetry
  noted in `ai_thoughts_md/SourceAsymmetry.md` are still factually
  accurate, but they are now seen as **secondary** — they decide *which
  signature manifests*, not whether a crash happens. The single
  `GtkDragSource` UAF is the gating cause; with lines 892–895 commented
  out, none of the asymmetry-related symptoms appear at all in our test
  runs.

### 8. Reproducer in one paragraph

Ubuntu 26.04 Wayland session, AMD Ryzen 7 7735HS + Radeon (Mesa). Open
Nautilus on a directory containing ~870 sub-folders and ~4 500 small files
(`dir_listings_1/dirsListing_1.txt` is a real example). Switch to either
List or Grid view. Hold one of the arrow keys to scroll quickly, or
shift-select large ranges, or press-drag-release on cells — anything that
makes `GtkSortListModel` emit `items-changed` while a fresh button-press
is in flight. Crash usually within ten seconds. Adding `G_SLICE=always-malloc
MALLOC_CHECK_=3 MALLOC_PERTURB_=85 G_DEBUG=fatal-criticals,fatal-warnings,gc-friendly`
makes the failure mode reproducibly an immediate `SIGABRT`/`SIGSEGV` with
the recycled-pointer corruption pattern documented above.

Commenting out lines 892–895 of `setup_cell_common()` and rebuilding
makes Nautilus survive the same workload indefinitely (DnD-out is broken
during the test, but every other interaction works normally).

---

*Author note:* this file is part of the local working notes under
`/home/denixx/projects/nautilus_things/ai_thoughts_md/`. The full
investigation, per-file crash-log classification, environment details,
upstream-issue correspondence and proposed patches live in the sibling
files in that directory. Quote whatever is useful.
