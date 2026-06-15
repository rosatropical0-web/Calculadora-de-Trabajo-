import { useState, useEffect, useRef } from "react";

const STORAGE_KEY = "trabajos-data-v2";

const getSemanaKey = (date = new Date()) => {
  const d = new Date(date);
  d.setHours(0, 0, 0, 0);
  d.setDate(d.getDate() + 4 - (d.getDay() || 7));
  const yearStart = new Date(d.getFullYear(), 0, 1);
  const weekNo = Math.ceil((((d - yearStart) / 86400000) + 1) / 7);
  const year = new Date(date).getFullYear();
  return `${year}-S${String(weekNo).padStart(2, "0")}`;
};

const getSemanaActual = () => getSemanaKey(new Date());

const getRangoSemana = (semanaKey) => {
  const [year, semStr] = semanaKey.split("-S");
  const weekNo = parseInt(semStr);
  const jan1 = new Date(parseInt(year), 0, 1);
  const dayOfWeek = jan1.getDay() || 7;
  const daysToMonday = dayOfWeek <= 4 ? 1 - dayOfWeek : 8 - dayOfWeek;
  const firstMonday = new Date(jan1);
  firstMonday.setDate(jan1.getDate() + daysToMonday);
  const monday = new Date(firstMonday);
  monday.setDate(firstMonday.getDate() + (weekNo - 1) * 7);
  const sunday = new Date(monday);
  sunday.setDate(monday.getDate() + 6);
  const fmtD = (d) => d.toLocaleDateString("es-ES", { day: "numeric", month: "short" });
  return { monday, sunday, label: `${fmtD(monday)} – ${fmtD(sunday)}` };
};

const getNombreSemana = (semanaKey) => {
  const { label } = getRangoSemana(semanaKey);
  const [year, semStr] = semanaKey.split("-S");
  return `Semana ${parseInt(semStr)} · ${label} ${year}`;
};

const getSemanasDeMes = (allData, mesKey) => {
  const [year, month] = mesKey.split("-");
  return Object.keys(allData)
    .filter(k => {
      const { monday } = getRangoSemana(k);
      return monday.getFullYear() === parseInt(year) && monday.getMonth() + 1 === parseInt(month);
    })
    .sort();
};

const getMesesDisponibles = (allData) => {
  const meses = new Set();
  Object.keys(allData).forEach(semKey => {
    if ((allData[semKey] || []).length > 0) {
      const { monday } = getRangoSemana(semKey);
      meses.add(`${monday.getFullYear()}-${String(monday.getMonth() + 1).padStart(2, "0")}`);
    }
  });
  return [...meses].sort().reverse();
};

