import { useState, useMemo, useRef, useEffect, useCallback } from "react";
import {
  LineChart, Line, XAxis, YAxis, CartesianGrid,
  ReferenceLine, Tooltip, ResponsiveContainer, Area, AreaChart
} from "recharts";

// ═══════════════════════════════════════════════════════════
// MATH ENGINE — Black-Scholes completo
// ═══════════════════════════════════════════════════════════
const ncdf = x => {
  const s = x < 0 ? -1 : 1, a = Math.abs(x) / Math.SQRT2;
  const t = 1 / (1 + 0.3275911 * a);
  return 0.5 * (1 + s * (1 - ((((1.061405429 * t - 1.453152027) * t + 1.421413741) * t - 0.284496736) * t + 0.254829592) * t * Math.exp(-a * a)));
};
const npdf = x => Math.exp(-0.5 * x * x) / Math.sqrt(2 * Math.PI);

const bs = (S, K, T, r, σ, type) => {
  if (T < 5e-5) {
    const p = type === "call" ? Math.max(S - K, 0) : Math.max(K - S, 0);
    return { price: p, delta: type === "call" ? (S > K ? 1 : 0) : -(S < K ? 1 : 0), gamma: 0, theta: 0, vega: 0 };
  }
  const sq = Math.sqrt(T);
  const d1 = (Math.log(S / K) + (r + 0.5 * σ * σ) * T) / (σ * sq);
  const d2 = d1 - σ * sq;
  const nd1 = npdf(d1);
  const eRT = Math.exp(-r * T);
  return type === "call"
    ? { price: S * ncdf(d1) - K * eRT * ncdf(d2), delta: ncdf(d1),        gamma: nd1 / (S * σ * sq), theta: (-S * nd1 * σ / (2 * sq) - r * K * eRT * ncdf(d2)) / 365, vega: S * nd1 * sq / 100 }
    : { price: K * eRT * ncdf(-d2) - S * ncdf(-d1), delta: ncdf(d1) - 1,   gamma: nd1 / (S * σ * sq), theta: (-S * nd1 * σ / (2 * sq) + r * K * eRT * ncdf(-d2)) / 365, vega: S * nd1 * sq / 100 };
};

const genChain = (S, dte = 30, iv0 = 0.40) => {
  const T = dte / 365, r = 0.05;
  const step = S < 80 ? 2.5 : S < 250 ? 5 : S < 600 ? 10 : 25;
  return Array.from({ length: 17 }, (_, i) => {
    const K = Math.round((S - 8 * step + i * step) / step) * step;
    const mono = Math.abs(K - S) / S;
    const iv = iv0 * (1 + 1.5 * mono * mono);
    const sp = 0.025;
    const c = bs(S, K, T, r, iv, "call"), p = bs(S, K, T, r, iv, "put");
    return {
      strike: K, isATM: Math.abs(K - S) < step * 0.6,
      call: { ...c, bid: +(c.price * (1 - sp)).toFixed(2), ask: +(c.price * (1 + sp)).toFixed(2), iv, vol: Math.floor(Math.random() * 5000 + 100), oi: Math.floor(Math.random() * 28000 + 300) },
      put:  { ...p, bid: +(p.price * (1 - sp)).toFixed(2), ask: +(p.price * (1 + sp)).toFixed(2), iv, vol: Math.floor(Math.random() * 5000 + 100), oi: Math.floor(Math.random() * 28000 + 300) }
    };
  });
};

const calcPayoff = (legs, S) =>
  legs.reduce((tot, l) => {
    const intr = l.type === "call" ? Math.max(S - l.strike, 0) : Math.max(l.strike - S, 0);
    return tot + (l.dir === "buy" ? intr - l.prem : l.prem - intr) * l.qty * 100;
  }, 0);

// ═══════════════════════════════════════════════════════════
// DATA
// ═══════════════════════════════════════════════════════════
const TICKERS = {
  NVDA: { price: 875, iv: 0.45, name: "NVIDIA Corp" },
  SPY:  { price: 525, iv: 0.16, name: "SPDR S&P 500 ETF" },
  AAPL: { price: 196, iv: 0.24, name: "Apple Inc" },
  TSLA: { price: 185, iv: 0.58, name: "Tesla Inc" },
  META: { price: 520, iv: 0.33, name: "Meta Platforms" },
};

