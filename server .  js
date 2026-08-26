"use strict";
/**
 * NERVE Dispatch Engine — backend
 * -------------------------------
 * The entire simulation (world generation, Dijkstra routing, dispatch
 * scoring, ambulance movement, road closures, facility state) runs here,
 * once, on the server. Every connected browser is a passive viewer/
 * controller over WebSocket — nobody runs their own copy of the sim, so
 * everyone sees the same live state.
 *
 * Protocol (JSON over ws, path /ws):
 *   server -> client : {type:'state', ...fullWorldSnapshot}
 *   client -> server : {type:'toggle'}                 // play/pause
 *                       {type:'speed', value:number}    // 0.5–4
 *                       {type:'spawn'}                  // force a new emergency
 *                       {type:'closure'}                // force a road closure
 *                       {type:'reset'}                  // rebuild the world
 */

const path = require("path");
const http = require("http");
const express = require("express");
const { WebSocketServer } = require("ws");
const { performance } = require("perf_hooks");

/* =========================================================================
   CORE ALGORITHMS — binary-heap Dijkstra, graph generation
   (ported verbatim from the original client-side simulation)
   ========================================================================= */

class MinHeap {
  constructor() { this.a = []; }
  size() { return this.a.length; }
  push(item, key) {
    const a = this.a;
    a.push({ item, key });
    let i = a.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (a[p].key <= a[i].key) break;
      [a[p], a[i]] = [a[i], a[p]];
      i = p;
    }
  }
  pop() {
    const a = this.a;
    if (!a.length) return null;
    const top = a[0];
    const last = a.pop();
    if (a.length) {
      a[0] = last;
      let i = 0;
      while (true) {
        let l = i * 2 + 1, r = i * 2 + 2, s = i;
        if (l < a.length && a[l].key < a[s].key) s = l;
        if (r < a.length && a[r].key < a[s].key) s = r;
        if (s === i) break;
        [a[s], a[i]] = [a[i], a[s]];
        i = s;
      }
    }
    return top;
  }
}

function seededRandom(seed) {
  let s = seed % 2147483647;
  if (s <= 0) s += 2147483646;
  return function () { s = (s * 16807) % 2147483647; return (s - 1) / 2147483646; };
}

function dijkstra(adj, source, n) {
  const dist = new Float64Array(n).fill(Infinity);
  const prev = new Int32Array(n).fill(-1);
  dist[source] = 0;
  const heap = new MinHeap();
  heap.push(source, 0);
  const visited = new Uint8Array(n);
  while (heap.size()) {
    const { item: u, key: d } = heap.pop();
    if (visited[u]) continue;
    visited[u] = 1;
    if (d > dist[u]) continue;
    const edges = adj[u];
    for (let k = 0; k < edges.length; k++) {
      const e = edges[k];
      if (e.blocked) continue;
      const nd = dist[u] + e.w;
      if (nd < dist[e.to]) {
        dist[e.to] = nd;
        prev[e.to] = u;
        heap.push(e.to, nd);
      }
    }
  }
  return { dist, prev };
}

function pathFromSource(prev, target) {
  const path = [];
  let cur = target;
  let guard = 0;
  while (cur !== -1 && guard++ < 100000) { path.push(cur); cur = prev[cur]; }
  return path.reverse();
}

function edgeWeight(adj, a, b) {
  const edges = adj[a];
  for (let i = 0; i < edges.length; i++) if (edges[i].to === b) return edges[i].w;
  return 0;
}

/* =========================================================================
   WORLD GENERATION
   ========================================================================= */

