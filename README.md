# TransitOps
Smart Transport Operations Platform for Odoo Hackathon 2026


/app/frontend/src/App.js --file-text "import React from \"react\";
import \"./App.css\";
import { BrowserRouter, Routes, Route, Navigate } from \"react-router-dom\";
import { AuthProvider, useAuth } from \"./lib/auth\";
import { Toaster } from \"sonner\";
import Layout from \"./components/Layout\";
import Login from \"./pages/Login\";
import Dashboard from \"./pages/Dashboard\";
import Dispatch from \"./pages/Dispatch\";
import Emergencies from \"./pages/Emergencies\";
import Ambulances from \"./pages/Ambulances\";
import Drivers from \"./pages/Drivers\";
import Hospitals from \"./pages/Hospitals\";
import Maintenance from \"./pages/Maintenance\";
import FuelExpenses from \"./pages/FuelExpenses\";
import Risk from \"./pages/Risk\";
import LiveMap from \"./pages/LiveMap\";
import Reports from \"./pages/Reports\";

function Protected({ children }) {
  const { user, ready } = useAuth();
  if (!ready) return <div className=\"min-h-screen flex items-center justify-center text-white/40 text-sm\">Loading...</div>;
  if (!user) return <Navigate to=\"/login\" replace />;
  return <Layout>{children}</Layout>;
}

export default function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Toaster theme=\"dark\" position=\"top-right\" richColors />
        <Routes>
          <Route path=\"/login\" element={<Login />} />
          <Route path=\"/\" element={<Protected><Dashboard /></Protected>} />
          <Route path=\"/dispatch\" element={<Protected><Dispatch /></Protected>} />
          <Route path=\"/emergencies\" element={<Protected><Emergencies /></Protected>} />
          <Route path=\"/ambulances\" element={<Protected><Ambulances /></Protected>} />
          <Route path=\"/drivers\" element={<Protected><Drivers /></Protected>} />
          <Route path=\"/hospitals\" element={<Protected><Hospitals /></Protected>} />
          <Route path=\"/maintenance\" element={<Protected><Maintenance /></Protected>} />
          <Route path=\"/fuel\" element={<Protected><FuelExpenses /></Protected>} />
          <Route path=\"/risk\" element={<Protected><Risk /></Protected>} />
          <Route path=\"/map\" element={<Protected><LiveMap /></Protected>} />
          <Route path=\"/reports\" element={<Protected><Reports /></Protected>} />
          <Route path=\"*\" element={<Navigate to=\"/\" replace />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
"
Observation: Overwrite successful: /app/frontend/src/App.js

import React, { useState, useEffect, useRef, useCallback } from "react";
import { Truck, Building2, AlertTriangle, Navigation2, Radio, Clock3, Home } from "lucide-react";

// ---------- Map topology ----------
const DEPOT = { id: "depot", x: 70, y: 360, label: "Central Depot", type: "depot" };
const ZONES = [
  { id: "downtown", x: 260, y: 220, label: "Downtown", type: "zone" },
  { id: "northridge", x: 420, y: 80, label: "North Ridge", type: "zone" },
  { id: "harbor", x: 540, y: 280, label: "Harbor District", type: "zone" },
  { id: "university", x: 150, y: 90, label: "University Row", type: "zone" },
  { id: "westend", x: 90, y: 200, label: "West End", type: "zone" },
  { id: "industrial", x: 430, y: 340, label: "Industrial Park", type: "zone" },
];
const HOSPITALS = [
  { id: "riverside", x: 300, y: 40, label: "Riverside General", type: "hospital" },
  { id: "stmarys", x: 560, y: 150, label: "St. Mary's Trauma", type: "hospital" },
];
const ALL_NODES = [DEPOT, ...ZONES, ...HOSPITALS];
const NODE_BY_ID = Object.fromEntries(ALL_NODES.map((n) => [n.id, n]));
const ROADS = [
  ["depot", "westend"], ["depot", "downtown"], ["depot", "industrial"],
  ["westend", "university"], ["westend", "downtown"],
  ["university", "riverside"], ["university", "downtown"],
  ["downtown", "northridge"], ["downtown", "industrial"],
  ["northridge", "riverside"], ["northridge", "stmarys"],
  ["harbor", "stmarys"], ["harbor", "industrial"],
  ["industrial", "stmarys"],
];

const DRIVERS = [
  "Officer R. Alvarez", "Officer T. Chen", "Officer S. Kapoor",
  "Officer J. Whitfield", "Officer L. Okafor", "Officer M. Novak",
];