const STRATS = [
  { name: "Long Call",       emoji: "↗", bg: "#00e67614", border: "#00e67650", sentiment: "Bullish",    risk: "Defined",
    desc: "Compras el derecho a comprar 100 acciones al strike price. Ganancia potencialmente ilimitada si sube; pérdida máxima = prima pagada.",
    build: (atm) => [{ strike: atm, type: "call", dir: "buy", qty: 1 }] },
  { name: "Long Put",        emoji: "↘", bg: "#ff336614", border: "#ff336650", sentiment: "Bearish",    risk: "Defined",
    desc: "Compras el derecho a vender 100 acciones. Ganas si el precio cae. Pérdida máxima = prima pagada.",
    build: (atm, chain) => [{ strike: chain[chain.findIndex(c=>c.isATM)].strike, type: "put", dir: "buy", qty: 1 }] },
  { name: "Bull Call Spread", emoji: "⬆", bg: "#00e67614", border: "#00e67650", sentiment: "Mod. Bull", risk: "Defined",
    desc: "Compras un call + vendes un call a strike mayor. Reduce el costo pero también limita la ganancia máxima. Ideal para movimiento moderado.",
    build: (atm, chain) => { const i = chain.findIndex(c=>c.isATM); return [{ strike: chain[i]?.strike, type:"call", dir:"buy", qty:1 }, { strike: chain[i+2]?.strike, type:"call", dir:"sell", qty:1 }]; } },
  { name: "Bear Put Spread",  emoji: "⬇", bg: "#ff336614", border: "#ff336650", sentiment: "Mod. Bear", risk: "Defined",
    desc: "Compras un put ATM + vendes un put a strike menor. Exposición bajista con menor costo de prima.",
    build: (atm, chain) => { const i = chain.findIndex(c=>c.isATM); return [{ strike: chain[i]?.strike, type:"put", dir:"buy", qty:1 }, { strike: chain[i-2]?.strike, type:"put", dir:"sell", qty:1 }]; } },
  { name: "Long Straddle",    emoji: "↕", bg: "#ffd74014", border: "#ffd74050", sentiment: "Neutral+",  risk: "Defined",
    desc: "Compras ATM call + ATM put. Ganás si el precio se mueve mucho en cualquier dirección. Requiere un movimiento mayor que la prima combinada.",
    build: (atm, chain) => { const s = chain.find(c=>c.isATM)?.strike; return [{ strike:s, type:"call", dir:"buy", qty:1 }, { strike:s, type:"put", dir:"buy", qty:1 }]; } },
  { name: "Long Strangle",    emoji: "⟺", bg: "#ffd74014", border: "#ffd74050", sentiment: "Neutral+",  risk: "Defined",
    desc: "Compras OTM call + OTM put. Similar al straddle pero más barato y requiere mayor movimiento para ser rentable.",
    build: (atm, chain) => { const i = chain.findIndex(c=>c.isATM); return [{ strike:chain[i+2]?.strike, type:"call", dir:"buy", qty:1 }, { strike:chain[i-2]?.strike, type:"put", dir:"buy", qty:1 }]; } },
  { name: "Iron Condor",      emoji: "◆", bg: "#448aff14", border: "#448aff50", sentiment: "Neutral",   risk: "Defined",
    desc: "Vendes OTM strangle + compras protección más OTM. Cobras prima si el precio se queda en rango. Estrategia favorita en mercados de baja volatilidad.",
    build: (atm, chain) => { const i = chain.findIndex(c=>c.isATM); return [{ strike:chain[i-2]?.strike, type:"put", dir:"sell", qty:1 }, { strike:chain[i-4]?.strike, type:"put", dir:"buy", qty:1 }, { strike:chain[i+2]?.strike, type:"call", dir:"sell", qty:1 }, { strike:chain[i+4]?.strike, type:"call", dir:"buy", qty:1 }]; } },
  { name: "Covered Call",     emoji: "🛡", bg: "#448aff14", border: "#448aff50", sentiment: "Neutral+",  risk: "Stock",
    desc: "Vendes un OTM call contra acciones que ya posees. Cobras la prima como ingreso adicional, a cambio de limitar tu upside.",
    build: (atm, chain) => { const i = chain.findIndex(c=>c.isATM); return [{ strike:chain[i+2]?.strike, type:"call", dir:"sell", qty:1 }]; } },
];