const SPECIALTIES = ['General Medicine', 'Cardiology', 'Orthopedics', 'Pediatrics', 'Neurology', 'Trauma Surgery'];
const MED_CATS = ['Antibiotics', 'Analgesics', 'Cardiac Meds', 'Anti-Inflammatory', 'IV Fluids', 'Trauma Kits'];
const SPEC_TO_MED = {
  'General Medicine': 'Antibiotics', 'Cardiology': 'Cardiac Meds', 'Orthopedics': 'Anti-Inflammatory',
  'Pediatrics': 'Analgesics', 'Neurology': 'IV Fluids', 'Trauma Surgery': 'Trauma Kits'
};
const URGENCY = [
  { name: 'Critical', weight: 0.15, rank: 0, slaTicks: 9, color: 'var(--red)' },
  { name: 'Urgent', weight: 0.35, rank: 1, slaTicks: 20, color: 'var(--amber)' },
  { name: 'Routine', weight: 0.50, rank: 2, slaTicks: 9999, color: 'var(--text-dim)' }
];
const HOSPITAL_NAMES = [
  'Sunrise District Hospital', 'Riverside Health Center', 'St. Meridian Medical', 'Hilltop Community Clinic',
  'Green Valley General', 'North Ridge Hospital', 'Lotus Rural Health Center', 'Union Trauma Institute',
  'Pinegrove Medical Center', 'Ashford District Hospital', 'Blue Delta Clinic', 'Cedar Point General',
  'Harbor Light Hospital', 'Wellspring Health Center'
];
const VILLAGE_SYL = ['Ka', 'Ra', 'Ni', 'Su', 'Ma', 'Ta', 'Vi', 'Lo', 'Pa', 'Do', 'Chi', 'Bel', 'Ha', 'Sa', 'Ru', 'Mi', 'Ko', 'Wa'];

function genVillageName(rand) {
  const n = 2 + Math.floor(rand() * 2);
  let s = '';
  for (let i = 0; i < n; i++) s += VILLAGE_SYL[Math.floor(rand() * VILLAGE_SYL.length)];
  return s + (rand() < 0.4 ? 'pur' : rand() < 0.5 ? 'gaon' : '');
}

const N_VILLAGES = 90;
const N_HOSPITALS = 14;
const K_NEAREST = 4;
const AMBULANCES_PER_HOSPITAL = 1;
const BASE_SPEED = 46; // distance units per tick
const BASE_TICK_MS = 1000;

function buildWorld(seed) {
  const rand = seededRandom(seed);
  const nodes = [];
  for (let i = 0; i < N_VILLAGES; i++) {
    nodes.push({ id: i, type: 'village', x: 40 + rand() * 920, y: 40 + rand() * 820, name: genVillageName(rand) });
  }
  for (let i = 0; i < N_HOSPITALS; i++) {
    const maxBeds = 6 + Math.floor(rand() * 10);
    const specialists = {};
    SPECIALTIES.forEach(s => { specialists[s] = (s === 'General Medicine') ? true : rand() < 0.42; });
    const medicine = {};
    MED_CATS.forEach(m => medicine[m] = 4 + Math.floor(rand() * 14));
    nodes.push({
      id: N_VILLAGES + i, type: 'hospital',
      x: 40 + rand() * 920, y: 40 + rand() * 820,
      name: HOSPITAL_NAMES[i % HOSPITAL_NAMES.length],
      maxBeds, beds: maxBeds, specialists, medicine, queueCount: 0, occupants: []
    });
  }
  SPECIALTIES.forEach(spec => {
    const has = nodes.some(nd => nd.type === 'hospital' && nd.specialists[spec]);
    if (!has) {
      const h = nodes.filter(nd => nd.type === 'hospital')[Math.floor(rand() * N_HOSPITALS)];
      h.specialists[spec] = true;
    }
  });

  const n = nodes.length;
  const adj = Array.from({ length: n }, () => []);
  const seen = new Set();
  function dist2(a, b) { return Math.hypot(a.x - b.x, a.y - b.y); }
  function addEdge(a, b) {
    if (a === b) return;
    const k = a < b ? a + '_' + b : b + '_' + a;
    if (seen.has(k)) return;
    seen.add(k);
    const w = dist2(nodes[a], nodes[b]) * (0.75 + rand() * 0.7);
    adj[a].push({ to: b, w, blocked: false });
    adj[b].push({ to: a, w, blocked: false });
  }
  for (let i = 0; i < n; i++) {
    const ds = [];
    for (let j = 0; j < n; j++) if (j !== i) ds.push([j, dist2(nodes[i], nodes[j])]);
    ds.sort((a, b) => a[1] - b[1]);
    for (let k = 0; k < K_NEAREST; k++) addEdge(i, ds[k][0]);
  }
  function components() {
    const comp = new Array(n).fill(-1); let c = 0;
    for (let s = 0; s < n; s++) {
      if (comp[s] !== -1) continue;
      const stack = [s]; comp[s] = c;
      while (stack.length) {
        const u = stack.pop();
        for (const e of adj[u]) if (comp[e.to] === -1) { comp[e.to] = c; stack.push(e.to); }
      }
      c++;
    }
    return { comp, c };
  }
  let { comp, c } = components();
  let guard = 0;
  while (c > 1 && guard++ < 500) {
    const zero = []; for (let i = 0; i < n; i++) if (comp[i] === 0) zero.push(i);
    let best = null, bestD = Infinity;
    for (const i of zero) for (let j = 0; j < n; j++) if (comp[j] !== 0) {
      const d = dist2(nodes[i], nodes[j]);
      if (d < bestD) { bestD = d; best = [i, j]; }
    }
    if (!best) break;
    addEdge(best[0], best[1]);
    ({ comp, c } = components());
  }

  const hospitalIds = nodes.filter(nd => nd.type === 'hospital').map(nd => nd.id);
  const ambulances = [];
  let ambId = 1;
  hospitalIds.forEach(hid => {
    for (let k = 0; k < AMBULANCES_PER_HOSPITAL; k++) {
      ambulances.push({
        id: 'A-' + String(ambId++).padStart(2, '0'),
        homeNode: hid, currentNode: hid,
        status: 'idle', route: null, cumDist: null, traveled: 0,
        x: nodes[hid].x, y: nodes[hid].y,
        targetVillage: null, targetHospital: null, requestId: null, phaseSplit: 0
      });
    }
  });
  for (let k = 0; k < 3; k++) {
    const vid = Math.floor(rand() * N_VILLAGES);
    ambulances.push({
      id: 'A-' + String(ambId++).padStart(2, '0'),
      homeNode: vid, currentNode: vid, status: 'idle', route: null, cumDist: null, traveled: 0,
      x: nodes[vid].x, y: nodes[vid].y, targetVillage: null, targetHospital: null, requestId: null, phaseSplit: 0
    });
  }

  return {
    nodes, adj, n, hospitalIds, ambulances,
    pending: [], activeDispatches: [], decisionLog: [],
    tick: 0, nextReqId: 1, blocked: [],
    metrics: { totalDispatched: 0, totalCostSum: 0, slaBreaches: 0, lastAlgoMs: 0 },
    flaggedSla: new Set()
  };
}