const dist = (p1, p2) => Math.hypot(p1.x - p2.x, p1.y - p2.y);
const legTicks = (p1, p2) => Math.max(6, Math.min(26, Math.round(dist(p1, p2) / 22)));
const nearestHospital = (pos) =>
  HOSPITALS.reduce((best, h) => (dist(pos, h) < dist(pos, best) ? h : best), HOSPITALS[0]);
const fmtCountdown = (secs) => {
  const s = Math.max(0, Math.round(secs));
  const m = Math.floor(s / 60);
  const r = s % 60;
  return `${m}:${r.toString().padStart(2, "0")}`;
};

const STATUS_META = {
  available: { label: "Available", color: "#2DD4BF" },
  dispatched: { label: "En Route to Scene", color: "#F5A623" },
  at_scene: { label: "On Scene", color: "#F5A623" },
  transporting: { label: "Transporting Patient", color: "#EF4444" },
  at_hospital: { label: "At Hospital", color: "#F5A623" },
  returning: { label: "Returning to Depot", color: "#64748B" },
};

const PRIORITY_META = {
  Critical: { color: "#EF4444" },
  Urgent: { color: "#F5A623" },
  Routine: { color: "#2DD4BF" },
};

function makeInitialAmbulances() {
  return Array.from({ length: 6 }).map((_, i) => ({
    id: i + 1,
    callsign: `MED-${i + 1}`,
    driver: DRIVERS[i],
    status: "available",
    pos: { x: DEPOT.x, y: DEPOT.y },
    origin: null,
    dest: null,
    destLabel: null,
    progress: 0,
    totalTicks: 0,
    ticksElapsed: 0,
    sceneTimer: 0,
    incidentId: null,
  }));
}