// ═══════════════════════════════════════════════════════════
// STYLES (shared tokens)
// ═══════════════════════════════════════════════════════════
const C = {
  bg: "#07070f", surf: "#0d0d1c", surf2: "#111124",
  border: "#1a1a30", border2: "#252540",
  green: "#00e676", red: "#ff3366", yellow: "#ffd740",
  blue: "#448aff", purple: "#b070ff",
  text: "#ddddf8", muted: "#50507a", faint: "#2a2a48",
  font: "'IBM Plex Mono', monospace",
};
const globalStyle = `
  @import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@300;400;500;600&family=Syne:wght@600;700;800&display=swap');
  *{box-sizing:border-box;margin:0;padding:0}
  ::-webkit-scrollbar{width:3px;height:3px}
  ::-webkit-scrollbar-track{background:transparent}
  ::-webkit-scrollbar-thumb{background:${C.border2};border-radius:2px}
  .chain-row:hover{background:${C.faint}!important}
  .chain-row td.c-cell{cursor:pointer}
  .chain-row:hover td.c-cell{background:${C.green}12}
  .chain-row:hover td.p-cell{background:${C.red}12}
  .tab{background:none;border:none;font-family:${C.font};font-size:11px;letter-spacing:.1em;cursor:pointer;padding:10px 18px;border-bottom:2px solid transparent;color:${C.muted};transition:color .2s,border-color .2s;text-transform:uppercase}
  .tab:hover{color:${C.text}}
  .tab.on{color:${C.green};border-bottom-color:${C.green}}
  .strat{background:${C.surf};border:1px solid ${C.border};border-radius:8px;padding:16px;cursor:pointer;transition:all .2s}
  .strat:hover{border-color:${C.blue};transform:translateY(-1px)}
  input,select,textarea{background:${C.surf};border:1px solid ${C.border};color:${C.text};font-family:${C.font};font-size:12px;border-radius:4px;padding:6px 10px;outline:none}
  input:focus,select:focus,textarea:focus{border-color:${C.blue}}
  .btn-primary{background:${C.green};color:#000;border:none;font-family:${C.font};font-size:12px;font-weight:600;padding:9px 18px;border-radius:4px;cursor:pointer;letter-spacing:.05em;transition:opacity .15s}
  .btn-primary:hover{opacity:.85}
  .btn-primary:disabled{opacity:.4;cursor:not-allowed}
  .btn-ghost{background:none;border:1px solid ${C.border2};color:${C.muted};font-family:${C.font};font-size:11px;padding:5px 12px;border-radius:4px;cursor:pointer;transition:all .15s}
  .btn-ghost:hover{border-color:${C.text};color:${C.text}}
  .dir-btn{background:none;border:1px solid ${C.border2};font-family:${C.font};font-size:11px;padding:5px 14px;border-radius:3px;cursor:pointer;letter-spacing:.06em;text-transform:uppercase;transition:all .15s}
  .blink{animation:blink .9s step-end infinite}
  @keyframes blink{50%{opacity:0}}
  .pill{font-size:10px;padding:2px 8px;border-radius:10px;font-weight:500}
`;

// ═══════════════════════════════════════════════════════════
// CUSTOM TOOLTIP for recharts
// ═══════════════════════════════════════════════════════════
const PayoffTooltip = ({ active, payload, label }) => {
  if (!active || !payload?.length) return null;
  const v = payload[0].value;
  return (
    <div style={{ background: "#0d0d1c", border: `1px solid #1a1a30`, borderRadius: 4, padding: "8px 12px", fontSize: 11, fontFamily: C.font }}>
      <div style={{ color: C.muted }}>S = ${label}</div>
      <div style={{ color: v >= 0 ? C.green : C.red, fontWeight: 600 }}>{v >= 0 ? "+" : ""}${v}</div>
    </div>
  );
};