/* =========================================================================
   SIMULATION LOGIC (ported verbatim)
   ========================================================================= */

function nodeById(w, id) { return w.nodes[id]; }

function spawnRequest(w) {
  const rand = Math.random;
  const villageId = Math.floor(rand() * N_VILLAGES);
  const specialty = SPECIALTIES[Math.floor(rand() * SPECIALTIES.length)];
  let r = rand(), acc = 0, urgency = URGENCY[URGENCY.length - 1];
  for (const u of URGENCY) { acc += u.weight; if (r <= acc) { urgency = u; break; } }
  const req = { id: w.nextReqId++, villageId, specialty, urgency, spawnTick: w.tick, waitTicks: 0 };
  w.pending.push(req);
  logLine(w, 'info', `New request <b>#${req.id}</b> from ${nodeById(w, villageId).name} — needs ${specialty} (${urgency.name})`);
  return req;
}

/**
 * Validate and inject a request that a user submitted from the frontend
 * (as opposed to spawnRequest, which the sim generates on its own).
 * Returns {ok:true, req} on success or {ok:false, error} on bad input —
 * never throws, and never trusts the packet's shape.
 */
function submitUserRequest(w, data) {
  if (!data || typeof data !== 'object') {
    return { ok: false, error: 'Malformed request packet.' };
  }

  const villageId = Number(data.villageId);
  if (!Number.isInteger(villageId) || villageId < 0 || villageId >= N_VILLAGES) {
    return { ok: false, error: 'villageId must be an integer identifying a known village.' };
  }
  const villageNode = nodeById(w, villageId);
  if (!villageNode || villageNode.type !== 'village') {
    return { ok: false, error: 'villageId does not refer to a village.' };
  }

  const specialty = String(data.specialty || '');
  if (!SPECIALTIES.includes(specialty)) {
    return { ok: false, error: `specialty must be one of: ${SPECIALTIES.join(', ')}` };
  }

  const urgencyName = String(data.urgency || '');
  const urgency = URGENCY.find(u => u.name === urgencyName);
  if (!urgency) {
    return { ok: false, error: `urgency must be one of: ${URGENCY.map(u => u.name).join(', ')}` };
  }

  // basic abuse guard — cap total outstanding pending requests
  if (w.pending.length >= 400) {
    return { ok: false, error: 'Too many pending requests already queued — try again shortly.' };
  }

  const req = {
    id: w.nextReqId++,
    villageId, specialty, urgency,
    spawnTick: w.tick, waitTicks: 0,
    source: 'operator'
  };
  w.pending.push(req);
  logLine(w, 'info', `New request <b>#${req.id}</b> from ${villageNode.name} — needs ${specialty} (${urgency.name}) [operator-submitted]`);
  return { ok: true, req };
}

