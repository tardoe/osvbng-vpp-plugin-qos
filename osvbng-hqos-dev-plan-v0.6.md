# osvbng S-tag HQoS: Development Plan

**Version:** v0.6
**Date:** 7 August 2026
**Status:** Draft for review
**Supersedes:** v0.5
**Depends on:** `osvbng-hqos-analysis-v0.1.md`, `osvbng-hqos-dev-plan-v0.4-validation.md`

**Change log.** v0.4 → v0.5: Incorporates the four-track validation of v0.4 (source verification of both repos, VPP mechanism research, industry-norm research — see the validation report). Three structural changes. First, a new **Workstream A0** precedes everything: the shipped per-port aggregate contains a dequeue-gate CAS livelock, a gate-closed retry livelock, three buffer-accounting leak paths, and a rate representation that cannot express multi-gigabit rates — the foundation must be fixed and proven before it is extended, and §4 now catalogues each defect with a proposed fix. Second, **auto-attach is re-designed around an explicit verification gate (V1)**: strong source evidence says session interfaces are standalone midchain interfaces whose `sup_sw_if_index` walk terminates immediately, meaning the shipped port aggregate likely never attaches to real sessions and the v0.4 outer-VLAN read has nothing to read; §3.2 now carries both outcome branches, pending a lab check. Third, the design adopts industry-review corrections: configurable burst window (10–50 ms default at aggregates, replacing the fixed 150 ms), explicitly documented fairness losses (no priority propagation, no CIR floors), an Ethernet-presets-only restriction on tranche-one overhead accounting, and a stated contention ceiling (~40 Gbit/s per aggregate pending measurement). §2 corrects factual errors carried in v0.4 (tin scheduling is strict priority, not DRR; the 150 ms constant is a debt ceiling, not a burst window; shaper and buffer accounting use different length bases). Estimate revised from 3–5 weeks to **5–7 weeks**. v0.3 → v0.4 and earlier: see v0.4.

---

## 1. Scope

Extend osvbng QoS from two levels (subscriber CAKE, per-port aggregate) to three (subscriber → S-VLAN aggregate → port aggregate), plus control-plane plumbing — **on a repaired foundation**: the shipped port-aggregate tier is fixed, proven attached, and proven accurate first (A0). Out of scope: NIC offload, `vnet/tm`, upstream-direction hierarchy, weighted fairness across subscribers (documented as an accepted loss, revisit criteria in §3.5).

---

## 2. Verified current design (corrected from v0.4 §2)

Everything below is verified against `src/` at the current head of `osvbng-vpp-plugin-qos` (most recent commit: the cache-line split, `7c04b13`). Corrections to v0.4 are marked **[corrected]**.

**Subscriber level (`cake_sched_t`, `osvbng_qos_sched.h:175`).** Real queues: DiffServ tins → flows, COBALT AQM, virtual-time shaper (`global_shaper_time_ns` advanced by `adj_len * rate_ns_per_byte`). Owner-thread model with cross-thread handoff. Per-subscriber overhead compensation (`cake_overhead_adjust`) produces `adj_len`.

- **[corrected] Tin scheduling is strict priority, not DRR.** `cake_dequeue.c:255-265` services the highest-index tin with an active flow; `tin_deficit`/`tin_quantum` are written but never read (dead fields). DRR exists only across flows within a tin.
- **[corrected] The 150 ms constant is a debt ceiling, not a burst window.** It caps how far virtual time may run *ahead* of now after charging (`cake_dequeue.c:164-167`). There is no catch-up clamp on the subscriber shaper: an idle subscriber's virtual time lags `now` unboundedly, so accumulated credit is bounded only by dispatch batching, not by design.

**Aggregate level (`cake_aggregate_t`, `osvbng_qos_sched.h:155-173`).** One per physical/bond port, keyed by `sw_if_index` via `agg_index_by_sw_if_index`. No thread pinning. Two shared atomics (`global_shaper_time_ns` on cacheline1, `buffer_usage` on cacheline2), each on its own cache line; stats in per-thread slots (`cake_agg_stats_t *stats`). Preserve this layout exactly; new fields join the read-mostly cacheline0 group — anything placed after `buffer_usage` buys a fourth cache line.

- *Dequeue gate* (`cake_agg_dequeue_gate`, `osvbng_qos_sched.h:702-736`): CAS loop on `global_shaper_time_ns`. Gate closed → `cake_dequeue.c:158-162` un-pops (`flow->head--`) and retries on a later dispatch. **As shipped, this function livelocks — see D1.**
- *Buffer admission* (`cake_enqueue.c:257-276`): check-then-charge on `buffer_usage` against `buffer_limit`; exceed → drop + per-thread backpressure count. **Discharge coverage is incomplete — see D3.**
- **[corrected] Charging bases differ by mechanism.** The shaper gates charge overhead-adjusted `adj_len`; buffer admission charges raw `pkt_len` (`cake_enqueue.c:274`, `cake_dequeue.c:144,175`). §3.4 fixes the unit choice for the new tier explicitly.

**Attachment.** No bind API. `cake_sched_enable_disable` walks `sup_sw_if_index` (`osvbng_qos_sched.c:237-255`) toward the physical parent and caches `aggregate_index`. The walk only inspects ancestors, never the shaped interface itself. **Whether this walk ever succeeds for real sessions is the V1 gate (§3.2): sessions are midchain interfaces, and three independent lines of evidence say `sup_sw_if_index == self` for them.**

**Node model.** `cake-dequeue` is `VLIB_NODE_TYPE_INPUT`, registered DISABLED, enabled per worker while that worker owns schedulers — the idiomatic VPP pattern (upstream precedent: `nsim-wheel`). Gate-closed retry is free via polling; no wake machinery.