// ═══════════════════════════════════════════════════════════
// MAIN APP
// ═══════════════════════════════════════════════════════════
export default function OptionsApp() {
  const [ticker, setTicker]     = useState("NVDA");
  const [dte, setDte]           = useState(30);
  const [legs, setLegs]         = useState([]);
  const [tab, setTab]           = useState("chain");
  const [dir, setDir]           = useState("buy");
  const [qty, setQty]           = useState(1);
  const [msgs, setMsgs]         = useState([{
    role: "assistant",
    content: `¡Bienvenido a OPTIX! 👋\n\nSoy tu tutor de opciones financieras. Puedo explicarte cualquier estrategia, concepto o término.\n\n📐 Los Greeks (Delta, Gamma, Theta, Vega, Rho)\n📊 Estrategias (spreads, straddles, condors)\n🔍 Cómo leer una options chain\n💡 Risk/reward y cuando usar cada estrategia\n\n¿Por dónde empezamos?`
  }]);
  const [chatInput, setChatInput] = useState("");
  const [loading, setLoading]   = useState(false);
  const chatRef = useRef(null);

  const tk    = TICKERS[ticker];
  const chain = useMemo(() => genChain(tk.price, dte, tk.iv), [ticker, dte]);

  const resolvedLegs = useMemo(() => legs.map(leg => {
    const row = chain.find(r => r.strike === leg.strike);
    if (!row) return null;
    const opt  = leg.type === "call" ? row.call : row.put;
    const prem = leg.dir === "buy" ? opt.ask : opt.bid;
    return { ...leg, prem, delta: opt.delta, gamma: opt.gamma, theta: opt.theta, vega: opt.vega };
  }).filter(Boolean), [legs, chain]);

  const payoffData = useMemo(() => {
    if (!resolvedLegs.length) return [];
    const lo = tk.price * 0.68, hi = tk.price * 1.32;
    return Array.from({ length: 120 }, (_, i) => {
      const price = lo + i * (hi - lo) / 119;
      return { price: Math.round(price), pnl: Math.round(calcPayoff(resolvedLegs, price)) };
    });
  }, [resolvedLegs, ticker]);

  const greeks = useMemo(() => resolvedLegs.reduce((acc, l) => {
    const s = (l.dir === "buy" ? 1 : -1) * l.qty * 100;
    return { delta: acc.delta + l.delta * s, gamma: acc.gamma + l.gamma * s, theta: acc.theta + l.theta * s, vega: acc.vega + l.vega * s };
  }, { delta: 0, gamma: 0, theta: 0, vega: 0 }), [resolvedLegs]);

  const netCost = useMemo(() => resolvedLegs.reduce((s, l) =>
    s + (l.dir === "buy" ? l.prem : -l.prem) * l.qty * 100, 0), [resolvedLegs]);

  const maxPnl = payoffData.length ? Math.max(...payoffData.map(d => d.pnl)) : 0;
  const minPnl = payoffData.length ? Math.min(...payoffData.map(d => d.pnl)) : 0;

  const addLeg = useCallback((strike, type) => {
    setLegs(p => [...p, { strike, type, dir, qty }]);
  }, [dir, qty]);

  const removeLeg = (i) => setLegs(p => p.filter((_, j) => j !== i));

  const applyStrat = (s) => {
    const atmStrike = chain.find(r => r.isATM)?.strike || tk.price;
    const newLegs = s.build(atmStrike, chain).filter(l => l.strike);
    setLegs(newLegs);
    setTab("chain");
  };

  const sendMsg = async (msg) => {
    if (!msg.trim() || loading) return;
    const posCtx = resolvedLegs.length
      ? `\n\nPosición activa: ${resolvedLegs.map(l => `${l.dir.toUpperCase()} ${l.qty}x ${ticker} ${l.strike}${l.type[0].toUpperCase()} @$${l.prem?.toFixed(2)}`).join(" | ")}\nGreeks totales: Δ=${greeks.delta.toFixed(2)} Γ=${greeks.gamma.toFixed(4)} Θ=${greeks.theta.toFixed(2)}/día ν=${greeks.vega.toFixed(2)}\nCosto neto: ${netCost >= 0 ? "DEBIT" : "CREDIT"} $${Math.abs(netCost).toFixed(0)}`
      : "";
    const newMsgs = [...msgs, { role: "user", content: msg }];
    setMsgs(newMsgs);
    setChatInput("");
    setLoading(true);
    try {
      const res = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: `Eres un experto en opciones financieras (equity options). Tu rol es enseñar y guiar a traders. Responde en el idioma del usuario. Sé preciso, usa números reales y ejemplos concretos. Explica riesgos honestamente.

Contexto del mercado simulado:
- Ticker: ${ticker} (${tk.name}) @ $${tk.price}
- IV base: ${(tk.iv * 100).toFixed(0)}%
- DTE seleccionado: ${dte} días${posCtx}`,
          messages: newMsgs
        })
      });
      const data = await res.json();
      const reply = data.content?.[0]?.text || "Error al conectar. Intenta de nuevo.";
      setMsgs(p => [...p, { role: "assistant", content: reply }]);
    } catch {
      setMsgs(p => [...p, { role: "assistant", content: "Error de conexión. Intenta de nuevo." }]);
    }
    setLoading(false);
  };

  useEffect(() => {
    if (chatRef.current) chatRef.current.scrollTop = chatRef.current.scrollHeight;
  }, [msgs, loading]);

  // ─────────────────────────── RENDER ────────────────────────────
  return (
    <div style={{ fontFamily: C.font, background: C.bg, color: C.text, height: "100vh", display: "flex", flexDirection: "column", overflow: "hidden" }}>
      <style>{globalStyle}</style>

      {/* ── HEADER ── */}
      <div style={{ background: "#0a0a14", borderBottom: `1px solid ${C.border}`, padding: "10px 20px", display: "flex", alignItems: "center", gap: 20, flexShrink: 0 }}>
        <span style={{ fontFamily: "Syne, sans-serif", fontWeight: 800, fontSize: 18, color: C.green, letterSpacing: "-.02em" }}>◈ OPTIX</span>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <select value={ticker} onChange={e => { setTicker(e.target.value); setLegs([]); }} style={{ width: 90 }}>
            {Object.keys(TICKERS).map(t => <option key={t}>{t}</option>)}
          </select>
          <span style={{ color: C.yellow, fontWeight: 600, fontSize: 15 }}>${tk.price}</span>
          <span style={{ color: C.muted, fontSize: 10 }}>{tk.name}</span>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
          <span style={{ color: C.muted, fontSize: 10, marginRight: 2 }}>EXP:</span>
          {[7, 14, 30, 45, 60, 90].map(d => (
            <button key={d} onClick={() => setDte(d)} style={{ background: dte === d ? C.blue + "22" : "none", border: `1px solid ${dte === d ? C.blue : C.border}`, color: dte === d ? C.blue : C.muted, fontFamily: C.font, fontSize: 10, padding: "3px 9px", borderRadius: 3, cursor: "pointer", transition: "all .15s" }}>{d}D</button>
          ))}
        </div>
        <div style={{ marginLeft: "auto", display: "flex", gap: 20, fontSize: 11 }}>
          <span style={{ color: C.muted }}>IV <b style={{ color: C.yellow }}>{(tk.iv * 100).toFixed(0)}%</b></span>
          <span style={{ color: C.muted }}>DTE <b style={{ color: C.text }}>{dte}</b></span>
          {resolvedLegs.length > 0 && <span style={{ color: netCost >= 0 ? C.red : C.green, fontWeight: 600 }}>{netCost >= 0 ? "DEBIT" : "CREDIT"} ${Math.abs(netCost).toFixed(0)}</span>}
        </div>
      </div>

      {/* ── TABS ── */}
      <div style={{ background: "#09091a", borderBottom: `1px solid ${C.border}`, display: "flex", flexShrink: 0, paddingLeft: 8 }}>
        {[["chain", "⛓ Chain"], ["strategies", "⚡ Strategies"], ["tutor", "🎓 AI Tutor"]].map(([id, label]) => (
          <button key={id} className={`tab${tab === id ? " on" : ""}`} onClick={() => setTab(id)}>{label}</button>
        ))}
      </div>

      {/* ══════════════════════════════════════════════════
          TAB: CHAIN
      ══════════════════════════════════════════════════ */}
      {tab === "chain" && (
        <div style={{ flex: 1, display: "flex", overflow: "hidden" }}>

          {/* Chain table */}
          <div style={{ flex: "0 0 62%", borderRight: `1px solid ${C.border}`, overflow: "auto", display: "flex", flexDirection: "column" }}>
            {/* Controls */}
            <div style={{ padding: "9px 16px", borderBottom: `1px solid ${C.border}`, display: "flex", gap: 10, alignItems: "center", position: "sticky", top: 0, background: C.bg, zIndex: 10, flexShrink: 0 }}>
              <span style={{ fontSize: 11, color: C.muted }}>ADD LEG:</span>
              {["buy", "sell"].map(d => (
                <button key={d} className="dir-btn" onClick={() => setDir(d)} style={{ borderColor: dir === d ? (d === "buy" ? C.green : C.red) : C.border2, color: dir === d ? (d === "buy" ? C.green : C.red) : C.muted, background: dir === d ? (d === "buy" ? C.green + "15" : C.red + "15") : "none" }}>{d}</button>
              ))}
              <input type="number" value={qty} min={1} max={50} onChange={e => setQty(Math.max(1, +e.target.value || 1))} style={{ width: 55, textAlign: "center" }} />
              <span style={{ fontSize: 11, color: C.muted }}>contracts — click BID/ASK para agregar</span>
            </div>

            {/* Table */}
            <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 11 }}>
              <thead style={{ position: "sticky", top: 47, background: "#09091a", zIndex: 9 }}>
                <tr>
                  <th colSpan={5} style={{ color: C.green, padding: "7px 10px", textAlign: "right", fontSize: 10, letterSpacing: ".08em", borderBottom: `1px solid ${C.border}` }}>CALLS</th>
                  <th style={{ color: C.yellow, padding: "7px 10px", textAlign: "center", fontSize: 10, letterSpacing: ".08em", borderBottom: `1px solid ${C.border}`, borderLeft: `1px solid ${C.border}`, borderRight: `1px solid ${C.border}`, minWidth: 70 }}>STRIKE</th>
                  <th colSpan={5} style={{ color: C.red, padding: "7px 10px", textAlign: "left", fontSize: 10, letterSpacing: ".08em", borderBottom: `1px solid ${C.border}` }}>PUTS</th>
                </tr>
                <tr style={{ background: "#0a0a16" }}>
                  {["OI", "Δ", "IV%", "BID", "ASK", "", "BID", "ASK", "IV%", "Δ", "OI"].map((h, i) => (
                    <th key={i} style={{ padding: "5px 8px", color: C.muted, fontWeight: 400, textAlign: i < 5 ? "right" : i === 5 ? "center" : "left", borderBottom: `1px solid ${C.border}`, borderLeft: i === 5 || i === 6 ? `1px solid ${C.border}` : "none", borderRight: i === 5 ? `1px solid ${C.border}` : "none" }}>{h}</th>
                  ))}
                </tr>
              </thead>
              <tbody>
                {chain.map(row => {
                  const atm = row.isATM;
                  return (
                    <tr key={row.strike} className="chain-row" style={{ background: atm ? "#111130" : "transparent", borderBottom: `1px solid ${C.border}22` }}>
                      <td style={{ padding: "6px 8px", textAlign: "right", color: C.muted }}>{(row.call.oi / 1000).toFixed(1)}k</td>
                      <td style={{ padding: "6px 8px", textAlign: "right", color: C.green + "90" }}>{row.call.delta.toFixed(2)}</td>
                      <td style={{ padding: "6px 8px", textAlign: "right", color: C.muted }}>{(row.call.iv * 100).toFixed(0)}%</td>
                      <td className="c-cell" onClick={() => addLeg(row.strike, "call")} style={{ padding: "6px 8px", textAlign: "right", color: C.green + "70", transition: "background .1s" }}>{row.call.bid.toFixed(2)}</td>
                      <td className="c-cell" onClick={() => addLeg(row.strike, "call")} style={{ padding: "6px 8px", textAlign: "right", color: C.green, fontWeight: 500, transition: "background .1s" }}>{row.call.ask.toFixed(2)}</td>

                      <td style={{ padding: "6px 10px", textAlign: "center", color: atm ? C.yellow : C.text, fontWeight: atm ? 700 : 400, borderLeft: `1px solid ${C.border}`, borderRight: `1px solid ${C.border}`, background: atm ? "#1a1a38" : "transparent", fontSize: atm ? 12 : 11 }}>
                        {atm && <span style={{ color: C.yellow, marginRight: 4, fontSize: 9 }}>◈</span>}{row.strike}
                      </td>

                      <td className="p-cell" onClick={() => addLeg(row.strike, "put")} style={{ padding: "6px 8px", textAlign: "left", color: C.red, fontWeight: 500, transition: "background .1s", cursor: "pointer" }}>{row.put.bid.toFixed(2)}</td>
                      <td className="p-cell" onClick={() => addLeg(row.strike, "put")} style={{ padding: "6px 8px", textAlign: "left", color: C.red + "70", transition: "background .1s", cursor: "pointer" }}>{row.put.ask.toFixed(2)}</td>
                      <td style={{ padding: "6px 8px", textAlign: "left", color: C.muted }}>{(row.put.iv * 100).toFixed(0)}%</td>
                      <td style={{ padding: "6px 8px", textAlign: "left", color: C.red + "90" }}>{row.put.delta.toFixed(2)}</td>
                      <td style={{ padding: "6px 8px", textAlign: "left", color: C.muted }}>{(row.put.oi / 1000).toFixed(1)}k</td>
                    </tr>
                  );
                })}
              </tbody>
            </table>
          </div>

          {/* Right panel */}
          <div style={{ flex: 1, display: "flex", flexDirection: "column", overflow: "hidden" }}>
            {/* Position builder */}
            <div style={{ borderBottom: `1px solid ${C.border}`, padding: "12px 16px", flexShrink: 0, minHeight: 140 }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
                <span style={{ fontSize: 10, color: C.muted, letterSpacing: ".1em" }}>POSITION BUILDER</span>
                {legs.length > 0 && <button className="btn-ghost" onClick={() => setLegs([])}>CLEAR ALL</button>}
              </div>
              {resolvedLegs.length === 0 ? (
                <div style={{ color: C.muted, fontSize: 11, textAlign: "center", padding: "18px 0" }}>Selecciona una estrategia o clickea en la chain →</div>
              ) : (
                <div style={{ display: "flex", flexDirection: "column", gap: 5 }}>
                  {resolvedLegs.map((l, i) => (
                    <div key={i} style={{ display: "flex", alignItems: "center", gap: 8, fontSize: 11, padding: "6px 10px", background: C.surf, borderRadius: 4, border: `1px solid ${C.border}` }}>
                      <span style={{ color: l.dir === "buy" ? C.green : C.red, fontWeight: 600, minWidth: 36 }}>{l.dir.toUpperCase()}</span>
                      <span>{l.qty}×</span>
                      <span style={{ color: C.yellow }}>{l.strike}</span>
                      <span style={{ color: l.type === "call" ? C.green : C.red }}>{l.type.toUpperCase()}</span>
                      <span style={{ color: C.muted, flex: 1 }}>@ ${l.prem?.toFixed(2)}</span>
                      <span style={{ color: C.muted, fontSize: 10 }}>Δ{l.delta?.toFixed(2)}</span>
                      <button onClick={() => removeLeg(i)} style={{ background: "none", border: "none", color: C.muted, cursor: "pointer", fontSize: 16, lineHeight: 1, padding: "0 2px" }}>×</button>
                    </div>
                  ))}
                  <div style={{ display: "flex", gap: 16, fontSize: 10, padding: "4px 10px", color: C.muted }}>
                    <span>NET: <b style={{ color: netCost >= 0 ? C.red : C.green }}>{netCost >= 0 ? "+" : ""}{netCost < 0 ? "-" : ""}${Math.abs(netCost).toFixed(0)}</b></span>
                    <span>Δ <b style={{ color: C.text }}>{greeks.delta.toFixed(2)}</b></span>
                    <span>Θ <b style={{ color: greeks.theta < 0 ? C.red : C.green }}>{greeks.theta.toFixed(2)}/d</b></span>
                    <span>ν <b style={{ color: C.text }}>{greeks.vega.toFixed(2)}</b></span>
                    <span>Γ <b style={{ color: C.text }}>{greeks.gamma.toFixed(4)}</b></span>
                  </div>
                </div>
              )}
            </div>

            {/* Payoff diagram */}
            <div style={{ flex: 1, padding: "12px 16px 8px", display: "flex", flexDirection: "column", overflow: "hidden" }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10, flexShrink: 0 }}>
                <span style={{ fontSize: 10, color: C.muted, letterSpacing: ".1em" }}>P&L AT EXPIRATION</span>
                {payoffData.length > 0 && (
                  <div style={{ display: "flex", gap: 12, fontSize: 11 }}>
                    <span style={{ color: C.green }}>MAX +${maxPnl.toLocaleString()}</span>
                    <span style={{ color: C.red }}>MAX -${Math.abs(minPnl).toLocaleString()}</span>
                  </div>
                )}
              </div>
              {payoffData.length === 0 ? (
                <div style={{ flex: 1, display: "flex", alignItems: "center", justifyContent: "center", color: C.muted, fontSize: 11 }}>Agrega legs para ver el diagrama</div>
              ) : (
                <div style={{ flex: 1 }}>
                  <ResponsiveContainer width="100%" height="100%">
                    <AreaChart data={payoffData} margin={{ top: 5, right: 12, bottom: 5, left: 12 }}>
                      <defs>
                        <linearGradient id="pnlUp" x1="0" y1="0" x2="0" y2="1">
                          <stop offset="5%" stopColor={C.green} stopOpacity={0.15} />
                          <stop offset="95%" stopColor={C.green} stopOpacity={0} />
                        </linearGradient>
                        <linearGradient id="pnlDown" x1="0" y1="0" x2="0" y2="1">
                          <stop offset="5%" stopColor={C.red} stopOpacity={0} />
                          <stop offset="95%" stopColor={C.red} stopOpacity={0.15} />
                        </linearGradient>
                      </defs>
                      <CartesianGrid strokeDasharray="2 4" stroke={C.border} />
                      <XAxis dataKey="price" stroke={C.muted} tick={{ fontSize: 10, fill: C.muted }} tickFormatter={v => `$${v}`} />
                      <YAxis stroke={C.muted} tick={{ fontSize: 10, fill: C.muted }} tickFormatter={v => `${v >= 0 ? "+" : ""}$${v}`} width={65} />
                      <ReferenceLine y={0} stroke={C.muted} strokeDasharray="3 3" strokeWidth={1} />
                      <ReferenceLine x={tk.price} stroke={C.yellow} strokeDasharray="4 2" strokeWidth={1.5} label={{ value: "NOW", fill: C.yellow, fontSize: 9, position: "insideTopRight" }} />
                      <Tooltip content={<PayoffTooltip />} />
                      <Area type="monotone" dataKey="pnl" stroke={C.blue} strokeWidth={2} fill="url(#pnlUp)" dot={false} />
                    </AreaChart>
                  </ResponsiveContainer>
                </div>
              )}
            </div>
          </div>
        </div>
      )}

      {/* ══════════════════════════════════════════════════
          TAB: STRATEGIES
      ══════════════════════════════════════════════════ */}
      {tab === "strategies" && (
        <div style={{ flex: 1, overflow: "auto", padding: 20 }}>
          <div style={{ fontFamily: "Syne, sans-serif", fontWeight: 700, fontSize: 15, marginBottom: 4 }}>Estrategias de Opciones</div>
          <div style={{ fontSize: 11, color: C.muted, marginBottom: 20 }}>Click para cargar la estrategia en el builder. Los strikes se calculan automáticamente con {ticker} @ ${tk.price}.</div>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(300px, 1fr))", gap: 12 }}>
            {STRATS.map(s => (
              <div key={s.name} className="strat" onClick={() => applyStrat(s)} style={{ borderColor: s.border + "80" }}>
                <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", marginBottom: 8 }}>
                  <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                    <span style={{ fontSize: 18, background: s.bg, border: `1px solid ${s.border}`, borderRadius: 6, padding: "4px 8px", color: C.text }}>{s.emoji}</span>
                    <span style={{ fontFamily: "Syne, sans-serif", fontWeight: 700, fontSize: 13 }}>{s.name}</span>
                  </div>
                  <span className="pill" style={{ background: s.bg, border: `1px solid ${s.border}`, color: C.text }}>{s.sentiment}</span>
                </div>
                <div style={{ fontSize: 11, color: C.muted, lineHeight: 1.6, marginBottom: 10 }}>{s.desc}</div>
                <div style={{ display: "flex", justifyContent: "space-between", fontSize: 10, color: C.muted }}>
                  <span>Risk: <span style={{ color: C.text }}>{s.risk}</span></span>
                  <span style={{ color: C.blue }}>→ Cargar en builder</span>
                </div>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* ══════════════════════════════════════════════════
          TAB: AI TUTOR
      ══════════════════════════════════════════════════ */}
      {tab === "tutor" && (
        <div style={{ flex: 1, display: "flex", flexDirection: "column", overflow: "hidden" }}>
          {/* Chat history */}
          <div ref={chatRef} style={{ flex: 1, overflow: "auto", padding: "20px", display: "flex", flexDirection: "column", gap: 14 }}>
            {msgs.map((m, i) => (
              <div key={i} style={{ display: "flex", gap: 10, flexDirection: m.role === "user" ? "row-reverse" : "row", alignItems: "flex-start" }}>
                <div style={{ width: 28, height: 28, borderRadius: "50%", flexShrink: 0, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 11, background: m.role === "user" ? C.blue + "25" : C.green + "25", border: `1px solid ${m.role === "user" ? C.blue : C.green}`, color: m.role === "user" ? C.blue : C.green, fontWeight: 600 }}>
                  {m.role === "user" ? "U" : "◈"}
                </div>
                <div style={{ maxWidth: "76%", background: m.role === "user" ? "#111134" : C.surf, border: `1px solid ${C.border}`, borderRadius: 8, padding: "12px 14px", fontSize: 12, lineHeight: 1.7, color: C.text, whiteSpace: "pre-wrap", wordBreak: "break-word" }}>
                  {m.content}
                </div>
              </div>
            ))}
            {loading && (
              <div style={{ display: "flex", gap: 10, alignItems: "flex-start" }}>
                <div style={{ width: 28, height: 28, borderRadius: "50%", flexShrink: 0, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 11, background: C.green + "25", border: `1px solid ${C.green}`, color: C.green, fontWeight: 600 }}>◈</div>
                <div style={{ background: C.surf, border: `1px solid ${C.border}`, borderRadius: 8, padding: "12px 14px", fontSize: 12, color: C.muted }}>
                  <span className="blink">▌</span>
                </div>
              </div>
            )}
          </div>

          {/* Quick questions */}
          <div style={{ padding: "10px 20px 0", display: "flex", gap: 6, flexWrap: "wrap", borderTop: `1px solid ${C.border}` }}>
            {["¿Qué es el Theta decay?", "Explica el Iron Condor", "¿Cuándo usar un straddle?", "¿Cómo afecta la IV?", "Analiza mi posición"].map(q => (
              <button key={q} className="btn-ghost" style={{ fontSize: 10 }} onClick={() => sendMsg(q)}>{q}</button>
            ))}
          </div>

          {/* Input */}
          <div style={{ padding: "10px 20px 16px", display: "flex", gap: 10 }}>
            <input
              value={chatInput}
              onChange={e => setChatInput(e.target.value)}
              onKeyDown={e => e.key === "Enter" && !e.shiftKey && sendMsg(chatInput)}
              placeholder="Pregunta sobre opciones, griegas, estrategias..."
              style={{ flex: 1, padding: "10px 14px" }}
            />
            <button className="btn-primary" onClick={() => sendMsg(chatInput)} disabled={loading || !chatInput.trim()}>SEND</button>
          </div>
        </div>
      )}
    </div>
  );
}# vite-react

[Edit on StackBlitz ⚡️](https://stackblitz.com/edit/vite-react)