function scoreRequest(req) { return req.urgency.rank * 1000 - req.waitTicks * 4; }

function logLine(w, kind, msg) {
  w.decisionLog.unshift({ kind, msg, tick: w.tick, ts: Date.now() });
  if (w.decisionLog.length > 150) w.decisionLog.length = 150;
}

function fmtClock(ticks) {
  const totalSec = ticks * 4;
  const m = Math.floor(totalSec / 60).toString().padStart(2, '0');
  const s = (totalSec % 60).toString().padStart(2, '0');
  return m + ':' + s;
}

function attemptDispatch(w, req, distCache) {
  let dp = distCache.get(req.villageId);
  const t0 = performance.now();
  if (!dp) { dp = dijkstra(w.adj, req.villageId, w.n); distCache.set(req.villageId, dp); }
  w.metrics.lastAlgoMs = performance.now() - t0;
  const { dist, prev } = dp;

  const hospitals = w.hospitalIds.map(id => nodeById(w, id)).filter(h => h.beds > 0 && isFinite(dist[h.id]));
  if (hospitals.length === 0) {
    logLine(w, 'warn', `Req <b>#${req.id}</b> (${nodeById(w, req.villageId).name}): no reachable hospital with free beds — holding in queue (fallback routing standby)`);
    return false;
  }

  let pool = hospitals.filter(h => h.specialists[req.specialty]);
  let matched = true;
  if (pool.length === 0) {
    matched = false;
    pool = hospitals.filter(h => h.specialists['Trauma Surgery'] || h.specialists['General Medicine']);
    if (pool.length === 0) pool = hospitals;
  }

  function cost(h) {
    const travel = dist[h.id];
    const wait = (h.maxBeds - h.beds) * 5 + h.queueCount * 6;
    const medCat = SPEC_TO_MED[req.specialty];
    const medPenalty = (h.medicine[medCat] <= 2) ? 45 : 0;
    return { total: travel + wait + medPenalty, travel, wait, medPenalty };
  }

  const ranked = pool.map(h => ({ h, c: cost(h) })).sort((a, b) => a.c.total - b.c.total);
  const chosen = ranked[0];

  const idle = w.ambulances.filter(a => a.status === 'idle');
  if (idle.length === 0) {
    const top3 = ranked.slice(0, 3).map(r => `${r.h.name.split(' ')[0]}(${r.c.total.toFixed(0)})`).join(', ');
    logLine(w, 'warn', `Req <b>#${req.id}</b>: best facility ${chosen.h.name} selected, but <b>all ambulances occupied</b> — queued. Candidates: ${top3}`);
    return false;
  }
  const rankedAmb = idle.map(a => ({ a, d: dist[a.currentNode] })).sort((x, y) => x.d - y.d);
  const chosenAmb = rankedAmb[0];

  const villagePathFromAmb = pathFromSource(prev, chosenAmb.a.currentNode).reverse();
  const hospitalPathFromVillage = pathFromSource(prev, chosen.h.id);
  const fullRoute = villagePathFromAmb.concat(hospitalPathFromVillage.slice(1));
  const cumDist = [0];
  for (let i = 1; i < fullRoute.length; i++) {
    cumDist.push(cumDist[i - 1] + edgeWeight(w.adj, fullRoute[i - 1], fullRoute[i]));
  }

  const amb = chosenAmb.a;
  amb.status = 'to_patient';
  amb.route = fullRoute;
  amb.cumDist = cumDist;
  amb.traveled = 0;
  amb.phaseSplit = cumDist[villagePathFromAmb.length - 1];
  amb.targetVillage = req.villageId;
  amb.targetHospital = chosen.h.id;
  amb.requestId = req.id;
  chosen.h.queueCount++;

  w.activeDispatches.push({ reqId: req.id, ambId: amb.id, hospitalId: chosen.h.id, villageId: req.villageId, spawnTick: req.spawnTick, urgency: req.urgency, specialty: req.specialty });
  w.metrics.totalDispatched++;
  w.metrics.totalCostSum += chosen.c.total;

  const reasonBits = [];
  if (!matched) reasonBits.push('no on-duty specialist nearby — routed to nearest capable facility');
  if (chosen.c.medPenalty > 0) reasonBits.push('low medicine stock flagged, still admitted');
  const otherCandidate = ranked[1];
  let compareTxt = otherCandidate ? ` (rejected ${otherCandidate.h.name.split(' ')[0]}: cost ${otherCandidate.c.total.toFixed(0)} vs chosen ${chosen.c.total.toFixed(0)})` : '';

  logLine(w, 'dispatch',
    `Req <b>#${req.id}</b> ${nodeById(w, req.villageId).name} → <b>${chosen.h.name}</b> [travel ${chosen.c.travel.toFixed(0)} + wait ${chosen.c.wait.toFixed(0)}${chosen.c.medPenalty ? ' + med-risk ' + chosen.c.medPenalty : ''} = ${chosen.c.total.toFixed(0)}]${compareTxt}. Ambulance <b>${amb.id}</b> dispatched (${chosenAmb.d.toFixed(0)} units away).${reasonBits.length ? ' ⚠ ' + reasonBits.join('; ') : ''}`
  );

  if (req.urgency.rank === 0 && req.waitTicks > req.urgency.slaTicks) {
    w.metrics.slaBreaches++;
    logLine(w, 'crit', `SLA BREACH — critical req <b>#${req.id}</b> waited ${req.waitTicks * 4}s before dispatch (limit ${req.urgency.slaTicks * 4}s)`);
  }

  return true;
}