const getMesActual = () => {
  const now = new Date();
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}`;
};

const nombreMes = (mesKey) => {
  const [year, month] = mesKey.split("-");
  return new Date(parseInt(year), parseInt(month) - 1, 1)
    .toLocaleDateString("es-ES", { month: "long", year: "numeric" });
};

const fmt = (n) =>
  new Intl.NumberFormat("es-AR", { style: "currency", currency: "ARS", minimumFractionDigits: 2 }).format(n);

export default function CalculadoraTrabajo() {
  const [data, setData] = useState({});
  const semanaActual = getSemanaActual();
  const mesActual = getMesActual();

  const [descripcion, setDescripcion] = useState("");
  const [precio, setPrecio] = useState("");
  const [fechaTrabajo, setFechaTrabajo] = useState(() => new Date().toISOString().split("T")[0]);
  const [porcentaje, setPorcentaje] = useState(35);
  const [editandoPct, setEditandoPct] = useState(false);
  const [pctTemp, setPctTemp] = useState("35");
  const [mesViendo, setMesViendo] = useState(getMesActual());
  const [semanaViendo, setSemanaViendo] = useState(getSemanaActual());
  const [animando, setAnimando] = useState(false);
  const [vista, setVista] = useState("semana");
  const [modalBackup, setModalBackup] = useState(false);
  const [importMsg, setImportMsg] = useState(null); // {tipo: "ok"|"error", texto}
  const fileInputRef = useRef();

  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) setData(JSON.parse(saved));
    } catch {}
  }, []);

  const guardar = (newData) => {
    setData(newData);
    try { localStorage.setItem(STORAGE_KEY, JSON.stringify(newData)); } catch {}
  };

  const trabajosDeSemana = (sem) => data[sem] || [];

  const agregarTrabajo = () => {
    const p = parseFloat(precio.replace(",", "."));
    if (!descripcion.trim() || isNaN(p) || p <= 0) return;
    const ganancia = parseFloat((p * porcentaje / 100).toFixed(2));
    const fechaObj = new Date(fechaTrabajo + "T12:00:00");
    const semanaDelTrabajo = getSemanaKey(fechaObj);
    const nuevo = {
      id: Date.now(),
      descripcion: descripcion.trim(),
      precio: p,
      ganancia,
      fecha: fechaObj.toLocaleDateString("es-ES", { weekday: "short", day: "numeric", month: "short" }),
      fechaISO: fechaTrabajo,
    };
    const newData = {
      ...data,
      [semanaDelTrabajo]: [...trabajosDeSemana(semanaDelTrabajo), nuevo].sort((a, b) => a.fechaISO.localeCompare(b.fechaISO)),
    };
    guardar(newData);
    setDescripcion("");
    setPrecio("");
    setSemanaViendo(semanaDelTrabajo);
    const mesDelTrabajo = `${fechaObj.getFullYear()}-${String(fechaObj.getMonth() + 1).padStart(2, "0")}`;
    setMesViendo(mesDelTrabajo);
    setAnimando(true);
    setTimeout(() => setAnimando(false), 600);
  };

  const eliminar = (sem, id) => {
    const newData = { ...data, [sem]: trabajosDeSemana(sem).filter(t => t.id !== id) };
    guardar(newData);
  };

  const totalPrecioSem = (sem) => trabajosDeSemana(sem).reduce((s, t) => s + t.precio, 0);
  const totalGananciaSem = (sem) => trabajosDeSemana(sem).reduce((s, t) => s + t.ganancia, 0);

  const totalPrecioMes = (mes) => getSemanasDeMes(data, mes).reduce((s, sem) => s + totalPrecioSem(sem), 0);
  const totalGananciaMes = (mes) => getSemanasDeMes(data, mes).reduce((s, sem) => s + totalGananciaSem(sem), 0);
  const totalTrabajosMes = (mes) => getSemanasDeMes(data, mes).reduce((s, sem) => s + trabajosDeSemana(sem).length, 0);

  const totalTrabajos = Object.values(data).reduce((s, arr) => s + arr.length, 0);

  // ---- EXPORTAR ----
  const exportar = () => {
    const backup = {
      version: 1,
      exportadoEl: new Date().toISOString(),
      porcentaje,
      data,
    };
    const blob = new Blob([JSON.stringify(backup, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    const fecha = new Date().toLocaleDateString("es-ES").replace(/\//g, "-");
    a.href = url;
    a.download = `respaldo-trabajos-${fecha}.json`;
    a.click();
    URL.revokeObjectURL(url);
  };

  // ---- IMPORTAR ----
  const importar = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      try {
        const parsed = JSON.parse(ev.target.result);
        if (!parsed.data || typeof parsed.data !== "object") throw new Error("Formato inválido");
        // Merge: combinar datos existentes con los importados
        const merged = { ...data };
        Object.keys(parsed.data).forEach(sem => {
          if (!merged[sem]) {
            merged[sem] = parsed.data[sem];
          } else {
            // agregar solo los que no existen por id
            const idsExistentes = new Set(merged[sem].map(t => t.id));
            const nuevos = parsed.data[sem].filter(t => !idsExistentes.has(t.id));
            merged[sem] = [...merged[sem], ...nuevos].sort((a, b) => a.fechaISO.localeCompare(b.fechaISO));
          }
        });
        if (parsed.porcentaje) setPorcentaje(parsed.porcentaje);
        guardar(merged);
        setImportMsg({ tipo: "ok", texto: "✅ Datos restaurados correctamente" });
      } catch {
        setImportMsg({ tipo: "error", texto: "❌ El archivo no es válido" });
      }
      setTimeout(() => setImportMsg(null), 4000);
    };
    reader.readAsText(file);
    e.target.value = "";
  };

  const mesesDisponibles = [...new Set([mesActual, ...getMesesDisponibles(data)])];
  const semanasDeMesViendo = [...new Set([...(mesViendo === mesActual ? [semanaActual] : []), ...getSemanasDeMes(data, mesViendo)])].sort().reverse();
  const precioNum = parseFloat(precio.replace(",", "."));

  return (
    <div style={{ minHeight: "100vh", background: "linear-gradient(135deg, #0f0c1a 0%, #1a1030 50%, #0f1a2e 100%)", fontFamily: "'Georgia', 'Times New Roman', serif", padding: "0 0 80px", color: "#e8d5b7" }}>

      {/* Header */}
      <div style={{ background: "linear-gradient(180deg, rgba(255,200,80,0.12) 0%, transparent 100%)", borderBottom: "1px solid rgba(255,200,80,0.2)", padding: "28px 24px 20px", textAlign: "center", position: "relative" }}>
        <div style={{ fontSize: 11, letterSpacing: 6, color: "#f0b840", textTransform: "uppercase", marginBottom: 6 }}>Registro de Ingresos</div>
        <h1 style={{ margin: 0, fontSize: "clamp(22px, 5vw, 36px)", fontWeight: 700, color: "#fff", textShadow: "0 0 40px rgba(255,200,80,0.4)", letterSpacing: -1 }}>
          Calculadora de Trabajo
        </h1>
        <div style={{ marginTop: 10, display: "flex", justifyContent: "center", alignItems: "center", gap: 8, flexWrap: "wrap" }}>
          <span style={{ color: "#a89878", fontSize: 13 }}>Ganancia:</span>
          {editandoPct ? (
            <span style={{ display: "flex", gap: 6, alignItems: "center" }}>
              <input type="number" value={pctTemp} onChange={e => setPctTemp(e.target.value)}
                style={{ width: 55, background: "rgba(255,200,80,0.1)", border: "1px solid #f0b840", borderRadius: 6, color: "#f0b840", fontSize: 15, textAlign: "center", padding: "2px 4px", outline: "none" }} />
              <span style={{ color: "#f0b840" }}>%</span>
              <button onClick={() => { const v = parseFloat(pctTemp); if (!isNaN(v) && v > 0 && v <= 100) setPorcentaje(v); setEditandoPct(false); }}
                style={{ background: "#f0b840", color: "#0f0c1a", border: "none", borderRadius: 6, padding: "3px 10px", cursor: "pointer", fontWeight: 700, fontSize: 13 }}>OK</button>
            </span>
          ) : (
            <button onClick={() => { setPctTemp(String(porcentaje)); setEditandoPct(true); }}
              style={{ background: "rgba(255,200,80,0.15)", border: "1px solid rgba(255,200,80,0.4)", borderRadius: 20, color: "#f0b840", fontSize: 15, fontWeight: 700, padding: "2px 12px", cursor: "pointer" }}>
              {porcentaje}% ✎
            </button>
          )}
        </div>
        {/* Botón backup en esquina */}
        <button onClick={() => setModalBackup(true)} style={{ position: "absolute", top: 16, right: 16, background: "rgba(255,200,80,0.1)", border: "1px solid rgba(255,200,80,0.25)", borderRadius: 10, color: "#f0b840", fontSize: 18, width: 38, height: 38, cursor: "pointer", display: "flex", alignItems: "center", justifyContent: "center" }} title="Respaldo de datos">
          🗂️
        </button>
      </div>

      {/* Formulario */}
      <div style={{ maxWidth: 560, margin: "24px auto 0", padding: "0 16px" }}>
        <div style={{ background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,200,80,0.18)", borderRadius: 16, padding: "20px 18px" }}>
          <div style={{ fontSize: 10, letterSpacing: 4, color: "#f0b840", textTransform: "uppercase", marginBottom: 14 }}>Nuevo trabajo</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            <input placeholder="Descripción del trabajo..." value={descripcion} onChange={e => setDescripcion(e.target.value)} onKeyDown={e => e.key === "Enter" && agregarTrabajo()}
              style={{ background: "rgba(255,255,255,0.06)", border: "1px solid rgba(255,255,255,0.12)", borderRadius: 10, color: "#fff", fontSize: 15, padding: "11px 13px", outline: "none", fontFamily: "inherit" }} />
            <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
              <label style={{ color: "#a89878", fontSize: 13, whiteSpace: "nowrap" }}>📅 Fecha:</label>
              <input type="date" value={fechaTrabajo} onChange={e => setFechaTrabajo(e.target.value)}
                style={{ flex: 1, background: "rgba(255,255,255,0.06)", border: "1px solid rgba(255,255,255,0.12)", borderRadius: 10, color: "#e8d5b7", fontSize: 14, padding: "9px 12px", outline: "none", fontFamily: "inherit", colorScheme: "dark" }} />
            </div>
            <div style={{ display: "flex", gap: 10 }}>
              <div style={{ flex: 1, position: "relative" }}>
                <span style={{ position: "absolute", left: 11, top: "50%", transform: "translateY(-50%)", color: "#a89878", fontSize: 14 }}>$</span>
                <input placeholder="0.00" value={precio} onChange={e => setPrecio(e.target.value)} onKeyDown={e => e.key === "Enter" && agregarTrabajo()} type="text" inputMode="decimal"
                  style={{ width: "100%", background: "rgba(255,255,255,0.06)", border: "1px solid rgba(255,255,255,0.12)", borderRadius: 10, color: "#fff", fontSize: 15, padding: "11px 13px 11px 26px", outline: "none", fontFamily: "inherit", boxSizing: "border-box" }} />
              </div>
              {precio && !isNaN(precioNum) && precioNum > 0 && (
                <div style={{ display: "flex", alignItems: "center", background: "rgba(255,200,80,0.08)", border: "1px solid rgba(255,200,80,0.2)", borderRadius: 10, padding: "0 12px", color: "#f0b840", fontSize: 13, whiteSpace: "nowrap" }}>
                  → {fmt(precioNum * porcentaje / 100)}
                </div>
              )}
            </div>
            <button onClick={agregarTrabajo}
              style={{ background: "linear-gradient(135deg, #f0b840, #d4922a)", border: "none", borderRadius: 10, color: "#0f0c1a", fontSize: 15, fontWeight: 700, padding: "12px", cursor: "pointer", letterSpacing: 1, boxShadow: "0 4px 20px rgba(240,184,64,0.3)" }}
              onMouseDown={e => e.currentTarget.style.transform = "scale(0.97)"} onMouseUp={e => e.currentTarget.style.transform = "scale(1)"}>
              + Agregar trabajo
            </button>
          </div>
        </div>
      </div>

      {/* Toggle vista */}
      <div style={{ maxWidth: 560, margin: "16px auto 0", padding: "0 16px", display: "flex", gap: 0, background: "rgba(255,255,255,0.04)", borderRadius: 12, border: "1px solid rgba(255,255,255,0.08)", overflow: "hidden" }}>
        {["semana", "mes"].map(v => (
          <button key={v} onClick={() => setVista(v)} style={{ flex: 1, background: vista === v ? "rgba(255,200,80,0.18)" : "transparent", border: "none", color: vista === v ? "#f0b840" : "#7a6a5a", fontSize: 13, fontWeight: vista === v ? 700 : 400, padding: "10px", cursor: "pointer", fontFamily: "inherit", letterSpacing: 1, textTransform: "uppercase", transition: "all 0.2s" }}>
            {v === "semana" ? "Por semana" : "Resumen mensual"}
          </button>
        ))}
      </div>

      {/* VISTA SEMANA */}
      {vista === "semana" && (
        <div style={{ maxWidth: 560, margin: "14px auto 0", padding: "0 16px" }}>
          {semanasDeMesViendo.length > 1 && (
            <div style={{ display: "flex", gap: 6, overflowX: "auto", paddingBottom: 6, marginBottom: 12 }}>
              {semanasDeMesViendo.map(sem => (
                <button key={sem} onClick={() => setSemanaViendo(sem)} style={{ background: semanaViendo === sem ? "rgba(255,200,80,0.18)" : "rgba(255,255,255,0.04)", border: semanaViendo === sem ? "1px solid rgba(255,200,80,0.5)" : "1px solid rgba(255,255,255,0.1)", borderRadius: 20, color: semanaViendo === sem ? "#f0b840" : "#a89878", fontSize: 11, padding: "4px 12px", cursor: "pointer", whiteSpace: "nowrap", fontFamily: "inherit" }}>
                  {sem === semanaActual ? "● Esta semana" : `Sem. ${sem.split("-S")[1]}`}
                </button>
              ))}
            </div>
          )}
          <div style={{ fontSize: 10, letterSpacing: 3, color: "#a89878", textTransform: "uppercase", marginBottom: 10, textAlign: "center" }}>{getNombreSemana(semanaViendo)}</div>
          {trabajosDeSemana(semanaViendo).length === 0 ? (
            <div style={{ textAlign: "center", color: "#5a4a3a", padding: "36px 20px", border: "1px dashed rgba(255,255,255,0.08)", borderRadius: 14, fontSize: 14 }}>
              Sin trabajos esta semana.<br /><span style={{ fontSize: 12, marginTop: 6, display: "block" }}>¡Agregá el primero arriba!</span>
            </div>
          ) : (
            <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
              {trabajosDeSemana(semanaViendo).map((t, i) => (
                <div key={t.id} style={{ background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.09)", borderRadius: 12, padding: "12px 14px", display: "flex", alignItems: "center", gap: 10, animation: animando && i === trabajosDeSemana(semanaViendo).length - 1 ? "slideIn 0.4s ease" : "none" }}>
                  <div style={{ width: 30, height: 30, borderRadius: "50%", background: "linear-gradient(135deg, rgba(255,200,80,0.2), rgba(255,200,80,0.05))", border: "1px solid rgba(255,200,80,0.25)", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 11, color: "#f0b840", fontWeight: 700, flexShrink: 0 }}>{i + 1}</div>
                  <div style={{ flex: 1, minWidth: 0 }}>
                    <div style={{ color: "#e8d5b7", fontSize: 14, fontWeight: 600, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{t.descripcion}</div>
                    <div style={{ color: "#7a6a5a", fontSize: 11, marginTop: 2 }}>📅 {t.fecha}</div>
                  </div>
                  <div style={{ textAlign: "right", flexShrink: 0 }}>
                    <div style={{ color: "#a89878", fontSize: 12 }}>{fmt(t.precio)}</div>
                    <div style={{ color: "#f0b840", fontSize: 14, fontWeight: 700 }}>{fmt(t.ganancia)}</div>
                  </div>
                  <button onClick={() => eliminar(semanaViendo, t.id)} style={{ background: "rgba(255,60,60,0.1)", border: "1px solid rgba(255,60,60,0.2)", borderRadius: 8, color: "#ff6060", fontSize: 14, width: 26, height: 26, cursor: "pointer", display: "flex", alignItems: "center", justifyContent: "center", flexShrink: 0, padding: 0 }}>×</button>
                </div>
              ))}
            </div>
          )}
          {trabajosDeSemana(semanaViendo).length > 0 && (
            <div style={{ marginTop: 14, background: "linear-gradient(135deg, rgba(255,200,80,0.1), rgba(255,200,80,0.04))", border: "1px solid rgba(255,200,80,0.3)", borderRadius: 14, padding: "16px 18px" }}>
              <div style={{ fontSize: 10, letterSpacing: 4, color: "#f0b840", textTransform: "uppercase", marginBottom: 12 }}>Resumen semanal</div>
              <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6 }}><span style={{ color: "#a89878", fontSize: 13 }}>Trabajos:</span><span style={{ color: "#e8d5b7", fontSize: 13, fontWeight: 600 }}>{trabajosDeSemana(semanaViendo).length}</span></div>
              <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 6 }}><span style={{ color: "#a89878", fontSize: 13 }}>Facturado:</span><span style={{ color: "#e8d5b7", fontSize: 13, fontWeight: 600 }}>{fmt(totalPrecioSem(semanaViendo))}</span></div>
              <div style={{ display: "flex", justifyContent: "space-between", paddingTop: 10, borderTop: "1px solid rgba(255,200,80,0.2)" }}>
                <span style={{ color: "#f0b840", fontSize: 15 }}>Tu ganancia ({porcentaje}%):</span>
                <span style={{ color: "#f0b840", fontSize: 20, fontWeight: 700, textShadow: "0 0 20px rgba(255,200,80,0.5)" }}>{fmt(totalGananciaSem(semanaViendo))}</span>
              </div>
            </div>
          )}
        </div>
      )}

      {/* VISTA MES */}
      {vista === "mes" && (
        <div style={{ maxWidth: 560, margin: "14px auto 0", padding: "0 16px" }}>
          {mesesDisponibles.length > 1 && (
            <div style={{ display: "flex", gap: 6, overflowX: "auto", paddingBottom: 6, marginBottom: 12 }}>
              {mesesDisponibles.map(m => (
                <button key={m} onClick={() => setMesViendo(m)} style={{ background: mesViendo === m ? "rgba(255,200,80,0.18)" : "rgba(255,255,255,0.04)", border: mesViendo === m ? "1px solid rgba(255,200,80,0.5)" : "1px solid rgba(255,255,255,0.1)", borderRadius: 20, color: mesViendo === m ? "#
