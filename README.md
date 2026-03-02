import React, { useState, useEffect } from 'react';
import { 
  Settings, 
  UserPlus, 
  History, 
  Search, 
  Package, 
  Clock, 
  Trash2,
  Download,
  Timer,
  Save,
  Database,
  RefreshCcw,
  Cpu,
  CheckCircle,
  XCircle,
  ExternalLink
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  deleteDoc, 
  doc, 
  serverTimestamp 
} from 'firebase/firestore';
import { 
  getAuth, 
  signInWithCustomToken, 
  signInAnonymously, 
  onAuthStateChanged 
} from 'firebase/auth';

// --- CONFIGURACIÓN FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'makersago-sstt-2026';

// URL de tu Web App de Google App Script
const WEB_APP_URL = ""; 

const LOGO_URL = "https://yt3.googleusercontent.com/wDJ14y5wTpSE6ZsVCLFMbFaQtRczCJHdlRleD9vcN6f60dpW_itn7taUD3OSWgN9vHQcj1AX2Gw=s900-c-k-c0x00ffffff-no-rj";

const RESPONSABLES = ["Ronny Chacón", "Jazmin De la Cruz", "Jesús Junior Velásquez", "Saul Emanuel Mostacero"];
const MODELOS = [
  "IN01-WIFI/LAN", "IN01- LAN", "IN01- 4G", "SenceFace 2A", "SenceFace 3A", 
  "SenseFace 7A", "X628-C", "KA5000- 4G", "S922- 4G", "UFACE800-WIFI/LAN", 
  "SPEEDFACE-V4L", "SPEEDFACE-V5L", "MB10-VL WIFI", "160PRO", "460PRO"
];
const CONEXIONES = ["IP FIJA", "DHCP", "WIFI", "4G / 3G"];