function processQueue(w) {
  if (w.pending.length === 0) return;
  const heap = new MinHeap();
  w.pending.forEach(r => heap.push(r, scoreRequest(r)));
  const ordered = [];
  while (heap.size()) ordered.push(heap.pop().item);

  const distCache = new Map();
  const stillPending = [];
  for (const req of ordered) {
    const ok = attemptDispatch(w, req, distCache);
    if (!ok) {
      req.waitTicks++;
      if (req.urgency.rank === 0 && req.waitTicks > req.urgency.slaTicks && !w.flaggedSla.has(req.id)) {
        w.flaggedSla.add(req.id);
        w.metrics.slaBreaches++;
        logLine(w, 'crit', `SLA BREACH — critical req <b>#${req.id}</b> from ${nodeById(w, req.villageId).name} still waiting (${req.waitTicks * 4}s)`);
      }
      stillPending.push(req);
    }
  }
  w.pending = stillPending;
}

function setEdgeBlocked(w, a, b, val) {
  for (const e of w.adj[a]) if (e.to === b) e.blocked = val;
  for (const e of w.adj[b]) if (e.to === a) e.blocked = val;
}

function triggerRandomClosure(w) {
  const candidates = [];
  for (let a = 0; a < w.n; a++) for (const e of w.adj[a]) if (a < e.to && !e.blocked) candidates.push([a, e.to]);
  if (candidates.length === 0) return;
  const [a, b] = candidates[Math.floor(Math.random() * candidates.length)];
  setEdgeBlocked(w, a, b, true);
  const reopenTick = w.tick + 10 + Math.floor(Math.random() * 14);
  w.blocked.push({ a, b, aName: nodeById(w, a).name, bName: nodeById(w, b).name, reopenTick });
  logLine(w, 'warn', `Road closure: <b>${nodeById(w, a).name} – ${nodeById(w, b).name}</b> blocked. Active/future routes rerouting around it.`);
}

function updateRoadClosures(w) {
  w.blocked = w.blocked.filter(b => {
    if (w.tick >= b.reopenTick) {
      setEdgeBlocked(w, b.a, b.b, false);
      logLine(w, 'success', `Road reopened between ${b.aName} and ${b.bName}`);
      return false;
    }
    return true;
  });
  if (w.tick > 0 && w.tick % 11 === 0 && Math.random() < 0.55) {
    triggerRandomClosure(w);
  }
}

