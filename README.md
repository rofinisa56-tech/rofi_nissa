import React, { useState, useMemo, useEffect } from 'react';
import { 
  LayoutDashboard, 
  PlusCircle, 
  FileText, 
  ClipboardList, 
  PieChart, 
  TrendingUp, 
  TrendingDown, 
  Wallet,
  Calendar,
  Info,
  ChevronRight,
  Download,
  Printer,
  Sparkles,
  Loader2,
  AlertCircle,
  Edit2,
  Trash2,
  X,
  Check,
  Save
} from 'lucide-react';

// --- Configuration ---
const apiKey = ""; 

// --- Default Data ---
const INITIAL_TRANSACTIONS = [
  { id: 1, date: '2023-10-01', desc: 'Saldo Awal Kas', category: 'Modal', type: 'in', amount: 5000000 },
  { id: 2, date: '2023-10-05', desc: 'Iuran Anggota Oktober', category: 'Iuran', type: 'in', amount: 200000 },
  { id: 3, date: '2023-10-10', desc: 'Konsumsi Rapat Bulanan', category: 'Konsumsi', type: 'out', amount: 150000 },
  { id: 4, date: '2023-10-15', desc: 'Sewa Lapangan Futsal', category: 'Olahraga', type: 'out', amount: 300000 },
];

const INITIAL_EVENTS = [
  { id: 1, name: 'Turnamen Futsal Cup', budget: 2000000, date: '2023-11-20', status: 'Planning', notes: 'Rencana 16 tim pendaftaran.' },
  { id: 2, name: 'Bakti Sosial Desa', budget: 1500000, date: '2023-12-10', status: 'Draft', notes: 'Pembagian sembako ke 50 KK.' }
];