export default function App() {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('config');
  const [registrosConfig, setRegistrosConfig] = useState([]);
  const [registrosCreacion, setRegistrosCreacion] = useState([]);
  const [loading, setLoading] = useState(true);
  
  const [timerActive, setTimerActive] = useState(null);
  const [startTime, setStartTime] = useState(null);
  const [elapsed, setElapsed] = useState(0);

  // Formulario Configuración Técnica
  const [formConfig, setFormConfig] = useState({
    ticket: '', servicio: 'Venta', salesOrder: '', notaVenta: '',
    tipoConfig: 'Asistencia', responsable: RESPONSABLES[0],
    modelo: MODELOS[0], serie: '', empresa: '', direccion: '',
    distrito: '', provincia: 'LIMA', sim: 'N/A', planilla: 'COMPLETA',
    conexion: 'IP FIJA', ip: '', subred: '255.255.255.0', mac: '',
    firmware: '', cantUsuarios: '', comentarios: ''
  });

  // Formulario Creación GeoVictoria
  const [formCreacion, setFormCreacion] = useState({
    ticket: '', serie: '', empresa: '', modelo: MODELOS[0],
    asistencia: true, acceso: false, casino: false,
    creado: true, grupos: true, cantUsuarios: ''
  });

  const cssStyles = `
    :root {
      --gv-celeste: #00ADEF;
      --gv-amarillo: #FFD100;
      --gv-oscuro: #1A2B3C;
    }
    .bg-gv-celeste { background-color: var(--gv-celeste); }
    .text-gv-celeste { color: var(--gv-celeste); }
    .bg-gv-amarillo { background-color: var(--gv-amarillo); }
    .border-gv-celeste { border-color: var(--gv-celeste); }
    
    .btn-primary { 
      background-color: var(--gv-celeste); 
      color: white;
      font-weight: 800;
      transition: all 0.3s;
    }
    .btn-primary:hover { 
      filter: brightness(1.1);
      transform: translateY(-1px);
    }
    .btn-secondary {
      background-color: var(--gv-amarillo);
      color: var(--gv-oscuro);
      font-weight: 800;
      transition: all 0.3s;
    }
    .input-gv {
      width: 100%;
      padding: 0.75rem 1rem;
      border: 2px solid #f1f5f9;
      border-radius: 0.75rem;
      outline: none;
      transition: all 0.2s;
    }
    .input-gv:focus {
      border-color: var(--gv-celeste);
      background: white;
    }
    .label-gv {
      font-size: 0.65rem;
      font-weight: 900;
      color: #94a3b8;
      text-transform: uppercase;
      margin-bottom: 0.25rem;
      display: block;
    }
  `;

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else { await signInAnonymously(auth); }
      } catch (err) { console.error("Auth error:", err); }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const unsubConfig = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'configuraciones'), (snap) => {
      setRegistrosConfig(snap.docs.map(d => ({ id: d.id, ...d.data() })).sort((a,b) => b.timestamp - a.timestamp));
    });
    const unsubCreacion = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'creaciones'), (snap) => {
      setRegistrosCreacion(snap.docs.map(d => ({ id: d.id, ...d.data() })).sort((a,b) => b.timestamp - a.timestamp));
      setLoading(false);
    });
    return () => { unsubConfig(); unsubCreacion(); };
  }, [user]);

  useEffect(() => {
    let interval;
    if (timerActive) {
      interval = setInterval(() => setElapsed(Math.floor((new Date() - startTime) / 1000)), 1000);
    }
    return () => clearInterval(interval);
  }, [timerActive, startTime]);

  const syncToGoogleSheets = async (data, type) => {
    if (!WEB_APP_URL) return;
    try {
      await fetch(WEB_APP_URL, {
        method: 'POST',
        mode: 'no-cors',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ ...data, type })
      });
    } catch (e) { console.error("Error en sincronización:", e); }
  };

  const handleStartTimer = (type) => {
    setStartTime(new Date());
    setTimerActive(type);
  };

  const handleSaveConfig = async () => {
    const endTime = new Date();
    const payload = {
      ...formConfig,
      fechaInicio: startTime.toLocaleDateString(),
      horaInicio: startTime.toLocaleTimeString(),
      fechaFin: endTime.toLocaleDateString(),
      horaFin: endTime.toLocaleTimeString(),
      duracion: ((endTime - startTime) / 60000).toFixed(2),
      timestamp: serverTimestamp()
    };
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'configuraciones'), payload);
      await syncToGoogleSheets(payload, 'CONFIGURACION');
      setTimerActive(null);
      setFormConfig({ ...formConfig, serie: '', ticket: '', empresa: '', mac: '', ip: '' });
    } catch (e) { console.error(e); }
  };

  const handleSaveCreacion = async () => {
    const endTime = new Date();
    const payload = {
      ...formCreacion,
      fechaInicio: startTime.toLocaleDateString(),
      horaInicio: startTime.toLocaleTimeString(),
      fechaFin: endTime.toLocaleDateString(),
      horaFin: endTime.toLocaleTimeString(),
      duracion: ((endTime - startTime) / 60000).toFixed(2),
      timestamp: serverTimestamp()
    };
    try {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'creaciones'), payload);
      await syncToGoogleSheets(payload, 'CREACION');
      setTimerActive(null);
      setFormCreacion({ ...formCreacion, serie: '', ticket: '', empresa: '', cantUsuarios: '' });
    } catch (e) { console.error(e); }
  };

  const formatTimer = (seconds) => {
    const m = Math.floor(seconds / 60);
    const s = seconds % 60;
    return `${m}:${s.toString().padStart(2, '0')}`;
  };

  const deleteItem = async (col, id) => {
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', col, id));
  };

  if (!user) return (
    <div className="h-screen flex items-center justify-center bg-[#1A2B3C] text-white font-black">
      <div className="text-center animate-pulse">
        <img src={LOGO_URL} alt="Logo" className="w-20 h-20 mx-auto mb-4 rounded-xl shadow-lg border-2 border-gv-amarillo" />
        GESTIÓN DE ALMACÉN
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-[#f1f5f9] text-slate-800 font-sans pb-20">
      <style dangerouslySetInnerHTML={{ __html: cssStyles }} />
      
      {/* HEADER CORPORATIVO */}
      <header className="bg-[#1A2B3C] text-white shadow-2xl sticky top-0 z-50">
        <div className="max-w-7xl mx-auto p-4 flex flex-col md:flex-row justify-between items-center gap-4">
          <div className="flex items-center gap-4">
            <div className="bg-white p-1 rounded-xl shadow-inner">
              <img src={LOGO_URL} alt="GeoVictoria Logo" className="w-12 h-12 object-contain" />
            </div>
            <div>
              <h1 className="font-black text-xl leading-none tracking-tight uppercase">GESTIÓN DE ALMACÉN</h1>
              <span className="text-[#00ADEF] font-black text-[10px] tracking-[0.3em] uppercase">SSTT • SOPORTE TÉCNICO</span>
            </div>
          </div>
          
          <nav className="flex bg-slate-800/40 p-1 rounded-2xl border border-slate-700">
            <button onClick={() => setActiveTab('config')} className={`px-6 py-2 rounded-xl text-[11px] font-black transition-all flex items-center gap-2 ${activeTab === 'config' ? 'bg-[#00ADEF] text-white shadow-lg' : 'text-slate-400'}`}>
              <Settings className="w-4 h-4" /> HARDWARE
            </button>
            <button onClick={() => setActiveTab('creacion')} className={`px-6 py-2 rounded-xl text-[11px] font-black transition-all flex items-center gap-2 ${activeTab === 'creacion' ? 'bg-[#FFD100] text-slate-900 shadow-lg' : 'text-slate-400'}`}>
              <UserPlus className="w-4 h-4" /> PLATAFORMA
            </button>
            <button onClick={() => setActiveTab('historial')} className={`px-6 py-2 rounded-xl text-[11px] font-black transition-all flex items-center gap-2 ${activeTab === 'historial' ? 'bg-white text-slate-900' : 'text-slate-400'}`}>
              <History className="w-4 h-4" /> REPORTES
            </button>
          </nav>
        </div>
      </header>

      <main className="max-w-7xl mx-auto p-6">
        
        {/* VISTA DE CONFIGURACIÓN FÍSICA */}
        {activeTab === 'config' && (
          <div className="grid grid-cols-1 lg:grid-cols-4 gap-8">
            <div className="lg:col-span-3 space-y-6">
              <div className="bg-white rounded-[2rem] shadow-sm border border-slate-200 overflow-hidden">
                <div className="bg-gv-celeste p-6 flex justify-between items-center">
                  <h2 className="text-white font-black text-sm uppercase tracking-widest flex items-center gap-2">
                    <Database className="w-5 h-5" /> Configuración de Hardware
                  </h2>
                </div>
                
                <div className="p-8 grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                  <div className="space-y-4">
                    <div>
                      <label className="label-gv">Ticket de Gestión</label>
                      <input className="input-gv" placeholder="TKT-XXXXXX" value={formConfig.ticket} onChange={e => setFormConfig({...formConfig, ticket: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Sales Order / Nota Venta</label>
                      <input className="input-gv" placeholder="SO-XXXX" value={formConfig.salesOrder} onChange={e => setFormConfig({...formConfig, salesOrder: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Responsable Core</label>
                      <select className="input-gv" value={formConfig.responsable} onChange={e => setFormConfig({...formConfig, responsable: e.target.value})}>
                        {RESPONSABLES.map(r => <option key={r} value={r}>{r}</option>)}
                      </select>
                    </div>
                  </div>

                  <div className="space-y-4">
                    <div>
                      <label className="label-gv">Modelo ZKTeco</label>
                      <select className="input-gv" value={formConfig.modelo} onChange={e => setFormConfig({...formConfig, modelo: e.target.value})}>
                        {MODELOS.map(m => <option key={m} value={m}>{m}</option>)}
                      </select>
                    </div>
                    <div>
                      <label className="label-gv text-gv-celeste">Número de Serie (SN)</label>
                      <input className="input-gv font-mono text-sm border-gv-celeste bg-blue-50/30" placeholder="Escanea el equipo..." value={formConfig.serie} onChange={e => setFormConfig({...formConfig, serie: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Empresa / Cliente</label>
                      <input className="input-gv" placeholder="Nombre del cliente" value={formConfig.empresa} onChange={e => setFormConfig({...formConfig, empresa: e.target.value})} />
                    </div>
                  </div>

                  <div className="space-y-4">
                    <div>
                      <label className="label-gv">Dirección IP Estática</label>
                      <input className="input-gv font-mono text-sm" placeholder="192.168.X.X" value={formConfig.ip} onChange={e => setFormConfig({...formConfig, ip: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">MAC Address</label>
                      <input className="input-gv font-mono text-sm" placeholder="00:17:..." value={formConfig.mac} onChange={e => setFormConfig({...formConfig, mac: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Versión de Firmware</label>
                      <input className="input-gv text-sm" placeholder="8.0.4.1..." value={formConfig.firmware} onChange={e => setFormConfig({...formConfig, firmware: e.target.value})} />
                    </div>
                  </div>
                  
                  <div className="md:col-span-2 lg:col-span-3">
                    <label className="label-gv">Observaciones del SSTT</label>
                    <textarea className="input-gv h-24" placeholder="Indique detalles de la configuración..." value={formConfig.comentarios} onChange={e => setFormConfig({...formConfig, comentarios: e.target.value})}></textarea>
                  </div>
                </div>
              </div>
            </div>

            <div className="space-y-6">
              <div className="bg-white p-8 rounded-[2rem] shadow-xl border-t-8 border-gv-celeste text-center sticky top-28">
                <div className="mb-6">
                  <div className={`w-20 h-20 mx-auto rounded-full flex items-center justify-center ${timerActive === 'config' ? 'bg-blue-50 text-gv-celeste animate-pulse' : 'bg-slate-50 text-slate-300'}`}>
                    <Timer className="w-10 h-10" />
                  </div>
                </div>
                <h4 className="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Tiempo de Configuración</h4>
                <div className="text-6xl font-mono font-black text-slate-900 mb-8 tracking-tighter">
                  {timerActive === 'config' ? formatTimer(elapsed) : "00:00"}
                </div>
                
                {timerActive === 'config' ? (
                  <button onClick={handleSaveConfig} className="w-full btn-primary py-5 rounded-2xl flex items-center justify-center gap-2 text-sm shadow-lg">
                    <Save className="w-5 h-5" /> TERMINAR Y GUARDAR
                  </button>
                ) : (
                  <button onClick={() => handleStartTimer('config')} className="w-full bg-[#1A2B3C] text-white font-black py-5 rounded-2xl hover:bg-black transition-all flex items-center justify-center gap-2 text-sm shadow-lg">
                    <Timer className="w-5 h-5" /> INICIAR TRABAJO
                  </button>
                )}
              </div>
            </div>
          </div>
        )}

        {/* VISTA DE PLATAFORMA */}
        {activeTab === 'creacion' && (
          <div className="grid grid-cols-1 lg:grid-cols-4 gap-8">
            <div className="lg:col-span-3 space-y-6">
              <div className="bg-white rounded-[2rem] shadow-sm border border-slate-200 overflow-hidden">
                <div className="bg-gv-amarillo p-6 flex justify-between items-center">
                  <h2 className="text-slate-900 font-black text-sm uppercase tracking-widest flex items-center gap-2">
                    <UserPlus className="w-5 h-5" /> Carga Lógica en Plataforma
                  </h2>
                </div>
                
                <div className="p-8 grid grid-cols-1 md:grid-cols-2 gap-8">
                  <div className="space-y-5">
                    <div>
                      <label className="label-gv">Ticket Relacionado</label>
                      <input className="input-gv" placeholder="TKT-XXXXXX" value={formCreacion.ticket} onChange={e => setFormCreacion({...formCreacion, ticket: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Validar Serie (SN)</label>
                      <input className="input-gv font-mono text-sm" placeholder="CQUH..." value={formCreacion.serie} onChange={e => setFormCreacion({...formCreacion, serie: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Empresa GeoVictoria</label>
                      <input className="input-gv" placeholder="Nombre en plataforma" value={formCreacion.empresa} onChange={e => setFormCreacion({...formCreacion, empresa: e.target.value})} />
                    </div>
                    <div>
                      <label className="label-gv">Usuarios a Cargar</label>
                      <input type="number" className="input-gv" placeholder="0" value={formCreacion.cantUsuarios} onChange={e => setFormCreacion({...formCreacion, cantUsuarios: e.target.value})} />
                    </div>
                  </div>

                  <div className="bg-slate-50 p-8 rounded-[1.5rem] border border-slate-100 shadow-inner">
                    <h4 className="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-6 border-b pb-2">Checklist de Plataforma</h4>
                    <div className="grid grid-cols-1 gap-3">
                      {[
                        { id: 'asistencia', label: 'Modulo Asistencia', icon: CheckCircle },
                        { id: 'acceso', label: 'Modulo Acceso', icon: CheckCircle },
                        { id: 'casino', label: 'Modulo Casino', icon: CheckCircle },
                        { id: 'creado', label: 'Equipo Creado', icon: Cpu },
                        { id: 'grupos', label: 'Grupos Cargados', icon: Database },
                      ].map(item => (
                        <label key={item.id} className={`flex items-center justify-between p-4 rounded-xl border-2 transition-all cursor-pointer ${formCreacion[item.id] ? 'bg-white border-gv-celeste' : 'bg-slate-100 border-transparent text-slate-400'}`}>
                          <div className="flex items-center gap-3">
                            <item.icon className={`w-4 h-4 ${formCreacion[item.id] ? 'text-gv-celeste' : 'text-slate-300'}`} />
                            <span className="text-xs font-black uppercase tracking-tight">{item.label}</span>
                          </div>
                          <input 
                            type="checkbox" 
                            className="w-5 h-5 accent-gv-celeste rounded"
                            checked={formCreacion[item.id]} 
                            onChange={e => setFormCreacion({...formCreacion, [item.id]: e.target.checked})} 
                          />
                        </label>
                      ))}
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <div className="space-y-6">
              <div className="bg-white p-8 rounded-[2rem] shadow-xl border-t-8 border-gv-amarillo text-center sticky top-28">
                <div className="mb-6">
                  <div className={`w-20 h-20 mx-auto rounded-full flex items-center justify-center ${timerActive === 'creacion' ? 'bg-yellow-50 text-gv-amarillo animate-pulse' : 'bg-slate-50 text-slate-300'}`}>
                    <Cpu className="w-10 h-10 text-[#FFD100]" />
                  </div>
                </div>
                <h4 className="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Tiempo en Plataforma</h4>
                <div className="text-6xl font-mono font-black text-slate-900 mb-8 tracking-tighter">
                  {timerActive === 'creacion' ? formatTimer(elapsed) : "00:00"}
                </div>
                
                {timerActive === 'creacion' ? (
                  <button onClick={handleSaveCreacion} className="w-full btn-secondary py-5 rounded-2xl flex items-center justify-center gap-2 text-sm shadow-lg">
                    <Save className="w-5 h-5" /> GUARDAR EN CORE
                  </button>
                ) : (
                  <button onClick={() => handleStartTimer('creacion')} className="w-full bg-[#1A2B3C] text-white font-black py-5 rounded-2xl hover:bg-black transition-all flex items-center justify-center gap-2 text-sm shadow-lg">
                    <Timer className="w-5 h-5" /> INICIAR CARGA
                  </button>
                )}
              </div>
            </div>
          </div>
        )}

        {/* VISTA DE REPORTES */}
        {activeTab === 'historial' && (
          <div className="space-y-8 animate-in fade-in duration-500">
            <div className="bg-white rounded-[2rem] shadow-sm border border-slate-200 overflow-hidden">
              <div className="p-5 bg-gv-celeste text-white flex justify-between items-center">
                <span className="font-black text-xs uppercase tracking-widest">Hardware Registrado</span>
                <RefreshCcw className="w-4 h-4" />
              </div>
              <div className="overflow-x-auto">
                <table className="w-full text-left text-sm">
                  <thead className="bg-slate-50 text-[10px] text-slate-400 font-black uppercase tracking-widest border-b border-slate-100">
                    <tr>
                      <th className="p-4">TKT</th>
                      <th className="p-4">Serie</th>
                      <th className="p-4">Cliente</th>
                      <th className="p-4">Duración</th>
                      <th className="p-4">Finalización</th>
                      <th className="p-4"></th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-slate-100">
                    {registrosConfig.map(r => (
                      <tr key={r.id} className="hover:bg-blue-50/30 group transition-colors">
                        <td className="p-4 font-black text-gv-celeste">{r.ticket}</td>
                        <td className="p-4 font-mono text-xs text-slate-500">{r.serie}</td>
                        <td className="p-4 text-xs font-bold">{r.empresa}</td>
                        <td className="p-4">
                          <span className="bg-blue-100 text-gv-celeste px-2 py-1 rounded-full text-[10px] font-black">{r.duracion} min</span>
                        </td>
                        <td className="p-4 text-[10px] text-slate-400 font-bold">{r.fechaFin} {r.horaFin}</td>
                        <td className="p-4 text-right">
                          <button onClick={() => deleteItem('configuraciones', r.id)} className="text-slate-200 hover:text-red-500 transition-colors opacity-0 group-hover:opacity-100"><Trash2 className="w-4 h-4" /></button>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>

            <div className="bg-white rounded-[2rem] shadow-sm border border-slate-200 overflow-hidden">
              <div className="p-5 bg-gv-amarillo text-slate-900 flex justify-between items-center">
                <span className="font-black text-xs uppercase tracking-widest">Plataformas Cargadas</span>
                <CheckCircle className="w-4 h-4" />
              </div>
              <div className="overflow-x-auto">
                <table className="w-full text-left text-sm">
                  <thead className="bg-slate-50 text-[10px] text-slate-400 font-black uppercase tracking-widest border-b border-slate-100">
                    <tr>
                      <th className="p-4">Ticket</th>
                      <th className="p-4">Serie</th>
                      <th className="p-4">Usuarios</th>
                      <th className="p-4">Duración</th>
                      <th className="p-4"></th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-slate-100">
                    {registrosCreacion.map(r => (
                      <tr key={r.id} className="hover:bg-yellow-50/30 group transition-colors">
                        <td className="p-4 font-black">{r.ticket}</td>
                        <td className="p-4 font-mono text-xs">{r.serie}</td>
                        <td className="p-4 font-black text-slate-400">{r.cantUsuarios}</td>
                        <td className="p-4 text-gv-celeste font-black">{r.duracion} min</td>
                        <td className="p-4 text-right">
                          <button onClick={() => deleteItem('creaciones', r.id)} className="text-slate-200 hover:text-red-500 transition-colors opacity-0 group-hover:opacity-100"><Trash2 className="w-4 h-4" /></button>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}
      </main>

      {/* NOTIFICACIÓN FLOTANTE */}
      {timerActive && (
        <div className="fixed bottom-10 left-1/2 -translate-x-1/2 bg-[#1A2B3C] border-2 border-gv-celeste text-white px-8 py-4 rounded-full shadow-2xl flex items-center gap-6 z-50 animate-bounce">
          <div className="flex items-center gap-2">
            <div className={`w-3 h-3 rounded-full ${timerActive === 'config' ? 'bg-gv-celeste' : 'bg-gv-amarillo'} animate-ping`}></div>
            <span className="text-[10px] font-black uppercase tracking-[0.2em]">{timerActive === 'config' ? 'Hardware' : 'Plataforma'}</span>
          </div>
          <div className="text-2xl font-mono font-black text-[#FFD100]">{formatTimer(elapsed)}</div>
        </div>
      )}

      {/* FOOTER */}
      <footer className="fixed bottom-0 w-full bg-white border-t border-slate-200 p-2 text-center text-[9px] text-slate-400 font-bold uppercase tracking-widest">
        Powered by MAKERSAGOS S.A.C • 2026
      </footer>
    </div>
  );
}