function updateAmbulances(w, speedUnitsPerTick) {
  for (const amb of w.ambulances) {
    if (amb.status === 'idle') continue;
    amb.traveled += speedUnitsPerTick;
    const cum = amb.cumDist, route = amb.route;
    const total = cum[cum.length - 1];

    if (amb.status === 'to_patient' && amb.traveled >= amb.phaseSplit) {
      amb.status = 'to_hospital';
    }

    if (amb.traveled >= total) {
      if (amb.status === 'to_hospital') {
        const hosp = nodeById(w, amb.targetHospital);
        hosp.beds = Math.max(0, hosp.beds - 1);
        hosp.queueCount = Math.max(0, hosp.queueCount - 1);
        hosp.occupants.push(w.tick + 14 + Math.floor(Math.random() * 22));
        const medCat = SPEC_TO_MED[(w.activeDispatches.find(d => d.ambId === amb.id) || {}).specialty] || 'Antibiotics';
        hosp.medicine[medCat] = Math.max(0, hosp.medicine[medCat] - 1);
        logLine(w, 'success', `Ambulance <b>${amb.id}</b> delivered patient to <b>${hosp.name}</b> — bed occupied (${hosp.beds}/${hosp.maxBeds} free)`);
        w.activeDispatches = w.activeDispatches.filter(d => d.ambId !== amb.id);
        const { prev } = dijkstra(w.adj, amb.targetHospital, w.n);
        const homePath = pathFromSource(prev, amb.homeNode);
        const cumD = [0];
        for (let i = 1; i < homePath.length; i++) cumD.push(cumD[i - 1] + edgeWeight(w.adj, homePath[i - 1], homePath[i]));
        amb.status = 'returning';
        amb.route = homePath; amb.cumDist = cumD; amb.traveled = 0;
        amb.currentNode = amb.targetHospital;
      } else if (amb.status === 'returning') {
        amb.status = 'idle';
        amb.currentNode = amb.homeNode;
        amb.route = null; amb.cumDist = null; amb.traveled = 0;
        amb.x = nodeById(w, amb.homeNode).x; amb.y = nodeById(w, amb.homeNode).y;
      }
      continue;
    }
    let idx = 0;
    while (idx < cum.length - 2 && cum[idx + 1] < amb.traveled) idx++;
    const segStart = cum[idx], segEnd = cum[idx + 1] || segStart;
    const frac = segEnd > segStart ? (amb.traveled - segStart) / (segEnd - segStart) : 0;
    const nA = nodeById(w, route[idx]), nB = nodeById(w, route[Math.min(idx + 1, route.length - 1)]);
    amb.x = nA.x + (nB.x - nA.x) * frac;
    amb.y = nA.y + (nB.y - nA.y) * frac;
    if (amb.status !== 'returning') amb.currentNode = route[idx];
  }
}

function updateFacilities(w) {
  for (const id of w.hospitalIds) {
    const h = nodeById(w, id);
    if (h.occupants.length) {
      const staying = [];
      let freed = 0;
      for (const dTick of h.occupants) {
        if (w.tick >= dTick) freed++; else staying.push(dTick);
      }
      if (freed) {
        h.beds = Math.min(h.maxBeds, h.beds + freed);
        h.occupants = staying;
        logLine(w, 'info', `${freed} patient${freed > 1 ? 's' : ''} discharged from <b>${h.name}</b> — ${h.beds}/${h.maxBeds} beds now free`);
      }
    }
    if ((w.tick + id) % 26 === 0) {
      let restocked = false;
      MED_CATS.forEach(m => {
        if (h.medicine[m] < 16) { h.medicine[m] = Math.min(20, h.medicine[m] + 3 + Math.floor(Math.random() * 4)); restocked = true; }
      });
      if (restocked) logLine(w, 'info', `Supply run: medicine restocked at <b>${h.name}</b>`);
    }
  }
}

function simTick(w) {
  w.tick++;
  if (Math.random() < 0.45) spawnRequest(w);
  updateRoadClosures(w);
  updateFacilities(w);
  processQueue(w);
  updateAmbulances(w, BASE_SPEED);
}

/* =========================================================================
   SHARED SERVER STATE + TICK LOOP
   ========================================================================= */

let world = null;
let running = true;
let speedMult = 1;
let tickTimer = null;

function initWorld() {
  world = buildWorld(Date.now() % 1000000);
  logLine(world, 'info', `Simulation initialized — ${world.n} nodes, ${world.adj.reduce((s, a) => s + a.length, 0) / 2} weighted edges, ${world.ambulances.length} ambulances, ${world.hospitalIds.length} facilities.`);
}

function stopLoop() { if (tickTimer) clearInterval(tickTimer); }