**Synchronization.** All lifecycle mutation runs under the worker barrier — but mostly *implicitly*: VPP API and CLI handlers hold the barrier by default, and no handler in this plugin is marked mp-safe. Aggregate create/delete additionally take the barrier explicitly (recursion-safe, harmless). **Spec invariant to state:** all lifecycle paths run under the API/CLI barrier; no handler may be marked mp-safe without re-review. (`hqos-qinq/DECISIONS.md`'s "already under barrier" is true only via this default.)

**Fairness stance.** The aggregate has no child lists. Under aggregate congestion, credit goes to whichever subscriber's dequeue wins the CAS; excess backs into per-subscriber CAKE where tins act *within* each subscriber. See §3.5 for what this does and does not provide.

**Process.** The repo runs a spec-driven workflow (`context/PROCESS.md`). Caveats found in validation: `context/SUMMARY.md` is internally contradictory about hqos-qinq status (says Phase 1, Phase 5-complete, and lists pre-pivot next steps in one file) and still reports API v1.0.0 (actual: **2.0.0**); `DECISIONS.md` makes two claims the code does not honour (discharge "on all buffer-free paths"; the burst-cap snippet that is the source of D1). `src/` is the only ground truth.

**Control-plane state (verified).** The aggregate API messages exist end-to-end — plugin `.api` v2.0.0 and generated GoVPP bindings (`pkg/vpp/binapi/osvbng_qos_sched/`, built against `VPP 26.06-release` matching `versions.env`) — and nothing in the Go control plane calls them. `pkg/southbound/vpp/qos.go` wires only the per-subscriber scheduler. There is no config schema, no telemetry, no conf handler for aggregates. This asymmetry is the C-workstream's starting point, unchanged from v0.4.

---

## 3. Target design: the S-VLAN level

Unchanged in essence from v0.4 — a second aggregate tier using the same lockless pattern, no pinning, no handoff changes, no child lists — with the corrections below.

```
subscriber cake_sched_t  ──gate──▶  S-VLAN cake_aggregate_t  ──gate──▶  port cake_aggregate_t
      (real queues)                  (atomic virtual-time)               (atomic virtual-time)
```

### 3.1 Data model

Extend `cake_aggregate_t` with `level` (`CAKE_AGG_LEVEL_PORT` | `CAKE_AGG_LEVEL_SVLAN`), `parent_index` (`~0` for port), and identity — all in cacheline0. Per-port `svlan → agg_index` map: 4096-entry `u32` vector hung off the port aggregate (~16 KB per port, O(1)). Replace `rate_ns_per_byte` with the fixed-point representation from D5 at both levels. Add per-aggregate `burst_ns` (D1/§3.6). Extend `cake_sched_t` with both cached indices: `agg_svlan_index`, `agg_port_index` (either may be `~0`) — hot path stays pointer-free.

Contention note (unchanged): an S-VLAN aggregate is touched by fewer workers and packets than its parent port aggregate; the lockless design holds a fortiori. Stated ceiling from validation: contended-cache-line throughput is ~10–20 M ops/sec regardless of core count, which makes the design comfortable to ~40 Gbit/s imix per aggregate and **unproven at 100 Gbit/s per aggregate** — Workstream D measures this, and if failed-CAS retry storms appear, the fallback is a fetch-and-add formulation of the gate (FAA never fails; small accuracy trade documented in the spec).

### 3.2 Attachment — GATED ON LAB VERIFICATION (V1)

**The v0.4 design assumed the `sup_sw_if_index` walk reaches the port from a session interface and that the session's sub-interface config carries the outer VLAN. Validation found strong evidence both assumptions fail:**

- Stock VPP session-style (midchain/tunnel) interfaces are registered standalone; `sup_sw_if_index == sw_if_index`. The plugin's own README classifies IPoE/PPPoE sessions as exactly this category.
- osvbng's control plane passes `EncapIfIndex`/`OuterVlan`/`InnerVlan` explicitly at session create, *fabricates* `SupSwIfIndex: encapIfIndex` in its own Go-side interface manager, and re-derives encap from config on restore — never trusting VPP for parentage.
- osvbng's topology is: physical → `parent.svlan` **single-tag** sub-interface (via `CreateSubif`) → session interface. The C-VLAN is never a VPP sub-interface. So the S-VLAN, when readable at all, is read from a one-tag sub-interface (`sub.eth.flags.one_tag`, `sub.eth.outer_vlan_id`), not a QinQ subif.

If confirmed, the *shipped port tier attaches to nothing* for real sessions (which would also explain how D1 shipped unnoticed — the gate never ran).

**V1 — lab check: RESOLVED (8 Aug 2026), branch (b), confirmed structurally AND functionally.** Structural: `show hardware-interfaces ipoe_session0` shows the session as its own hardware-registered interface (device class "IPoE") — a standalone `vnet_register_interface` interface, therefore `sup_sw_if_index == self`. Functional: with a 1 Gbit/s aggregate on `eth1` and a CLI-applied `diffserv4` scheduler on a live IPoE session under BNG Blaster bidirectional UDP, the scheduler processed 657 packets (enqueued == dequeued, owner thread claimed, correct tin classification) while every aggregate counter — buffer usage, shaped, backpressure — stayed at zero. Buffer charging occurs at enqueue whenever `aggregate_index != ~0`, so all-zero proves the walk left the scheduler unattached. The shipped port aggregate never attaches to sessions (consistent with D1 shipping unnoticed — the gate never runs for unattached schedulers).

**Branch (a) — walk works** (the osvbng dataplane fork sets meaningful parentage): keep v0.4's design as written. During the existing walk, when the visited interface is `VNET_SW_INTERFACE_TYPE_SUB` with `sub.eth.flags.one_tag`, read `sub.eth.outer_vlan_id` as the S-VLAN and look it up in the port's map. Hit → cache `agg_svlan_index`; miss → port-only, exactly today.

**Branch (b) repair: (b1) selected (8 Aug 2026)** — fix parentage at the source in the sessions plugins, now that their source is available (`osvbng-vpp-plugin-ipoe/`, `osvbng-vpp-plugin-pppoe-control/` in the workspace). Both plugins create sessions via `vnet_register_interface` with a midchain adjacency stacked to `encap_if_index`, and neither ever touches `sup_sw_if_index`. The patch, per plugin:

- **IPoE** (`osvbng_ipoe.c`, after `s->sw_if_index = sw_if_index = hi->sw_if_index;`):
  ```c
  vnet_sw_interface_t *si = vnet_get_sw_interface (vnm, sw_if_index);
  si->sup_sw_if_index = a->encap_if_index; /* parent S-VLAN sub-interface */
  ```
- **PPPoE** (`osvbng_pppoe.c`): same assignment at the common point after the register/reuse branches (an `si` fetch already exists there) — it must run on **both** branches because PPPoE recycles hidden interfaces from `free_pppoe_session_hw_if_indices` and each reuse may carry a different encap. On the hide/freelist path, reset `si->sup_sw_if_index = sw_if_index;` so a parked interface never dangles at a possibly-deleted subif.

With parentage fixed, the walk traverses session → S-VLAN subif → physical, branch (a)'s outer-VLAN read applies verbatim (`sub.eth.flags.one_tag` → `sub.eth.outer_vlan_id` on the encap subif), and **the shipped port tier is fixed for free**.

**Safety audit: COMPLETE (8 Aug 2026, against the v26.06 tag — exhaustive over all 33 `sup_sw_if_index` references in `src/`).** Verdict: **(b1) is safe in VPP 26.06 core.** Findings, all verified by reading each site:

| Site | Verdict for a HARDWARE-type interface with `sup != self` |
|---|---|
| `vnet_get_sup_sw_interface` / `vnet_get_sup_hw_interface` (`interface_funcs.h:48-65`) | Type-gated: follow `sup` only for SUB/P2P/PIPE — core helper paths **ignore** the field entirely |
| Admin-up super-interface check (`interface.c:354-356`) | Gated on `type == VNET_SW_INTERFACE_TYPE_SUB` — sessions skip it; no bring-up ordering constraint |
| `vnet_sw_interface_is_sub` (`interface_funcs.h:322`, sup-based) | **Was mis-assessed here — see the amendment below.** `vnet_sw_interface_supports_addressing` (`interface.c:1345`) is additionally type-gated and safe, but the linux-cp caller is *not* limited to already-paired interfaces and required an ordering guarantee to be safe |
| Drop/punt counter path (`interface_output.c:1048`) | Sup-based, benign and arguably desirable: the encap subif's drop counters will include session drops, matching sub-interface semantics |
| `sw_interface_details` encoder (`interface_api.c:270-280`) | `type` field stays HARDWARE (type-gated switch); the sup-based else-branch emits the `sub` union — for sessions that means `sub_id = hw_if_index` (union aliasing) with zeroed tag/VLAN fields. Cosmetic only: osvbng ignores `sub_id` and reads `SubNumberOfTags`, which stays 0 — identical to today |
| hw-delete sub-interface cleanup (`interface.c:1075-1085`) | Iterates only the parent hw's `sub_interface_sw_if_index_by_id` hash — sessions are never registered there; deleting the physical port does not cascade into sessions (unchanged) |
| upstream `pppoe` plugin, `vnet/dev`, `sflow`, remaining hits | Not loaded / not applicable to sessions / debug-print / creation-time writes |

Supporting context: the IPoE plugin already reads the encap's `sub.eth` (TPID snapshot) as in-plugin precedent, and the osvbng control plane *already fabricates* `SupSwIfIndex = encapIfIndex` in its interface manager — (b1) makes `sw_interface_dump` agree with the control plane's own model instead of contradicting it on every `LoadInterfaces`. Standing constraints: reset `sup = self` when PPPoE parks an interface on the free-list (in the patch); never delete an S-VLAN subif while sessions reference it (already guaranteed by config structure); re-run this one-grep audit on any future `DATAPLANE_VERSION` bump.

**Audit amendment (9 Aug 2026) — the plugin layer, and the ordering constraint it imposes.** The audit above covered VPP *core* and dismissed linux-cp too quickly. `lcp_itf_interface_add_del` (`lcp_interface_sync.c:424`, registered `VNET_SW_INTERFACE_ADD_DEL_FUNCTION`) does **not** act only on already-paired interfaces: it acts on interfaces whose *parent* is paired, gating on `vnet_sw_interface_is_sub()` — defined purely as `sw->sw_if_index != sw->sup_sw_if_index` (`interface_funcs.h:322`), with no type check. osvbng loads `linux_cp_plugin.so` and enables `lcp-auto-subint` (`templates/dataplane.conf.tmpl:71`). Left unguarded, (b1) would make every session look like a sub-interface to linux-cp, which auto-creates a host TAP named `<parent_host_if>.<sub.id>` per session — `sub.id` is 0 on a hardware-type interface, so all sessions collide on one name. At subscriber scale that is a netlink/tap storm.

**What makes (b1) safe is placement, and it must be preserved.** `vnet_register_interface` builds the sw interface via `vnet_create_sw_interface_no_callbacks` — which sets `sup = self` for HARDWARE type (`interface.c:543-544`) — and fires the create-time add/del callbacks only at the end via `set_flags_helper(..., IS_CREATE)` (`interface.c:228`). Assigning `sup` **after** `vnet_register_interface()` returns means those callbacks still observe `sup == self`, so linux-cp early-returns before reaching `lcp_itf_pair_find_by_phy()`. Any future refactor moving the assignment earlier — or into a template passed to `vnet_register_interface` — reintroduces the storm. The shipped patches carry this reasoning in a comment at both sites; treat it as a review checklist item.

Residual, accepted: on IPoE teardown the delete callbacks *do* observe the parented value, so linux-cp's sync handler calls `lcp_itf_pair_delete()` on an unpaired interface — an `INDEX_INVALID` lookup returning immediately (`lcp_interface.c:522-535`), one futile hash lookup. PPPoE has no exposure (hide-and-recycle, never `vnet_delete_sw_interface`, so no callback). Lab state 9 Aug 2026: `eth1` is LCP-paired, encap `eth1.100` is **not** (`show lcp` lists only `loop100`, `eth0`, `eth1`, `eth2`), so even the guarded path finds nothing.

Correction to the table row above on the dump encoder: `sub` is a distinct struct member, not a union alias of `hw_if_index`, so sessions emit `sub_id = 0` rather than `sub_id = hw_if_index`. All emitted `sub` fields are zero, which is what the previous (neither-branch) path produced — wire result unchanged either way.

**Status: implemented (9 Aug 2026).** Branch `fix/session-sup-sw-if-index` in both `osvbng-vpp-plugin-ipoe` (16 lines) and `osvbng-vpp-plugin-pppoe-control` (23 lines), pushed to the `tardoe` forks; PR drafts in `PR-DRAFTS-session-sup-sw-if-index.md`. **Not compiled and not lab-verified** — both pending a Linux dataplane rebuild. Note the sequencing hazard: merging these makes attachment work, which exposes D1 (the gate CAS livelock) on the first shaped packet, so D1 must land before or with them.

**(b2) fallback, retained** if the audit finds a blocker: new `osvbng_cake_sched_enable_disable_v2` carrying `encap_sw_if_index` (CRC-stable for old clients); `ApplyScheduler` already holds the value.

Either way the v0.4 property that matters is preserved: **the control plane manages aggregates only; attachment stays automatic.** Store both resolved indices in the scheduler; disable clears both.

### 3.3 The two-level gate and the refund problem

Unchanged mechanism, now specified against the *fixed* gate (D1):

1. Check the **S-VLAN gate first**. Closed → stop; nothing charged.
2. Check the **port gate**. Closed → **refund** the S-VLAN charge (`__atomic_fetch_sub` of the same `cost_ns`), count `parent_blocked`, un-pop.

Order rationale unchanged: S-tag aggregates are normally provisioned under port rate, so child-first makes refunds rare. Refund safety unchanged: a concurrent worker that observed the inflated time waits marginally longer for one packet — bounded, self-correcting, accuracy-not-correctness. Validation found no precedent for a two-level atomic shaper with refund in VPP/DPDK — the pattern is sound by construction (every charge pairs with exactly one refund on every failure path) but genuinely novel; it is validated empirically in Workstream D, per the repo's "do not spec what you cannot verify" rule.

**Charging basis (decided):** both gates charge the subscriber-adjusted `adj_len`; both levels' *buffer admission* charges raw `pkt_len`. This matches shipped behaviour exactly and keeps each mechanism internally consistent; the spec states it. Tranche-two flag unchanged (per-aggregate overhead delta, one `i16` at gate time) with a new restriction from industry review: **tranche one's subscriber-basis charging is valid for Ethernet-family overhead presets only; ATM DSL presets under an aggregate misstate utilization 10–20% on small-packet mixes and are documented as unsupported until tranche two.**

### 3.4 Enqueue admission

As v0.4 — check/charge S-VLAN then port, unwind S-VLAN on port rejection, discharge both levels through one helper on all free paths — but built on the D3 unified-free fix and the D4 admission pattern (fetch-add then undo, bounding over-admission to one packet per worker instead of unbounded check-then-charge racing). Per-level backpressure counters are the operator congestion signal.

### 3.5 Fairness stance — documented, not implicit

Adopt the shipped trade-off, with the two losses named (industry review: Cisco/Juniper/Nokia all provide these; Nokia's Vport `agg-rate` is the deployed precedent for shipping without them):

- **No priority propagation.** Under a closed S-VLAN gate, the next credit goes to whichever worker polls first, tin-blind — one subscriber's bulk can delay another's voice. Bounded: the gate reopens continuously in virtual time (per-packet granularity, no timeslices), and each subscriber's own tins protect their mix within whatever share they win.
- **No CIR floors or weighted sharing.** Share under aggregate congestion tracks dequeue-attempt rate ≈ offered load: a 32-flow bulk subscriber out-pulls a single-call VoIP subscriber. This is the *expected* behaviour, not a pathology; the revisit trigger is lab evidence that capture ratios exceed what per-subscriber CAKE tolerably contains (D measures capture ratio under S-VLAN congestion with asymmetric offered load).

The pre-pivot DRR design, if ever needed, lives in the hqos-qinq spec text and its Codex/Gemini reviews (`DECISIONS.md` itself holds only the one-line "Weighted DRR in Phase 1" rejection).

### 3.6 Burst window

Replace the fixed 150 ms with per-aggregate `burst_ns`, default **25 ms** (configurable 10–150 ms). Industry envelope is 10–125 ms and the low end at aggregates; 150 ms of credit at a 1 Gbit/s S-VLAN is ~18 MB released at port speed into exactly the downstream device the shaper protects. The subscriber-level 150 ms debt ceiling is out of scope here (pre-existing behaviour, noted for a future issue).

---

## 4. Defects in the shipped implementation, with proposed fixes (Workstream A0)

Each defect below was verified against source during validation (file:line cited). Fixes are proposed here for the spec phase; final form goes through the repo's review process. **D1–D5, D7 and D8 are dataplane prerequisites for the S-VLAN tier; D6 is control-plane and lands with C3.**

### D1 — Dequeue-gate CAS livelock (worker hang) — `osvbng_qos_sched.h:714-729`

The burst clamp mutates the CAS *expected* value locally while memory keeps the stale time. First packet with memory time `T < now_ns` (i.e. the first packet ever, since the field is written only at creation): local copy clamps to `now_ns`, CAS expects `now_ns`, memory holds `T`, CAS fails, loop reloads `T`, clamps again — **infinite spin; the worker hard-hangs.** No thread ever stores a value ≥ `now_ns`, so nothing unblocks it.

Fix — keep the loaded value as the CAS expected value; clamp into a separate base; fold in the configurable burst window (§3.6):

```c
static_always_inline u8
cake_agg_dequeue_gate (cake_main_t *cm, cake_aggregate_t *agg, u32 adj_len,
                       u64 now_ns, u32 thread_index)
{
  u64 cost_ns = ((u64) adj_len * agg->rate_scaled) >> CAKE_RATE_FRAC_BITS;
  u64 old_time, base, new_time;

  do
    {
      old_time = __atomic_load_n (&agg->global_shaper_time_ns, __ATOMIC_ACQUIRE);
      if (old_time > now_ns)
        return 0;                        /* shaper ahead of wall clock: closed */
      /* reclaim idle credit, bounded by the configured burst window */
      base = old_time;
      if (now_ns - base > agg->burst_ns)
        base = now_ns - agg->burst_ns;
      new_time = base + cost_ns;
    }
  while (!__atomic_compare_exchange_n (&agg->global_shaper_time_ns, &old_time,
                                       new_time, 1 /* weak */,
                                       __ATOMIC_ACQ_REL, __ATOMIC_ACQUIRE));
  /* per-thread stats as today */
  return 1;
}
```

The CAS expected value is always exactly what was loaded, so the loop terminates. Semantics change deliberately: instead of zero credit after idle (the intent of the old clamp), up to `burst_ns` of credit accumulates — the correct shaper behaviour and the industry norm.

### D2 — Gate-closed retry livelock in the dequeue loop — `cake_dequeue.c:211-212, 302-362`

`now_ns` is computed once per node dispatch, and neither `budget` nor `flow->deficit` decrements when the gate returns 0 — the bulk branch (`while (flow->deficit > 0 && budget > 0)`) re-attempts the same flow against the same closed gate with the same `now_ns` forever within one dispatch. Masked today only by D1 hanging first; in the two-level design gate-closed is the *common* case.

Fix — make gate-closed a distinct outcome that terminates work on this scheduler for this dispatch: change `cake_dequeue_one` to return an enum (`CAKE_DEQ_SENT` / `CAKE_DEQ_EMPTY` / `CAKE_DEQ_SHAPER_WAIT` / `CAKE_DEQ_GATE_CLOSED`); on `GATE_CLOSED`, break out of both the sparse and bulk per-flow loops and advance to the next scheduler in the per-thread bitmap. The polling node retries the gated scheduler on the next dispatch with a fresh `now_ns`. Exit criterion: a closed gate can never consume more than one gate check per scheduler per dispatch.

### D3 — Buffer-accounting leaks on three free paths — `cake_enqueue.c:54-58, 294-303`; `osvbng_qos_sched.c:297-299`

`cake_agg_discharge` is called at exactly two sites (AQM drop `cake_dequeue.c:144`, transmit `:175`). Three paths free buffers that were charged at enqueue without discharging: **flow ring-full drop**, **set-associative flow eviction** (which also leaks `cs->buffer_usage` and tin counters), and **subscriber teardown drain**. `buffer_usage` is a monotonically-leaking `u32` → the aggregate wedges into permanent backpressure on a long-running box. (`DECISIONS.md`'s "called on all buffer-free paths" is aspirational, not actual.)

Fix — one helper owns every charged-buffer free:

```c
/* Frees a buffer that passed enqueue admission. The ONLY legal way to free
 * such a buffer. Discharges aggregate (both levels once S-VLAN exists) and
 * subscriber accounting, then frees. */
static_always_inline void
cake_charged_buffer_free (vlib_main_t *vm, cake_main_t *cm, cake_sched_t *cs,
                          u32 buffer_index, u32 pkt_len)
{
  cake_agg_discharge (cm, cs, pkt_len);   /* generalises to walk both indices */
  cs->buffer_usage -= pkt_len;
  cs->queued_buffers--;
  vlib_buffer_free_one (vm, buffer_index);
}
```

Convert all five sites (transmit and AQM drop keep their existing counter updates around the call; ring-full drop, eviction via `cake_flow_ring_free` gaining a `cs` parameter and per-entry length reads, and teardown drain via `cake_tin_drain` adopt it). Exit criterion: after any traffic pattern including teardown under load, quiescent `buffer_usage == 0` at every level. Also fix in passing: the `AGG_SHAPED`/`AGG_BACKPRESSURE` error counters in `osvbng_qos_sched_error.def` are defined but never incremented — wire or remove them.

### D4 — Admission check-then-charge race — `cake_enqueue.c:262-275`

The load/compare/add sequence lets N workers over-admit simultaneously by an unbounded amount. Fix with charge-then-verify (same cost, overshoot bounded to one packet per worker):

```c
u32 prev = __atomic_fetch_add (&agg->buffer_usage, pkt_len, __ATOMIC_RELAXED);
if (PREDICT_FALSE (prev + pkt_len > agg->buffer_limit))
  {
    __atomic_fetch_sub (&agg->buffer_usage, pkt_len, __ATOMIC_RELAXED);
    vec_elt_at_index (agg->stats, thread_index)->backpressure_events++;
    /* drop as today */
  }
```

### D5 — Rate representation cannot express aggregate rates — `osvbng_qos_sched.c:178-179, 386-387`

`rate_ns_per_byte = (u64) 1e9 / rate_bytes_per_sec` truncates to integer nanoseconds per byte: above 8 Gbit/s the result is **0 (unshaped)**; 5 Gbit/s shapes at 8 Gbit/s (+60%); 3 Gbit/s at 4 Gbit/s (+33%); ±1% needs rates ≤ ~80 Mbit/s. Port and S-VLAN aggregates live exactly in the broken band.

Fix — 16 fractional bits, u64 arithmetic throughout (no 128-bit needed):

```c
#define CAKE_RATE_FRAC_BITS 16
/* rate_scaled = ns-per-byte in Q48.16; exact to ~0.002% at 100 Gbit/s */
agg->rate_scaled = rate_bytes_per_sec > 0
  ? (1000000000ULL << CAKE_RATE_FRAC_BITS) / rate_bytes_per_sec
  : 0;
/* hot path: */
u64 cost_ns = ((u64) adj_len * agg->rate_scaled) >> CAKE_RATE_FRAC_BITS;
```

Range check: numerator `1e9 << 16` ≈ 6.6e13 fits u64; worst-case product (jumbo `adj_len` ≈ 2^14 × `rate_scaled` at 8 kbit/s ≈ 2^36) ≈ 2^50 — no overflow. Apply to **both** subscriber and aggregate shapers (same formula, same field rename), since subscriber rates ≥ ~500 Mbit/s already carry multi-percent error today. `rate_bytes_per_sec` stays as the API/CLI-visible value.

### D6 — `ApplyScheduler` idempotency is broken on osvbngd restart — `osvbng/pkg/southbound/vpp/qos.go:220-225`

The guard map (`schedulerIfs`) is in-process and never rebuilt from the dataplane. On osvbngd restart with VPP surviving, replayed `ApplyScheduler` gets `VNET_API_ERROR_ENTRY_ALREADY_EXISTS`, treated as a hard error → map never repopulates → later `RemoveScheduler` no-ops → the CAKE instance leaks. The `apply.go:34-38` idempotency comment is wrong for this path.

Fix (lands with C3, same pattern used for aggregates from day one): on already-exists, **re-assert rather than adopt** — D8's PPPoE freelist reuse means an existing dataplane scheduler may belong to a *previous* subscriber at a different rate, so blind adoption is wrong:

```go
if err := v.checkRetval(reply.Retval); err != nil {
    if !errors.Is(err, api.ENTRY_ALREADY_EXISTS) {
        return fmt.Errorf("cake sched enable: %w", err)
    }
    // Dataplane already has a scheduler on this sw_if_index: either our own
    // (osvbngd restart with VPP preserved) or a stale one leaked by abrupt
    // teardown onto a recycled interface (D8). Parameters may differ —
    // remove and recreate so the dataplane matches THIS session's policy.
    if err := v.removeSchedulerRaw(swIfIndex); err != nil {
        return fmt.Errorf("cake sched replace: %w", err)
    }
    if err := v.applySchedulerRaw(swIfIndex, params); err != nil {
        return fmt.Errorf("cake sched replace: %w", err)
    }
}
v.mu.Lock()
v.schedulerIfs[swIfIndex] = struct{}{}
v.mu.Unlock()
```

(Exact error-matching helper per GoVPP's `api.RetvalToVPPApiError`; `*Raw` = the plain enable/disable calls without the map guard. Once the A2 update message exists, remove+recreate collapses to a single update call. Mirror the whole pattern in `ApplyAggregate`.)

### D7 (minor, fold into A0) — walk never matches the shaped interface itself

`osvbng_qos_sched.c:246-250` inspects only ancestors. Harmless today; add the self-check (or a comment stating the intent) so an aggregate configured directly on a shaped interface isn't silently ignored.

### D8 — no interface-deletion hook: stale schedulers on deleted/reused sw_if_index — lab-discovered 8 Aug 2026

The plugin registers no `VNET_SW_INTERFACE_ADD_DEL_FUNCTION` (verified: zero hits in `src/`). A scheduler whose interface is deleted out from under it (abrupt session teardown, VPP-side delete, or any path skipping the control plane's `RemoveScheduler`) becomes a stale pool entry that:

- silently stops shaping (feature-arc config dies with the interface), while still appearing healthy in `show cake scheduler` under whatever interface now owns the index;
- squats on the reused `sw_if_index`: VPP recycles interface indices, so the next session assigned that index gets `ENTRY_ALREADY_EXISTS` from `cake_sched_enable_disable` (`osvbng_qos_sched.c:165-167`) — and because the control plane treats QoS failures as non-fatal (`svcgroup/apply.go`), that subscriber silently runs **unshaped**.

Observed live: after a BNG Blaster restart recreated sessions, the previously-applied scheduler showed under `ipoe_session1` (the new holder of the recycled index) with `ip4-output: none configured` and zero counters.

**Session-plugin source review (8 Aug 2026) splits D8 into two cases:**

- **IPoE deletes** its session interfaces (`vnet_delete_hw_interface`, `osvbng_ipoe.c:250`) — the add/del callback below covers it, and the observed symptom is the deleted-interface one (feature config gone, scheduler inert).
- **PPPoE never deletes**: teardown hides the interface (`VNET_SW_INTERFACE_FLAG_HIDDEN`) and parks it on `free_pppoe_session_hw_if_indices` for reuse (`osvbng_pppoe.c:493-495`). The add/del callback never fires, **and the feature-arc config survives** — a stale scheduler on a recycled PPPoE interface keeps *actively shaping the next subscriber at the previous subscriber's rate*. Worse than inert. Mitigations, layered: the control-plane teardown path (`ReverseFromSession` → `RemoveScheduler`) remains the primary removal; the D6 fix must **re-assert parameters on `ENTRY_ALREADY_EXISTS`** (remove+recreate, or the A2 update message) rather than silently adopt, which converts any leaked stale scheduler into a correct one at next apply; and the qos plugin's enable path already replaces nothing silently, so no further dataplane change is required once D6 re-asserts.

Fix — register the callback and treat interface deletion as teardown:

```c
static clib_error_t *
cake_sw_interface_add_del (vnet_main_t *vnm, u32 sw_if_index, u32 is_add)
{
  cake_main_t *cm = &cake_main;
  if (is_add)
    return 0;
  if (sw_if_index < vec_len (cm->sched_index_by_sw_if_index) &&
      cm->sched_index_by_sw_if_index[sw_if_index] != ~0)
    cake_sched_enable_disable (vlib_get_main (), sw_if_index,
                               0 /* disable */, 0, 0, 0, 0, 0, 0, 0, 0, 0);
  /* aggregates keyed on a deleted port: same treatment via
     agg_index_by_sw_if_index (delete refuses while members exist —
     detach members first, per A6 semantics) */
  return 0;
}

VNET_SW_INTERFACE_ADD_DEL_FUNCTION (cake_sw_interface_add_del);
```

The disable path already drains tins and frees buffers; with D3 in place the drain also discharges aggregate accounting, so deletion under queued traffic stays leak-free. The callback runs in main-thread context during interface deletion (barrier semantics per the §2 invariant).

---

## 5. Workstreams

### Workstream A0 — fix and prove the port tier (plugin repo) — NEW

Its own issue and PR series upstream, independently valuable regardless of the S-VLAN tier. Spec through `context/PROCESS.md` as a defect-fix spec citing D1–D5/D7/D8 — D8 and the attach failure are now lab-verified facts, not inferences, which materially strengthens the issue.

Files: `src/osvbng_qos_sched.h`, `src/osvbng_qos_sched.c`, `src/cake_dequeue.c`, `src/cake_enqueue.c`, `src/osvbng_qos_sched_error.def`.

- **A0.1** Gate fix (D1) + configurable `burst_ns` (§3.6) + fixed-point rate (D5). 1–2 days.
- **A0.2** Dequeue gate-closed outcome (D2). 1 day.
- **A0.3** Unified charged-buffer free (D3) + admission pattern (D4) + interface-deletion hook (D8) + dead counters. 1.5–2.5 days.
- **A0.4** Attach repair: **(b1) selected, safety audit already complete** (§3.2) — `sup_sw_if_index = encap_if_index` in `osvbng-vpp-plugin-ipoe/osvbng_ipoe.c` (create) and `osvbng-vpp-plugin-pppoe-control/osvbng_pppoe.c` (create, both register/reuse branches; reset on hide/freelist). Remaining work is the two small patches + lab re-verification of attachment (the V1 functional test, expected to now show aggregate counters moving). Note this workstream spans three plugin repos. 0.5–1 day.
- **A0.5** Prove it: containerlab test 18/19 green (zero behaviour change for subscriber-only configs); a port aggregate on a live session demonstrably attached (`show cake scheduler` shows the index) and shaping within tolerance at a multi-gigabit rate (validates D5); quiescent `buffer_usage == 0` after teardown-under-load (validates D3). 1 day.

Exit: the port tier works, attached, accurate, leak-free — the "extend a proven pattern" premise of everything below is actually true. **3–5 days.**

### Workstream A — dataplane: the S-VLAN tier (plugin repo)

As v0.4 A1–A6 with these deltas:

- **A1. Data model and lifecycle (2–3 days).** As v0.4, plus: `burst_ns` and `rate_scaled` fields (from A0), both cached indices in `cake_sched_t` (`agg_svlan_index`, `agg_port_index` replacing single-index usage). Cache-line rule restated: new fields in cacheline0 only. Delete refuses while any scheduler references the aggregate (the existing delete-time scheduler sweep is the mechanism).
- **A2. Binary API (1–2 days).** As v0.4 — new messages `osvbng_cake_svlan_aggregate_create`/`_delete` (port `sw_if_index`, rate, buffer limit, `burst_ns`, svlan list/ranges), extended or v2 details with level/parent/svlan-set/`parent_blocked`. Confirmed correct strategy: CRCs are per-message; old control planes never look up new messages; osvbng should call GoVPP `CheckCompatibility` on the message set it uses. **Addition:** `osvbng_cake_aggregate_update` (rate, burst, buffer limit) for both levels — no modify message exists today, and `buffer_limit` is rate-derived at create so an update must recompute it (open question 4 is now closed: an atomic store is sufficient *for the aggregate* only if the update path also refreshes `rate_bytes_per_sec` and derived `buffer_limit`; the subscriber path is out of scope — its `mtu_time_us` feeds COBALT and needs more care). API freeze at A2 exit unblocks B and C1–C3.
- **A3. Auto-attach (1–2 days).** Per §3.2, branch decided by V1. Both branches read the S-VLAN from the encap sub-interface's `sub.eth.outer_vlan_id` under `flags.one_tag` (osvbng's S-VLAN subifs are single-tagged — not the QinQ read v0.4 described). Disable clears both indices.
- **A4. Two-level gate, refund, admission (3–5 days).** As v0.4, on the A0-fixed primitives: child-first gate chain with refund + `parent_blocked` (per §3.3), `cake_charged_buffer_free` walking both indices, two-level admission per §3.4. Zero behavioural delta for single-level configs.
- **A5. CLI and stats (1–2 days).** As v0.4: `show cake aggregate [interface]` renders port → S-VLAN tree with rate, burst, buffer usage/limit, shaped, backpressure, `parent_blocked`; `cake_agg_stats_sum` extended.
- **A6. Backfill on late create (1–2 days).** Elevated from "decide the semantic" to required: `cake_aggregate_create` (both levels) walks the scheduler pool under the barrier and re-resolves attachment for schedulers on that port/svlan — confirmed absent today, and confirmed necessary because delete+create currently orphans every live session's binding silently (delete sweeps, create does not). Delete semantics as v0.4: S-VLAN delete detaches members to the port level; port delete with children refused.

Dataplane total (A0 + A): **2–3.5 weeks** including tests.

### Workstream B — bindings

Regenerate `pkg/vpp/binapi/osvbng_qos_sched/` from the new API JSON. Half a day, gated on A2.

### Workstream C — control plane (osvbng)

1. **C1 config schema.** `qos-aggregates` map: name → interface, rate, optional `svlans`, optional burst/buffer limit. **Correction:** `ParseVLANRange` accepts a single value or one hyphen range only — model `svlans` as a YAML list of range strings (`svlans: ["100", "200-299"]`), parsed per-entry; no comma syntax.
2. **C2 config handler.** **This is the first QoS construct programmed at commit time — `qos-policies` has no conf handler today, so C2 is a new `conf.Handler`, not an extension.** Validation as v0.4 (interface exists — template: `validate_mss_clamp.go`; svlan sets disjoint per port; child rate ≤ parent rate; oversubscription across children allowed). Ordering via `Dependencies()`: the aggregate handler depends on the interface path; commit-time topological sort (`sortChangesByDependencies`) is confirmed to exist and enforce this.
3. **C3 southbound.** `ApplyAggregate`/`RemoveAggregate`/`UpdateAggregate` beside `ApplyScheduler` — idempotent *properly*: already-exists adopted as success with state recorded (D6 pattern), applied to the scheduler path in the same series. No per-session bind call, unchanged.
4. **C4 ordering.** Confirmed supported: boot applies config (`bootstrapDataplane` → `ApplyLoadedConfig`) before components start and sessions restore, so aggregates precede subscriber activation on fresh boot; A6 backfill covers runtime creation against live sessions. Nothing new to build beyond C2's declared dependencies.
5. **C5 telemetry.** Confirmed nearly free: a `show/qos/aggregate` handler calling `DumpAggregates` + `telemetry.RegisterMetric[southbound.AggregateState]` in its `init()` gets CLI, the existing 10 s poller, and Prometheus export in one file (`pkg/handlers/show/qos/scheduler.go` is the exact template). Export per-level shaped bytes, buffer usage, backpressure, `parent_blocked`.

Control-plane total: **1.5–2 weeks**, parallel with A4 onward.

### Workstream D — testing and calibration

v0.4 structure plus one new line item and one correction:

- **D0 (new): multi-thread shaper test harness (3–5 days).** No harness exists anywhere in the repo capable of hosting the promised refund-accuracy test — `tests/` is a containerlab benchmark driver only. Stand up a minimal standalone test binary that links the header inlines (`cake_agg_dequeue_gate`, admission, discharge) against stub `vlib` time, spawns N pthreads hammering two-level gate/refund/discharge, and asserts long-run rate accuracy ±1% per level and exact buffer-accounting balance. This is the only place D1/D3/D4 and the refund path are testable deterministically; containerlab cannot do it (TESTING.md: af-packet/veth has no bottleneck, cannot validate latency-under-load — unchanged constraint, no plan claims otherwise).
- Rate-accuracy per level under mixed load (S-VLAN constrained / port constrained / both) — now meaningful post-D5, including a ≥10 Gbit/s port-rate case.
- **Capture-ratio measurement (new, §3.5):** S-VLAN congested, asymmetric offered load (32-flow bulk vs 1-flow VoIP subscriber) — record the share ratio and per-subscriber tin latency; this is the documented revisit trigger for weighted sharing.
- Contention ceiling measurement (§3.1): per-aggregate Mpps at which failed-CAS rate inflects, on real multi-worker hardware.
- Overhead calibration against QinQ/PPPoE encapsulation and per-worker capacity benchmark — unchanged from v0.1.

Testing total: **2 weeks**, overlapping.

---

## 6. Contribution path

As v0.4, with one honesty fix and one addition:

1. **A0 goes first as its own issue** ("port aggregate: gate livelock, accounting leaks, rate truncation, attach verification") with the defect catalogue from §4 — it is a contribution on its own merits and builds the relationship before the feature ask.
2. Open or extend issue #1 for the S-VLAN tier, **explicitly acknowledging this reverses the documented per-port pivot** recorded in `hqos-qinq/DECISIONS.md` ("Aggregate scope: per physical/bond port (was per S-VLAN)") — with the argument that the lockless per-S-VLAN tier avoids the thread-pinning problems that motivated the pivot, which their own decision log anticipated as future work.
3. Author `context/specs/hqos-svlan/` per `context/PROCESS.md`, importing the accepted hqos-qinq findings (worker-barrier lifecycle — stated as the §2 invariant; unified drop helper — now actually implemented via D3; burst cap — in its D1-corrected form) and inheriting the lockless decisions rather than re-litigating DRR.
4. One review-friction item to pre-empt: `PROCESS.md` First-Class Requirement #2 mandates dual/quad-loop hot paths with `MULTIARCH_SOURCES` SIMD variants; the *existing* code is scalar single-loop throughout (and `cake_hash.c` is an empty stub). The spec should either scope batching as explicitly out (matching the shipped baseline, zero-delta rule) or negotiate the exemption up front — not discover it in review.
5. Run their Phase 2/3 review pattern, then implement.

---

## 7. Open questions

Closed since v0.4: gate placement, buffer accounting ownership, member iteration, thread model, HA (all v0.4 §6); backfill semantics (A6 — now decided: backfill on create, detach-to-parent on S-VLAN delete, refuse port delete with children); runtime rate update (A2 — new update message; aggregate-side safe with derived-state refresh; subscriber-side out of scope); polling cost (accepted practice with upstream precedent — nsim; measure in D but not gating).

Remaining:

1. **V1: fully closed — branch (b), repair (b1), safety audit complete** (see §3.2; lab-confirmed, source-confirmed, and audited against the v26.06 tag, 8 Aug 2026). All 33 `sup_sw_if_index` references in VPP core were read and dispositioned; no blocker. (b2) remains documented as fallback only in case the fork's tree diverges from upstream v26.06 in these files (verify with the same grep against the fork if it carries local core patches).
2. **Refund-path accuracy under contention** — D0 harness + D rate tests, empirical.
3. **Contention ceiling** — at what per-aggregate packet rate does CAS retry inflect; decides whether the FAA fallback is needed before 100 Gbit/s ports.
4. **Per-aggregate overhead** (tranche two) — unchanged threshold (~1–2%), now with the ATM-preset restriction documented in tranche one.
5. **Capture ratio** (§3.5) — measured in D; defines whether weighted sharing ever gets revisited.

---

## 8. Sequencing and definition of done

Sequencing: **V1 lab check** → A0 spec + fixes (upstream issue #A0) → hqos-svlan spec (contribution path 2–3) → A1/A2 API freeze → B → A3–A6 with C1–C5 in parallel → D (D0 harness may start any time after A0) → docs.

Total elapsed: **5–7 weeks** for one developer plus Claude Code, including both spec-review cycles. (v0.4 said 3–5; the delta is A0, the D0 harness, the attach repair, and honest review-cycle friction.)

Definition of done, carried from v0.4 with additions:

- Refund-path rate accuracy within ±1% on both levels in the D0 multi-thread stress test.
- Zero-behaviour-change guarantee for existing subscriber-only **and** per-port-aggregate deployments — *after* A0, "existing behaviour" means the fixed semantics: attached, non-hanging, leak-free, accurate at multi-gigabit rates; containerlab 18/19 green throughout.
- A port aggregate and an S-VLAN aggregate demonstrably attached on live sessions of **both** access types (IPoE and PPPoE), surviving create/delete under load with correct re-attachment (A6 exit).
- Quiescent `buffer_usage == 0` at every level after teardown-under-load.
- Operator visibility: `show cake aggregate` tree, `show qos aggregates`, Prometheus per-level shaped/backpressure/`parent_blocked`.
- Documented limitations shipped with the feature: fairness losses (§3.5), Ethernet-presets-only accounting (§3.3), measured contention ceiling (§3.1).