const App = () => {
  const [activeTab, setActiveTab] = useState('dashboard');
  const [transactions, setTransactions] = useState(INITIAL_TRANSACTIONS);
  const [events, setEvents] = useState(INITIAL_EVENTS);
  
  // Form States
  const [newTx, setNewTx] = useState({ date: '', desc: '', category: '', type: 'in', amount: '' });
  const [newEv, setNewEv] = useState({ name: '', budget: '', date: '', notes: '' });

  // Editing States
  const [editingTxId, setEditingTxId] = useState(null);
  const [editTxData, setEditTxData] = useState({});
  const [editingEvId, setEditingEvId] = useState(null);
  const [editEvData, setEditEvData] = useState({});

  // Gemini AI States
  const [aiLoading, setAiLoading] = useState(false);
  const [aiAnalysis, setAiAnalysis] = useState("");
  const [aiError, setAiError] = useState("");

  // --- Gemini API Handler ---
  const callGemini = async (prompt, systemPrompt = "Anda adalah asisten keuangan profesional untuk organisasi kepemudaan.") => {
    setAiLoading(true);
    setAiError("");
    let retries = 0;
    const maxRetries = 5;

    const performCall = async () => {
      try {
        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            systemInstruction: { parts: [{ text: systemPrompt }] }
          })
        });
        
        if (!response.ok) throw new Error('API request failed');
        const result = await response.json();
        return result.candidates?.[0]?.content?.parts?.[0]?.text;
      } catch (err) {
        if (retries < maxRetries) {
          retries++;
          const delay = Math.pow(2, retries) * 500;
          await new Promise(res => setTimeout(res, delay));
          return performCall();
        }
        throw err;
      }
    };

    try {
      const text = await performCall();
      setAiLoading(false);
      return text;
    } catch (err) {
      setAiLoading(false);
      setAiError("Gagal terhubung dengan asisten AI. Silakan coba lagi nanti.");
      return null;
    }
  };

  const handleAiFinancialAnalysis = async () => {
    const dataSummary = `
      Data Keuangan:
      Total Pemasukan: ${formatIDR(totalIn)}
      Total Pengeluaran: ${formatIDR(totalOut)}
      Saldo Akhir: ${formatIDR(balance)}
      Daftar Event Terencana: ${events.map(e => e.name + " (Budget: " + formatIDR(e.budget) + ")").join(", ")}
    `;
    
    const prompt = `Berikan analisis singkat dan saran strategis untuk Karang Taruna berdasarkan data berikut: ${dataSummary}. Fokus pada efisiensi anggaran dan keberlanjutan kas.`;
    const result = await callGemini(prompt, "Anda adalah konsultan keuangan ahli Karang Taruna. Berikan saran yang ramah, praktis, dan profesional dalam bahasa Indonesia.");
    if (result) setAiAnalysis(result);
  };

  const handleAiRABSuggestion = async () => {
    const name = editingEvId ? editEvData.name : newEv.name;
    if (!name) return;
    const prompt = `Buatlah rincian komponen biaya (RAB) singkat untuk kegiatan Karang Taruna bernama: "${name}". Berikan dalam format poin-poin singkat dengan estimasi persentase alokasi budget.`;
    const result = await callGemini(prompt, "Anda adalah ahli perencanaan event. Berikan rekomendasi rincian biaya yang masuk akal untuk organisasi pemuda.");
    if (result) {
      if (editingEvId) {
        setEditEvData({ ...editEvData, notes: result });
      } else {
        setNewEv({ ...newEv, notes: result });
      }
    }
  };

  // --- CRUD Handlers ---
  
  // Transactions
  const startEditTx = (tx) => {
    setEditingTxId(tx.id);
    setEditTxData(tx);
  };

  const saveEditTx = () => {
    setTransactions(transactions.map(t => t.id === editingTxId ? { ...editTxData, amount: parseFloat(editTxData.amount) } : t));
    setEditingTxId(null);
  };

  const deleteTx = (id) => {
    if (window.confirm("Hapus transaksi ini?")) {
      setTransactions(transactions.filter(t => t.id !== id));
    }
  };

  // Events
  const startEditEv = (ev) => {
    setEditingEvId(ev.id);
    setEditEvData(ev);
    setActiveTab('rab'); // Ensure we are on RAB tab
  };

  const saveEditEv = () => {
    setEvents(events.map(e => e.id === editingEvId ? { ...editEvData, budget: parseFloat(editEvData.budget) } : e));
    setEditingEvId(null);
    setEditEvData({});
  };

  const deleteEv = (id) => {
    if (window.confirm("Hapus rencana event ini?")) {
      setEvents(events.filter(e => e.id !== id));
    }
  };

  // --- Calculations ---
  const totalIn = useMemo(() => transactions.filter(t => t.type === 'in').reduce((acc, curr) => acc + curr.amount, 0), [transactions]);
  const totalOut = useMemo(() => transactions.filter(t => t.type === 'out').reduce((acc, curr) => acc + curr.amount, 0), [transactions]);
  const balance = totalIn - totalOut;

  const formatIDR = (val) => new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR' }).format(val);

  // --- Sidebar Component ---
  const SidebarItem = ({ id, icon: Icon, label }) => (
    <button
      onClick={() => {
        setActiveTab(id);
        setEditingEvId(null); // Reset editing when switching
      }}
      className={`w-full flex items-center space-x-3 p-3 rounded-lg transition-colors ${
        activeTab === id ? 'bg-blue-600 text-white shadow-md' : 'text-gray-600 hover:bg-gray-100'
      }`}
    >
      <Icon size={20} />
      <span className="font-medium">{label}</span>
    </button>
  );

  return (
    <div className="flex min-h-screen bg-gray-50 text-gray-800 font-sans">
      {/* Sidebar */}
      <aside className="w-64 bg-white border-r hidden md:block p-6">
        <div className="flex items-center space-x-2 mb-10">
          <div className="bg-blue-600 p-2 rounded-lg text-white">
            <PieChart size={24} />
          </div>
          <h1 className="text-xl font-bold tracking-tight text-blue-900">KT-Finance</h1>
        </div>
        
        <nav className="space-y-2">
          <SidebarItem id="dashboard" icon={LayoutDashboard} label="Dashboard" />
          <SidebarItem id="transactions" icon={PlusCircle} label="Transaksi" />
          <SidebarItem id="rab" icon={ClipboardList} label="RAB & Event" />
          <SidebarItem id="reports" icon={FileText} label="Laporan Lengkap" />
        </nav>
      </aside>

      {/* Main Content */}
      <main className="flex-1 overflow-y-auto p-4 md:p-8">
        <header className="flex flex-col md:flex-row md:items-center justify-between mb-8 gap-4">
          <div>
            <h2 className="text-2xl font-bold text-gray-900 capitalize">{activeTab.replace('-', ' ')}</h2>
            <p className="text-gray-500">Sistem Informasi Keuangan Karang Taruna</p>
          </div>
          <div className="flex items-center space-x-3 bg-white p-2 rounded-xl shadow-sm border">
            <Calendar size={18} className="text-gray-400" />
            <span className="text-sm font-medium">{new Date().toLocaleDateString('id-ID', { day: 'numeric', month: 'long', year: 'numeric' })}</span>
          </div>
        </header>

        {/* Dashboard View */}
        {activeTab === 'dashboard' && (
          <div className="space-y-6">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
              <div className="bg-white p-6 rounded-2xl shadow-sm border-b-4 border-blue-500">
                <div className="flex justify-between items-start">
                  <div>
                    <p className="text-sm text-gray-500 mb-1">Total Kas (Aset)</p>
                    <h3 className="text-2xl font-bold">{formatIDR(balance)}</h3>
                  </div>
                  <div className="p-3 bg-blue-50 text-blue-600 rounded-xl"><Wallet size={24}/></div>
                </div>
              </div>
              <div className="bg-white p-6 rounded-2xl shadow-sm border-b-4 border-green-500">
                <div className="flex justify-between items-start">
                  <div>
                    <p className="text-sm text-gray-500 mb-1">Pemasukan</p>
                    <h3 className="text-2xl font-bold text-green-600">{formatIDR(totalIn)}</h3>
                  </div>
                  <div className="p-3 bg-green-50 text-green-600 rounded-xl"><TrendingUp size={24}/></div>
                </div>
              </div>
              <div className="bg-white p-6 rounded-2xl shadow-sm border-b-4 border-red-500">
                <div className="flex justify-between items-start">
                  <div>
                    <p className="text-sm text-gray-500 mb-1">Pengeluaran</p>
                    <h3 className="text-2xl font-bold text-red-600">{formatIDR(totalOut)}</h3>
                  </div>
                  <div className="p-3 bg-red-50 text-red-600 rounded-xl"><TrendingDown size={24}/></div>
                </div>
              </div>
            </div>

            <div className="bg-white p-6 rounded-2xl shadow-sm border">
              <h4 className="font-bold mb-4 flex items-center gap-2"><PlusCircle size={18}/> Transaksi Terakhir</h4>
              <div className="overflow-x-auto">
                <table className="w-full text-left">
                  <thead>
                    <tr className="text-xs font-bold text-gray-400 border-b">
                      <th className="pb-2">TANGGAL</th>
                      <th className="pb-2">KETERANGAN</th>
                      <th className="pb-2 text-right">JUMLAH</th>
                      <th className="pb-2 text-center">AKSI</th>
                    </tr>
                  </thead>
                  <tbody className="divide-y">
                    {transactions.map(t => (
                      <tr key={t.id} className="text-sm hover:bg-gray-50 group">
                        <td className="py-3 text-gray-500">{t.date}</td>
                        <td className="py-3 font-medium">{t.desc}</td>
                        <td className={`py-3 text-right font-bold ${t.type === 'in' ? 'text-green-600' : 'text-red-600'}`}>
                          {t.type === 'in' ? '+' : '-'} {formatIDR(t.amount)}
                        </td>
                        <td className="py-3 text-center">
                          <div className="flex justify-center gap-2 opacity-0 group-hover:opacity-100 transition">
                            <button onClick={() => { setActiveTab('transactions'); startEditTx(t); }} className="p-1.5 text-blue-600 hover:bg-blue-50 rounded-lg"><Edit2 size={14}/></button>
                            <button onClick={() => deleteTx(t.id)} className="p-1.5 text-red-600 hover:bg-red-50 rounded-lg"><Trash2 size={14}/></button>
                          </div>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}

        {/* Transactions View */}
        {activeTab === 'transactions' && (
          <div className="max-w-4xl mx-auto space-y-6">
            <div className="bg-white p-8 rounded-2xl shadow-sm border">
              <div className="flex justify-between items-center mb-6">
                <h4 className="text-lg font-bold">{editingTxId ? 'Edit Transaksi' : 'Input Transaksi Baru'}</h4>
                {editingTxId && (
                  <button onClick={() => setEditingTxId(null)} className="text-gray-400 hover:text-red-500 flex items-center gap-1 text-sm font-bold">
                    <X size={16}/> Batal Edit
                  </button>
                )}
              </div>
              
              <form onSubmit={(e) => {
                e.preventDefault();
                if (editingTxId) {
                  saveEditTx();
                } else {
                  setTransactions([{ ...newTx, id: Date.now(), amount: parseFloat(newTx.amount) }, ...transactions]);
                  setNewTx({ date: '', desc: '', category: '', type: 'in', amount: '' });
                }
              }} className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div>
                  <label className="block text-sm font-semibold mb-2">Tanggal</label>
                  <input 
                    type="date" 
                    className="w-full p-3 bg-gray-50 border rounded-xl focus:ring-2 focus:ring-blue-500 outline-none" 
                    value={editingTxId ? editTxData.date : newTx.date}
                    onChange={e => editingTxId ? setEditTxData({...editTxData, date: e.target.value}) : setNewTx({...newTx, date: e.target.value})}
                    required
                  />
                </div>
                <div>
                  <label className="block text-sm font-semibold mb-2">Jenis</label>
                  <select 
                    className="w-full p-3 bg-gray-50 border rounded-xl focus:ring-2 focus:ring-blue-500 outline-none"
                    value={editingTxId ? editTxData.type : newTx.type}
                    onChange={e => editingTxId ? setEditTxData({...editTxData, type: e.target.value}) : setNewTx({...newTx, type: e.target.value})}
                  >
                    <option value="in">Pemasukan (+)</option>
                    <option value="out">Pengeluaran (-)</option>
                  </select>
                </div>
                <div className="md:col-span-2">
                  <label className="block text-sm font-semibold mb-2">Keterangan</label>
                  <input 
                    type="text" 
                    className="w-full p-3 bg-gray-50 border rounded-xl focus:ring-2 focus:ring-blue-500 outline-none"
                    value={editingTxId ? editTxData.desc : newTx.desc}
                    onChange={e => editingTxId ? setEditTxData({...editTxData, desc: e.target.value}) : setNewTx({...newTx, desc: e.target.value})}
                    required
                  />
                </div>
                <div>
                  <label className="block text-sm font-semibold mb-2">Kategori</label>
                  <input 
                    type="text" 
                    className="w-full p-3 bg-gray-50 border rounded-xl focus:ring-2 focus:ring-blue-500 outline-none"
                    value={editingTxId ? editTxData.category : newTx.category}
                    onChange={e => editingTxId ? setEditTxData({...editTxData, category: e.target.value}) : setNewTx({...newTx, category: e.target.value})}
                  />
                </div>
                <div>
                  <label className="block text-sm font-semibold mb-2">Jumlah (Rp)</label>
                  <input 
                    type="number" 
                    className="w-full p-3 bg-gray-50 border rounded-xl focus:ring-2 focus:ring-blue-500 outline-none font-bold text-blue-600"
                    value={editingTxId ? editTxData.amount : newTx.amount}
                    onChange={e => editingTxId ? setEditTxData({...editTxData, amount: e.target.value}) : setNewTx({...newTx, amount: e.target.value})}
                    required
                  />
                </div>
                <div className="md:col-span-2 pt-4">
                  <button type="submit" className={`w-full text-white font-bold py-4 rounded-xl transition shadow-lg flex items-center justify-center gap-2 ${editingTxId ? 'bg-orange-600 hover:bg-orange-700 shadow-orange-200' : 'bg-blue-600 hover:bg-blue-700 shadow-blue-200'}`}>
                    {editingTxId ? <Save size={18}/> : <PlusCircle size={18}/>}
                    {editingTxId ? 'Simpan Perubahan' : 'Tambah Transaksi'}
                  </button>
                </div>
              </form>
            </div>
            
            <div className="bg-white p-6 rounded-2xl shadow-sm border">
              <h4 className="font-bold mb-4">Riwayat Transaksi Lengkap</h4>
              <div className="divide-y">
                {transactions.map(t => (
                  <div key={t.id} className="py-4 flex justify-between items-center hover:bg-gray-50 px-2 rounded-lg transition group">
                    <div className="flex gap-4 items-center">
                      <div className={`p-2 rounded-lg ${t.type === 'in' ? 'bg-green-50 text-green-600' : 'bg-red-50 text-red-600'}`}>
                        {t.type === 'in' ? <TrendingUp size={20}/> : <TrendingDown size={20}/>}
                      </div>
                      <div>
                        <p className="font-bold text-sm">{t.desc}</p>
                        <p className="text-xs text-gray-500">{t.date} • {t.category}</p>
                      </div>
                    </div>
                    <div className="flex items-center gap-6">
                       <p className={`font-bold ${t.type === 'in' ? 'text-green-600' : 'text-red-600'}`}>
                         {t.type === 'in' ? '+' : '-'} {formatIDR(t.amount)}
                       </p>
                       <div className="flex gap-1 opacity-0 group-hover:opacity-100 transition">
                          <button onClick={() => startEditTx(t)} className="p-2 text-gray-400 hover:text-blue-600 hover:bg-blue-50 rounded-lg transition"><Edit2 size={16}/></button>
                          <button onClick={() => deleteTx(t.id)} className="p-2 text-gray-400 hover:text-red-600 hover:bg-red-50 rounded-lg transition"><Trash2 size={16}/></button>
                       </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}

        {/* RAB & Event View */}
        {activeTab === 'rab' && (
          <div className="space-y-6">
            <div className="bg-white p-6 rounded-2xl shadow-sm border">
              <div className="flex justify-between items-center mb-6">
                <h4 className="text-lg font-bold">{editingEvId ? 'Edit Rencana Event' : 'Buat Rencana Anggaran (RAB) / Event'}</h4>
                {editingEvId && (
                  <button onClick={() => { setEditingEvId(null); setEditEvData({}); }} className="text-gray-400 hover:text-red-500 flex items-center gap-1 text-sm font-bold">
                    <X size={16}/> Batal Edit
                  </button>
                )}
              </div>
              
              <div className="grid grid-cols-1 md:grid-cols-4 gap-4 items-end mb-4">
                <div className="md:col-span-1">
                  <label className="text-xs font-bold uppercase text-gray-500 mb-1 block">Nama Event</label>
                  <input 
                    type="text" className="w-full p-2 bg-gray-50 border rounded-lg text-sm" placeholder="HUT RI..." 
                    value={editingEvId ? editEvData.name : newEv.name} 
                    onChange={e => editingEvId ? setEditEvData({...editEvData, name: e.target.value}) : setNewEv({...newEv, name: e.target.value})}
                  />
                </div>
                <div>
                  <label className="text-xs font-bold uppercase text-gray-500 mb-1 block">Budget</label>
                  <input 
                    type="number" className="w-full p-2 bg-gray-50 border rounded-lg text-sm" placeholder="Rp" 
                    value={editingEvId ? editEvData.budget : newEv.budget} 
                    onChange={e => editingEvId ? setEditEvData({...editEvData, budget: e.target.value}) : setNewEv({...newEv, budget: e.target.value})}
                  />
                </div>
                <div>
                  <label className="text-xs font-bold uppercase text-gray-500 mb-1 block">Tanggal</label>
                  <input 
                    type="date" className="w-full p-2 bg-gray-50 border rounded-lg text-sm" 
                    value={editingEvId ? editEvData.date : newEv.date} 
                    onChange={e => editingEvId ? setEditEvData({...editEvData, date: e.target.value}) : setNewEv({...newEv, date: e.target.value})}
                  />
                </div>
                <div className="flex gap-2">
                  <button 
                    onClick={handleAiRABSuggestion}
                    disabled={aiLoading || (editingEvId ? !editEvData.name : !newEv.name)}
                    className="flex-1 bg-indigo-100 text-indigo-700 p-2 rounded-lg font-bold text-xs hover:bg-indigo-200 transition disabled:opacity-50 flex items-center justify-center gap-1"
                  >
                    {aiLoading ? <Loader2 size={14} className="animate-spin" /> : <Sparkles size={14}/>} 
                    Saran AI ✨
                  </button>
                  <button 
                    onClick={() => {
                      if (editingEvId) {
                        saveEditEv();
                      } else {
                        setEvents([{ ...newEv, id: Date.now(), budget: parseFloat(newEv.budget), status: 'Planning' }, ...events]);
                        setNewEv({ name: '', budget: '', date: '', notes: '' });
                      }
                    }} 
                    className={`flex-1 text-white p-2 rounded-lg font-bold text-xs transition ${editingEvId ? 'bg-orange-600 hover:bg-orange-700' : 'bg-blue-600 hover:bg-blue-700'}`}
                  >
                    {editingEvId ? 'Update' : 'Tambah'}
                  </button>
                </div>
              </div>
              <div>
                <label className="text-xs font-bold uppercase text-gray-500 mb-1 block">Detail Rencana</label>
                <textarea 
                  className="w-full p-3 bg-gray-50 border rounded-xl text-sm h-40" 
                  placeholder="Gunakan 'Saran AI' untuk bantuan..."
                  value={editingEvId ? editEvData.notes : newEv.notes}
                  onChange={e => editingEvId ? setEditEvData({...editEvData, notes: e.target.value}) : setNewEv({...newEv, notes: e.target.value})}
                ></textarea>
              </div>
            </div>

            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
              {events.map(ev => (
                <div key={ev.id} className={`bg-white p-6 rounded-2xl shadow-sm border transition group relative ${editingEvId === ev.id ? 'border-orange-500 ring-2 ring-orange-100 shadow-lg' : 'hover:shadow-md'}`}>
                  <div className="flex justify-between items-start mb-4">
                    <h5 className="font-bold text-lg">{ev.name}</h5>
                    <div className="flex gap-1">
                       <button onClick={() => startEditEv(ev)} className="p-1.5 text-gray-400 hover:text-blue-600 hover:bg-blue-50 rounded-lg"><Edit2 size={14}/></button>
                       <button onClick={() => deleteEv(ev.id)} className="p-1.5 text-gray-400 hover:text-red-600 hover:bg-red-50 rounded-lg"><Trash2 size={14}/></button>
                    </div>
                  </div>
                  <div className="space-y-2 mb-4">
                    <div className="flex justify-between text-sm">
                      <span className="text-gray-500">Anggaran:</span>
                      <span className="font-bold">{formatIDR(ev.budget)}</span>
                    </div>
                    <div className="flex justify-between text-sm">
                      <span className="text-gray-500">Tanggal:</span>
                      <span className="font-medium text-blue-600">{ev.date}</span>
                    </div>
                  </div>
                  <div className="bg-gray-50 p-3 rounded-xl">
                    <p className="text-[11px] font-bold text-gray-400 uppercase mb-1">Rincian:</p>
                    <p className="text-xs text-gray-600 italic line-clamp-4 whitespace-pre-line">
                      {ev.notes || "Belum ada rincian rencana."}
                    </p>
                  </div>
                  <div className="mt-4 pt-3 border-t flex items-center justify-between">
                     <span className="px-2 py-0.5 bg-blue-50 text-blue-700 rounded text-[10px] font-black uppercase tracking-widest">{ev.status}</span>
                     {editingEvId === ev.id && <span className="text-[10px] text-orange-600 font-bold animate-pulse">Sedang Diedit...</span>}
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

        {/* Reports View */}
        {activeTab === 'reports' && (
          <div className="space-y-8 pb-12">
            <div className="flex flex-col md:flex-row justify-between items-end gap-4 print:hidden">
              <div className="bg-indigo-50 border border-indigo-100 p-4 rounded-xl flex-1 w-full md:w-auto">
                 <h5 className="text-indigo-900 font-bold flex items-center gap-2 mb-2">
                    <Sparkles size={18} className="text-indigo-600"/> Analisis Keuangan AI ✨
                 </h5>
                 {aiAnalysis ? (
                   <div className="text-xs text-indigo-800 bg-white p-3 rounded-lg shadow-inner max-h-40 overflow-y-auto whitespace-pre-line">
                      {aiAnalysis}
                   </div>
                 ) : (
                   <p className="text-xs text-indigo-600">Tekan tombol di bawah untuk mendapatkan analisis mendalam dari Gemini AI.</p>
                 )}
                 <div className="mt-3 flex gap-2">
                    <button 
                      onClick={handleAiFinancialAnalysis} 
                      disabled={aiLoading}
                      className="bg-indigo-600 text-white px-4 py-2 rounded-lg text-xs font-bold hover:bg-indigo-700 flex items-center gap-2 disabled:opacity-50"
                    >
                      {aiLoading ? <Loader2 size={14} className="animate-spin" /> : <Sparkles size={14}/>}
                      {aiAnalysis ? "Perbarui Analisis ✨" : "Jalankan Analisis ✨"}
                    </button>
                    {aiAnalysis && (
                       <button onClick={() => setAiAnalysis("")} className="text-indigo-600 text-xs font-medium hover:underline">Hapus</button>
                    )}
                 </div>
              </div>

              <div className="flex gap-2 w-full md:w-auto">
                <button onClick={() => window.print()} className="flex-1 md:flex-none flex items-center justify-center gap-2 px-4 py-2 bg-gray-800 text-white rounded-lg text-sm hover:bg-black transition">
                  <Printer size={16} /> Cetak
                </button>
              </div>
            </div>

            <div className="bg-white p-8 rounded-2xl shadow-md border print:shadow-none print:border-none">
              <div className="text-center mb-10">
                <h3 className="text-2xl font-black uppercase text-blue-900 tracking-wider">Laporan Keuangan Karang Taruna</h3>
                <p className="text-gray-500">Update per: {new Date().toLocaleDateString('id-ID', { month: 'long', year: 'numeric' })}</p>
                <div className="w-24 h-1 bg-blue-600 mx-auto mt-4 rounded-full"></div>
              </div>

              <section className="mb-10">
                <h5 className="bg-gray-100 p-2 font-bold mb-4 rounded border-l-4 border-blue-600 uppercase text-sm tracking-wide">I. Laba Rugi (Ringkasan)</h5>
                <div className="space-y-3 px-4">
                  <div className="flex justify-between border-b pb-1">
                    <span className="text-sm font-medium">Total Pendapatan (Kas Masuk)</span>
                    <span className="text-green-600 font-bold">{formatIDR(totalIn)}</span>
                  </div>
                  <div className="flex justify-between border-b pb-1">
                    <span className="text-sm font-medium">Total Pengeluaran (Kas Keluar)</span>
                    <span className="text-red-600 font-bold">({formatIDR(totalOut)})</span>
                  </div>
                  <div className="flex justify-between font-black text-lg pt-2 bg-gray-50 p-3 rounded-lg">
                    <span>SALDO BERSIH</span>
                    <span className={balance >= 0 ? 'text-blue-700' : 'text-red-700'}>{formatIDR(balance)}</span>
                  </div>
                </div>
              </section>

              <section className="mb-10">
                <h5 className="bg-gray-100 p-2 font-bold mb-4 rounded border-l-4 border-blue-600 uppercase text-sm tracking-wide">II. Posisi Aset (Neraca Singkat)</h5>
                <div className="grid grid-cols-2 gap-10 px-4">
                   <div>
                     <p className="text-[10px] font-black text-gray-400 mb-2 uppercase">Aset Lancar</p>
                     <div className="flex justify-between border-b pb-1 text-sm"><span>Kas di Tangan</span><span>{formatIDR(balance)}</span></div>
                     <div className="flex justify-between font-bold pt-2 text-sm"><span>TOTAL ASET</span><span>{formatIDR(balance)}</span></div>
                   </div>
                   <div>
                     <p className="text-[10px] font-black text-gray-400 mb-2 uppercase">Ekuitas</p>
                     <div className="flex justify-between border-b pb-1 text-sm"><span>Modal Organisasi</span><span>{formatIDR(balance)}</span></div>
                     <div className="flex justify-between font-bold pt-2 text-sm"><span>TOTAL PASIVA</span><span>{formatIDR(balance)}</span></div>
                   </div>
                </div>
              </section>

              {aiAnalysis && (
                <section className="mb-10 bg-indigo-50 p-6 rounded-xl border border-indigo-200 print:hidden">
                   <h5 className="font-bold text-indigo-900 mb-2 flex items-center gap-2">
                     <Sparkles size={16}/> Lampiran Catatan Strategis AI
                   </h5>
                   <p className="text-sm text-indigo-800 leading-relaxed whitespace-pre-line">
                     {aiAnalysis}
                   </p>
                </section>
              )}

              <div className="mt-20 flex justify-between px-10 text-center">
                <div className="w-40">
                  <p className="text-sm mb-16 italic text-gray-400">(Tanda Tangan)</p>
                  <p className="font-bold border-b border-black">Bendahara</p>
                </div>
                <div className="w-40">
                  <p className="text-sm mb-16 italic text-gray-400">(Tanda Tangan)</p>
                  <p className="font-bold border-b border-black">Ketua Umum</p>
                </div>
              </div>
            </div>
          </div>
        )}
      </main>
    </div>
  );
};

export default App;