function startLoop() {
  stopLoop();
  tickTimer = setInterval(() => {
    if (!running) return;
    simTick(world);
    broadcastState();
  }, BASE_TICK_MS / speedMult);
}

/* =========================================================================
   HTTP + WEBSOCKET SERVER
   ========================================================================= */

const app = express();
app.use(express.static(path.join(__dirname, "public")));
app.get("/health", (req, res) => res.json({ ok: true, tick: world ? world.tick : null }));

const server = http.createServer(app);
const wss = new WebSocketServer({ server, path: "/ws" });

function snapshot() {
  return {
    type: "state",
    tick: world.tick,
    running, speedMult,
    n: world.n,
    nodes: world.nodes,
    adj: world.adj,
    hospitalIds: world.hospitalIds,
    ambulances: world.ambulances,
    pending: world.pending,
    activeDispatches: world.activeDispatches,
    decisionLog: world.decisionLog,
    metrics: world.metrics
  };
}

function broadcastState() {
  const payload = JSON.stringify(snapshot());
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(payload);
  }
}

function broadcastMeta() {
  // village ids/names change on reset (whole new world is generated), so
  // every connected client needs a fresh copy to keep its request form valid
  const payload = JSON.stringify({
    type: "meta",
    specialties: SPECIALTIES,
    urgencies: URGENCY.map(u => u.name),
    villages: world.nodes.filter(nd => nd.type === "village").map(nd => ({ id: nd.id, name: nd.name }))
  });
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(payload);
  }
}

const MAX_PACKET_BYTES = 4096; // any legitimate control message is tiny — reject anything bigger
const MSG_RATE_LIMIT = 20;     // max inbound packets per connection per rateWindowMs
const MSG_RATE_WINDOW_MS = 5000;

wss.on("connection", (ws) => {
  // sent once per connection (not every tick) — static reference data the
  // frontend needs to build its request form: valid specialties/urgencies
  // and the village list (id + name) to populate a picker.
  ws.send(JSON.stringify({
    type: "meta",
    specialties: SPECIALTIES,
    urgencies: URGENCY.map(u => u.name),
    villages: world.nodes.filter(nd => nd.type === "village").map(nd => ({ id: nd.id, name: nd.name }))
  }));
  ws.send(JSON.stringify(snapshot()));

  // per-connection rate limiting so one client can't flood the sim with packets
  ws._msgTimestamps = [];

  ws.on("message", (raw) => {
    if (raw.length > MAX_PACKET_BYTES) {
      ws.send(JSON.stringify({ type: "error", message: "Packet too large." }));
      return;
    }

    const now = Date.now();
    ws._msgTimestamps = ws._msgTimestamps.filter(t => now - t < MSG_RATE_WINDOW_MS);
    if (ws._msgTimestamps.length >= MSG_RATE_LIMIT) {
      ws.send(JSON.stringify({ type: "error", message: "Slow down — too many requests." }));
      return;
    }
    ws._msgTimestamps.push(now);

    let msg;
    try { msg = JSON.parse(raw); } catch {
      ws.send(JSON.stringify({ type: "error", message: "Invalid JSON packet." }));
      return;
    }
    if (!msg || typeof msg.type !== "string") {
      ws.send(JSON.stringify({ type: "error", message: "Packet missing a valid 'type' field." }));
      return;
    }

    switch (msg.type) {
      case "toggle":
        running = !running;
        broadcastState();
        break;
      case "speed":
        speedMult = Math.min(4, Math.max(0.5, Number(msg.value) || 1));
        startLoop();
        broadcastState();
        break;
      case "spawn":
        spawnRequest(world);
        broadcastState();
        break;
      case "custom_request": {
        // this is the path that carries actual user input from the
        // frontend's request form: {villageId, specialty, urgency}
        const result = submitUserRequest(world, msg);
        if (result.ok) {
          broadcastState();
        } else {
          ws.send(JSON.stringify({ type: "error", message: result.error }));
        }
        break;
      }
      case "closure":
        triggerRandomClosure(world);
        broadcastState();
        break;
      case "reset":
        initWorld();
        broadcastMeta();
        broadcastState();
        break;
      default:
        break;
    }
  });
});

/* =========================================================================
   BOOT
   ========================================================================= */

const PORT = process.env.PORT || 3000;
initWorld();
startLoop();
server.listen(PORT, "0.0.0.0", () => {
  console.log(`NERVE backend listening on :${PORT}`);
});
