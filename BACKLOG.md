# Inventory Management — Problem Backlog

Issues surfaced during Playwright MCP smoke test on 2026-05-26.

---

## Open

### BUG-001 — `PurchaseOrderModal` component not resolved
- **Symptom:** `[Vue warn]: Failed to resolve component: PurchaseOrderModal` fires on every page load
- **Source:** Console error visible on Overview (and all routes — fires at app mount)
- **Impact:** Low — no visible UI breakage, but Vue is silently dropping the component
- **Next step:** Find where `PurchaseOrderModal` is referenced (likely in Overview or App.vue), confirm whether it was removed or never imported

### BUG-002 — `/tasks` endpoint returns 404
- **Symptom:** `Failed to load tasks: AxiosError` in console; `App.vue:36` making a request that gets a 404
- **Source:** `client/src/App.vue` line 36
- **Impact:** Low — no visible UI breakage, but a fetch is silently failing on every page
- **Next step:** Check `App.vue:36` — either add the endpoint to the backend or remove the dead call

### BUG-003 — Unit Cost shows `—` for all Restocking items
- **Symptom:** Restocking tab shows `—` in the Unit Cost and Subtotal columns for WDG-001, GSK-203, FLT-405
- **Source:** Restocking items come from the demand/backlog data; those SKUs aren't cross-referenced against `inventory` data for pricing
- **Impact:** Medium — budget math can't work without unit cost; Place Order is disabled until qty > 0 but subtotal stays `—`
- **Next step:** Join restocking items against inventory data by SKU to pull `unit_cost`

---

## Done

*(none yet)*