export default function App() {
  const [ambulances, setAmbulances] = useState(makeInitialAmbulances);
  const [incidents, setIncidents] = useState([]);
  const [resolvedLog, setResolvedLog] = useState([]);
  const [clock, setClock] = useState(new Date());
  const idCounter = useRef(1);
  const tickCounter = useRef(0);

  const spawnIncident = useCallback(() => {
    const zone = ZONES[Math.floor(Math.random() * ZONES.length)];
    const roll = Math.random();
    const priority = roll < 0.22 ? "Critical" : roll < 0.55 ? "Urgent" : "Routine";
    const id = idCounter.current++;
    return {
      id,
      zoneLabel: zone.label,
      pos: { x: zone.x, y: zone.y },
      priority,
      status: "pending",
      waitTicks: 0,
    };
  }, []);

  useEffect(() => {
    setIncidents([spawnIncident()]);
    const t = setTimeout(() => setIncidents((prev) => [...prev, spawnIncident()]), 3000);
    return () => clearTimeout(t);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  useEffect(() => {
    const interval = setInterval(() => {
      setClock(new Date());
      tickCounter.current += 1;

      const arrivedAtHospitalIncidentIds = [];

      setAmbulances((prev) =>
        prev.map((a) => {
          if (a.status === "available") return a;

          if (a.status === "at_scene" || a.status === "at_hospital") {
            if (a.sceneTimer > 1) return { ...a, sceneTimer: a.sceneTimer - 1 };
            if (a.status === "at_scene") {
              const hosp = nearestHospital(a.pos);
              return {
                ...a,
                status: "transporting",
                origin: a.pos,
                dest: { x: hosp.x, y: hosp.y },
                destLabel: hosp.label,
                progress: 0,
                totalTicks: legTicks(a.pos, hosp),
                ticksElapsed: 0,
                sceneTimer: 0,
              };
            }
            arrivedAtHospitalIncidentIds.push(a.incidentId);
            return {
              ...a,
              status: "returning",
              origin: a.pos,
              dest: { x: DEPOT.x, y: DEPOT.y },
              destLabel: DEPOT.label,
              progress: 0,
              totalTicks: legTicks(a.pos, DEPOT),
              ticksElapsed: 0,
              sceneTimer: 0,
              incidentId: null,
            };
          }

          const ticksElapsed = a.ticksElapsed + 1;
          const progress = Math.min(1, ticksElapsed / a.totalTicks);
          const pos = {
            x: a.origin.x + (a.dest.x - a.origin.x) * progress,
            y: a.origin.y + (a.dest.y - a.origin.y) * progress,
          };

          if (progress >= 1) {
            if (a.status === "dispatched") {
              return { ...a, status: "at_scene", pos, progress: 1, ticksElapsed, sceneTimer: 4 };
            }
            if (a.status === "transporting") {
              return { ...a, status: "at_hospital", pos, progress: 1, ticksElapsed, sceneTimer: 3 };
            }
            if (a.status === "returning") {
              return {
                ...a, status: "available", pos, progress: 0, origin: null, dest: null,
                destLabel: null, totalTicks: 0, ticksElapsed: 0,
              };
            }
          }
          return { ...a, pos, progress, ticksElapsed };
        })
      );

      setIncidents((prev) => {
        let updated = prev.map((inc) =>
          inc.status === "pending" ? { ...inc, waitTicks: inc.waitTicks + 1 } : inc
        );
        if (arrivedAtHospitalIncidentIds.length) {
          const resolvedNow = updated.filter((i) => arrivedAtHospitalIncidentIds.includes(i.id));
          if (resolvedNow.length) {
            setResolvedLog((log) => [...resolvedNow.map((r) => ({ ...r, status: "resolved" })), ...log].slice(0, 5));
          }
          updated = updated.filter((i) => !arrivedAtHospitalIncidentIds.includes(i.id));
        }
        const pendingCount = updated.filter((i) => i.status === "pending").length;
        if (pendingCount < 4 && Math.random() < 0.16) {
          updated = [...updated, spawnIncident()];
        }
        return updated;
      });
    }, 1000);
    return () => clearInterval(interval);
  }, [spawnIncident]);

  const dispatch = (ambulanceId, incident) => {
    setAmbulances((prev) =>
      prev.map((a) => {
        if (a.id !== ambulanceId) return a;
        return {
          ...a,
          status: "dispatched",
          origin: a.pos,
          dest: incident.pos,
          destLabel: incident.zoneLabel,
          progress: 0,
          totalTicks: legTicks(a.pos, incident.pos),
          ticksElapsed: 0,
          incidentId: incident.id,
        };
      })
    );
    setIncidents((prev) =>
      prev.map((i) => (i.id === incident.id ? { ...i, status: "assigned", assignedCallsign: callsignOf(ambulanceId) } : i))
    );
  };

  const callsignOf = (id) => ambulances.find((a) => a.id === id)?.callsign ?? "";

  const autoDispatch = (incident) => {
    const available = ambulances.filter((a) => a.status === "available");
    if (!available.length) return;
    const nearest = available.reduce((best, a) => (dist(a.pos, incident.pos) < dist(best.pos, incident.pos) ? a : best), available[0]);
    dispatch(nearest.id, incident);
  };

  const availableCount = ambulances.filter((a) => a.status === "available").length;
  const pendingCount = incidents.filter((i) => i.status === "pending").length;

  return (
    <div className="min-h-screen w-full bg-[#0B1220] text-[#E8ECF1]" style={{ fontFamily: "Inter, sans-serif" }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap');
        .display-font { font-family: 'Space Grotesk', sans-serif; }
        .mono-font { font-family: 'JetBrains Mono', monospace; }
        .route-line { stroke-dasharray: 6 6; animation: dash 0.8s linear infinite; }
        @keyframes dash { to { stroke-dashoffset: -24; } }
        .scrollbar-thin::-webkit-scrollbar { width: 6px; }
        .scrollbar-thin::-webkit-scrollbar-thumb { background: #1E2A40; border-radius: 3px; }
      `}</style>

      <div className="max-w-[1400px] mx-auto p-4 md:p-6">
        {/* Header */}
        <div className="flex flex-wrap items-center justify-between gap-3 border-b border-[#1E2A40] pb-4 mb-5">
          <div>
            <h1 className="display-font text-xl md:text-2xl font-semibold tracking-wide">FLEET OPS</h1>
            <p className="text-xs text-[#64748B] mt-0.5">Live Ambulance Dispatch Console</p>
          </div>
          <div className="flex items-center gap-5">
            <div className="flex items-center gap-2">
              <span className="relative flex h-2.5 w-2.5">
                <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-[#2DD4BF] opacity-75"></span>
                <span className="relative inline-flex rounded-full h-2.5 w-2.5 bg-[#2DD4BF]"></span>
              </span>
              <span className="text-xs text-[#7C8CA6] uppercase tracking-wider">Live</span>
            </div>
            <div className="mono-font text-sm text-[#E8ECF1]">
              {clock.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit", second: "2-digit" })}
            </div>
            <div className="flex gap-4 text-xs mono-font">
              <div><span className="text-[#2DD4BF] font-medium">{availableCount}</span><span className="text-[#64748B]">/{ambulances.length} units</span></div>
              <div><span className="text-[#F5A623] font-medium">{pendingCount}</span><span className="text-[#64748B]"> pending</span></div>
            </div>
          </div>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-[280px_1fr_320px] gap-4">
          {/* Fleet list */}
          <div className="bg-[#111A2B] border border-[#1E2A40] rounded-lg p-3 order-2 lg:order-1">
            <h2 className="display-font text-xs uppercase tracking-wider text-[#7C8CA6] mb-3 px-1">Fleet</h2>
            <div className="space-y-2 max-h-[560px] overflow-y-auto scrollbar-thin pr-1">
              {ambulances.map((a) => {
                const meta = STATUS_META[a.status];
                const remaining = a.totalTicks ? (a.totalTicks - a.ticksElapsed) : 0;
                return (
                  <div key={a.id} className="bg-[#0E1626] border border-[#1E2A40] rounded-md p-2.5">
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <Truck size={14} color={meta.color} />
                        <span className="display-font text-sm font-medium">{a.callsign}</span>
                      </div>
                      <span className="text-[10px] mono-font" style={{ color: meta.color }}>{meta.label}</span>
                    </div>
                    <p className="text-[11px] text-[#64748B] mt-1">{a.driver}</p>
                    {a.status !== "available" && (
                      <>
                        <p className="text-[11px] text-[#7C8CA6] mt-1">→ {a.destLabel}</p>
                        <div className="flex items-center justify-between mt-1.5">
                          <div className="h-1 bg-[#1E2A40] rounded-full flex-1 mr-2 overflow-hidden">
                            <div className="h-full rounded-full" style={{ width: `${a.progress * 100}%`, backgroundColor: meta.color }} />
                          </div>
                          <span className="text-[10px] mono-font text-[#64748B]">{fmtCountdown(remaining)}</span>
                        </div>
                      </>
                    )}
                  </div>
                );
              })}
            </div>
          </div>

          {/* Map */}
          <div className="bg-[#111A2B] border border-[#1E2A40] rounded-lg p-3 order-1 lg:order-2">
            <h2 className="display-font text-xs uppercase tracking-wider text-[#7C8CA6] mb-2 px-1">Live Map</h2>
            <svg viewBox="0 0 640 420" className="w-full h-auto">
              {ROADS.map(([a, b], i) => {
                const n1 = NODE_BY_ID[a], n2 = NODE_BY_ID[b];
                return <line key={i} x1={n1.x} y1={n1.y} x2={n2.x} y2={n2.y} stroke="#1E2A40" strokeWidth="2" />;
              })}

              {ambulances.filter((a) => a.status !== "available" && a.origin && a.dest).map((a) => (
                <line key={`route-${a.id}`} x1={a.origin.x} y1={a.origin.y} x2={a.dest.x} y2={a.dest.y}
                  stroke={STATUS_META[a.status].color} strokeWidth="1.5" className="route-line" opacity="0.6" />
              ))}

              {ALL_NODES.map((n) => (
                <g key={n.id}>
                  {n.type === "hospital" && (
                    <>
                      <circle cx={n.x} cy={n.y} r="9" fill="#0E1626" stroke="#E8ECF1" strokeWidth="1.5" />
                      <line x1={n.x - 4} y1={n.y} x2={n.x + 4} y2={n.y} stroke="#E8ECF1" strokeWidth="1.5" />
                      <line x1={n.x} y1={n.y - 4} x2={n.x} y2={n.y + 4} stroke="#E8ECF1" strokeWidth="1.5" />
                    </>
                  )}
                  {n.type === "depot" && <rect x={n.x - 7} y={n.y - 7} width="14" height="14" fill="#0E1626" stroke="#64748B" strokeWidth="1.5" />}
                  {n.type === "zone" && <circle cx={n.x} cy={n.y} r="4" fill="#1E2A40" stroke="#64748B" strokeWidth="1" />}
                  <text x={n.x} y={n.y - 12} fontSize="9" fill="#7C8CA6" textAnchor="middle" fontFamily="JetBrains Mono, monospace">{n.label}</text>
                </g>
              ))}

              {incidents.filter((i) => i.status === "pending").map((i) => (
                <circle key={`inc-${i.id}`} cx={i.pos.x} cy={i.pos.y} r="7" fill="none" stroke={PRIORITY_META[i.priority].color} strokeWidth="2">
                  <animate attributeName="r" values="6;11;6" dur="1.6s" repeatCount="indefinite" />
                  <animate attributeName="opacity" values="1;0.2;1" dur="1.6s" repeatCount="indefinite" />
                </circle>
              ))}

              {ambulances.map((a) => (
                <g key={`unit-${a.id}`}>
                  <circle cx={a.pos.x} cy={a.pos.y} r="6" fill={STATUS_META[a.status].color} stroke="#0B1220" strokeWidth="1.5" />
                  <text x={a.pos.x} y={a.pos.y + 16} fontSize="8" fill="#E8ECF1" textAnchor="middle" fontFamily="JetBrains Mono, monospace">{a.callsign}</text>
                </g>
              ))}
            </svg>
            <div className="flex flex-wrap gap-x-4 gap-y-1.5 mt-2 px-1 text-[10px] text-[#64748B]">
              <span className="flex items-center gap-1"><span className="w-2 h-2 rounded-full inline-block" style={{ background: "#2DD4BF" }} />Available</span>
              <span className="flex items-center gap-1"><span className="w-2 h-2 rounded-full inline-block" style={{ background: "#F5A623" }} />En route / on scene</span>
              <span className="flex items-center gap-1"><span className="w-2 h-2 rounded-full inline-block" style={{ background: "#EF4444" }} />Transporting patient</span>
              <span className="flex items-center gap-1"><Building2 size={11} />Hospital</span>
              <span className="flex items-center gap-1"><Home size={11} />Depot</span>
            </div>
          </div>

          {/* Incidents */}
          <div className="bg-[#111A2B] border border-[#1E2A40] rounded-lg p-3 order-3">
            <h2 className="display-font text-xs uppercase tracking-wider text-[#7C8CA6] mb-3 px-1">Incident Queue</h2>
            <div className="space-y-2 max-h-[260px] overflow-y-auto scrollbar-thin pr-1">
              {incidents.length === 0 && <p className="text-xs text-[#64748B] px-1">No active calls.</p>}
              {incidents.map((inc) => {
                const pMeta = PRIORITY_META[inc.priority];
                const available = ambulances.filter((a) => a.status === "available");
                return (
                  <div key={inc.id} className="bg-[#0E1626] border border-[#1E2A40] rounded-md p-2.5">
                    <div className="flex items-center justify-between">
                      <span className="text-[10px] font-medium mono-font px-1.5 py-0.5 rounded" style={{ color: pMeta.color, border: `1px solid ${pMeta.color}55` }}>
                        {inc.priority.toUpperCase()}
                      </span>
                      <span className="text-[10px] text-[#64748B] mono-font">wait {fmtCountdown(inc.waitTicks)}</span>
                    </div>
                    <p className="text-sm mt-1.5">{inc.zoneLabel}</p>
                    {inc.status === "pending" ? (
                      <button
                        onClick={() => autoDispatch(inc)}
                        disabled={available.length === 0}
                        className="mt-2 w-full text-xs py-1.5 rounded-md border border-[#2DD4BF]/40 text-[#2DD4BF] disabled:text-[#64748B] disabled:border-[#1E2A40] hover:bg-[#2DD4BF]/10 transition-colors"
                      >
                        {available.length === 0 ? "No units available" : "Auto-dispatch nearest"}
                      </button>
                    ) : (
                      <p className="text-[11px] text-[#F5A623] mt-1.5">Assigned to {inc.assignedCallsign}</p>
                    )}
                  </div>
                );
              })}                                             
            </div>

            <h2 className="display-font text-xs uppercase tracking-wider text-[#7C8CA6] mt-4 mb-2 px-1">Recent Activity</h2>
            <div className="space-y-1.5 max-h-[180px] overflow-y-auto scrollbar-thin pr-1">
              {resolvedLog.length === 0 && <p className="text-xs text-[#64748B] px-1">Nothing resolved yet.</p>}
              {resolvedLog.map((r, idx) => (
                <div key={idx} className="flex items-center gap-2 text-[11px] text-[#7C8CA6] px-1">
                  <span className="w-1.5 h-1.5 rounded-full inline-block" style={{ background: PRIORITY_META[r.priority].color }} />
                  <span>{r.zoneLabel} — patient delivered</span>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}                                                                                                                                                                                                                                                                                                                                                                                                 
