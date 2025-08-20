import React, { useEffect, useMemo, useRef, useState } from "react"; import { motion, AnimatePresence } from "framer-motion"; import { Search, MessageCircle, CheckCircle2, XCircle, AlertCircle, Bell, Phone, MapPin, Filter, IndianRupee, Wallet, Upload, Users, Settings, Store, ChevronRight, ChevronDown, Clock, Plus, Trash2, RefreshCcw, } from "lucide-react";

/**

Servezy – Single‑File React App (Test Version)


---

This single-file app implements a functional, test-ready Servezy website:

Customer view: search, filter, list & view vendors, WhatsApp chat, promos banner


Vendor view: register/list service, upload images, manual address entry,


WhatsApp integration, application status tracking, plan payments, transactions

Admin view: approve/reject vendors, send notifications (to vendors & customers),


manage featured/promoted vendors

LocalStorage persistence — no backend required


Notes

Image uploads are stored as data URLs in localStorage for demo use only.


Payments are simulated (UPI/Razorpay) and logged to a local transaction ledger.


Designed for quick testing; you can split into modules later. */



// ---------- Utilities ---------- const uid = () => Math.random().toString(36).slice(2, 9); const todayISO = () => new Date().toISOString(); const currencyINR = (n) => new Intl.NumberFormat("en-IN", { style: "currency", currency: "INR" }).format(n);

const STORAGE_KEYS = { vendors: "servezy.vendors", notifications: "servezy.notifications", promos: "servezy.promos", transactions: "servezy.transactions", featured: "servezy.featured", };

const loadLS = (k, fallback) => { try { const v = localStorage.getItem(k); return v ? JSON.parse(v) : fallback; } catch { return fallback; } }; const saveLS = (k, v) => localStorage.setItem(k, JSON.stringify(v));

// ---------- Types (lightweight JSDoc) ---------- /** @typedef {{

id: string,

ownerName: string,

phone: string,

whatsapp: string,

serviceName: string,

category: string,

description: string,

images: string[], // data URLs or links

address: { line1: string, area: string, city: string, pincode: string },

rating: number,

createdAt: string,

status: "pending" | "approved" | "rejected",

statusNote?: string,

plan?: "monthly" | "quarterly" | "yearly",

lastPaymentAt?: string

}} Vendor */


/** @typedef {{ id: string, audience: "all"|"vendors"|"customers", message: string, createdAt: string }} Notice / /* @typedef {{ id: string, vendorId?: string, method: "UPI"|"Razorpay", amount: number, createdAt: string, note?: string }} Txn */

// ---------- Sample seed (only on first load) ---------- function useSeed() { useEffect(() => { if (!loadLS(STORAGE_KEYS.vendors)) { const seed = [ { id: uid(), ownerName: "Rahul Kumar", phone: "9000000001", whatsapp: "9000000001", serviceName: "SparkPro Electricians", category: "Electrician", description: "24/7 electrical repairs, wiring, meter board, fan & light installation.", images: [], address: { line1: "H.No 12-3/1, Gandhi Nagar", area: "Near Bus Stand", city: "Chennur", pincode: "504201" }, rating: 4.6, createdAt: todayISO(), status: "approved", statusNote: "Auto-approved seed", plan: "monthly", lastPaymentAt: todayISO(), }, { id: uid(), ownerName: "S. Priya", phone: "9000000002", whatsapp: "9000000002", serviceName: "CleanNest Home Cleaning", category: "Home Cleaning", description: "Deep cleaning, kitchen & bathroom detailing, sofa shampooing.", images: [], address: { line1: "3-45/7, Vivekananda Colony", area: "Opp. Govt Jr College", city: "Mancherial", pincode: "504208" }, rating: 4.8, createdAt: todayISO(), status: "approved", plan: "quarterly", lastPaymentAt: todayISO(), }, ]; saveLS(STORAGE_KEYS.vendors, seed); } if (!loadLS(STORAGE_KEYS.notifications)) { const n = [ { id: uid(), audience: "all", message: "Welcome to Servezy! Local pros you can trust.", createdAt: todayISO() }, ]; saveLS(STORAGE_KEYS.notifications, n); } if (!loadLS(STORAGE_KEYS.promos)) { saveLS(STORAGE_KEYS.promos, [ { id: uid(), text: "🎉 Launch Offer: 20% off on first service booking!" }, { id: uid(), text: "🧹 Festive Deep Clean packages now available." }, ]); } if (!loadLS(STORAGE_KEYS.transactions)) { saveLS(STORAGE_KEYS.transactions, []); } if (!loadLS(STORAGE_KEYS.featured)) { saveLS(STORAGE_KEYS.featured, []); } }, []); }

// ---------- UI helpers ---------- function Pill({ children }) { return ( <span className="inline-flex items-center rounded-full px-3 py-1 text-xs font-medium bg-gray-100">{children}</span> ); }

function Section({ title, subtitle, action, children }) { return ( <div className="bg-white rounded-2xl shadow p-6 mb-6"> <div className="flex items-start justify-between gap-4 mb-4"> <div> <h2 className="text-xl font-semibold">{title}</h2> {subtitle && <p className="text-sm text-gray-500">{subtitle}</p>} </div> {action} </div> <div>{children}</div> </div> ); }

function Toolbar({ children }) { return <div className="flex flex-wrap items-center gap-3 mb-4">{children}</div>; }

function TextInput({ label, ...props }) { return ( <label className="block"> <span className="text-sm text-gray-600">{label}</span> <input {...props} className={mt-1 w-full rounded-xl border p-2 outline-none focus:ring focus:ring-indigo-200 ${props.className||''}} /> </label> ); }

function TextArea({ label, ...props }) { return ( <label className="block"> <span className="text-sm text-gray-600">{label}</span> <textarea {...props} className={mt-1 w-full rounded-xl border p-2 outline-none focus:ring focus:ring-indigo-200 ${props.className||''}} /> </label> ); }

function Select({ label, children, ...props }) { return ( <label className="block"> <span className="text-sm text-gray-600">{label}</span> <select {...props} className={mt-1 w-full rounded-xl border p-2 bg-white outline-none focus:ring focus:ring-indigo-200 ${props.className||''}}>{children}</select> </label> ); }

function Button({ variant = "primary", className = "", ...props }) { const base = "rounded-2xl px-4 py-2 font-medium shadow-sm transition active:scale-95"; const styles = { primary: "bg-indigo-600 text-white hover:bg-indigo-700", ghost: "bg-transparent hover:bg-gray-100", outline: "border border-gray-300 hover:bg-gray-50", danger: "bg-rose-600 text-white hover:bg-rose-700", }; return <button {...props} className={${base} ${styles[variant]} ${className}} />; }

// ---------- Main App ---------- export default function ServezyApp() { useSeed();

const [tab, setTab] = useState("customer"); // customer | vendor | admin const [q, setQ] = useState(""); const [category, setCategory] = useState(""); const [city, setCity] = useState(""); const [minRating, setMinRating] = useState(0); const [vendors, setVendors] = useState(() => loadLS(STORAGE_KEYS.vendors, [])); const [notices, setNotices] = useState(() => loadLS(STORAGE_KEYS.notifications, [])); const [promos, setPromos] = useState(() => loadLS(STORAGE_KEYS.promos, [])); const [txns, setTxns] = useState(() => loadLS(STORAGE_KEYS.transactions, [])); const [featured, setFeatured] = useState(() => loadLS(STORAGE_KEYS.featured, []));

useEffect(() => saveLS(STORAGE_KEYS.vendors, vendors), [vendors]); useEffect(() => saveLS(STORAGE_KEYS.notifications, notices), [notices]); useEffect(() => saveLS(STORAGE_KEYS.promos, promos), [promos]); useEffect(() => saveLS(STORAGE_KEYS.transactions, txns), [txns]); useEffect(() => saveLS(STORAGE_KEYS.featured, featured), [featured]);

const categories = useMemo(() => [ "Electrician", "Plumber", "Home Cleaning", "Carpenter", "AC Service", "Pest Control", "Appliance Repair", "Painter", "Beautician", ], []);

const cities = useMemo(() => ["Chennur", "Mancherial", "Karimnagar", "Warangal", "Hyderabad"], []);

const filtered = useMemo(() => { const lc = q.toLowerCase(); return vendors.filter(v => (!lc || v.serviceName.toLowerCase().includes(lc) || v.description.toLowerCase().includes(lc) || v.ownerName.toLowerCase().includes(lc)) && (!category || v.category === category) && (!city || v.address.city === city) && (v.rating >= minRating) && v.status === "approved" ).sort((a,b) => (featured.includes(b.id) - featured.includes(a.id)) || b.rating - a.rating); }, [vendors, q, category, city, minRating, featured]);

return ( <div className="min-h-screen bg-gray-50"> <Header tab={tab} setTab={setTab} /> <main className="max-w-6xl mx-auto p-4 md:p-6"> <Announcements notices={notices} promos={promos} /> {tab === "customer" && ( <CustomerView
q={q} setQ={setQ}
category={category} setCategory={setCategory}
city={city} setCity={setCity}
minRating={minRating} setMinRating={setMinRating}
categories={categories} cities={cities}
vendors={filtered}
/> )} {tab === "vendor" && ( <VendorView vendors={vendors} setVendors={setVendors} categories={categories} cities={cities} txns={txns} setTxns={setTxns} /> )} {tab === "admin" && ( <AdminView vendors={vendors} setVendors={setVendors} notices={notices} setNotices={setNotices} promos={promos} setPromos={setPromos} txns={txns} featured={featured} setFeatured={setFeatured} /> )} </main> <Footer /> </div> ); }

// ---------- Header / Footer ---------- function Header({ tab, setTab }) { return ( <header className="bg-white border-b sticky top-0 z-30"> <div className="max-w-6xl mx-auto flex items-center justify-between p-4"> <div className="flex items-center gap-3"> <div className="h-10 w-10 rounded-2xl bg-indigo-600 text-white grid place-items-center font-bold">S</div> <div> <h1 className="text-lg font-bold">Servezy</h1> <p className="text-xs text-gray-500">Local Services • Telangana</p> </div> </div> <nav className="flex gap-2"> {[ { key: "customer", label: "Customer", icon: <Users className="h-4 w-4" /> }, { key: "vendor", label: "Vendor", icon: <Store className="h-4 w-4" /> }, { key: "admin", label: "Admin", icon: <Settings className="h-4 w-4" /> }, ].map((t) => ( <Button key={t.key} variant={tab === t.key ? "primary" : "outline"} onClick={() => setTab(t.key)} className="flex items-center gap-2" > {t.icon} {t.label} </Button> ))} </nav> </div> </header> ); }

function Footer() { return ( <footer className="py-10 text-center text-sm text-gray-500"> Built with ❤ for quick testing — Split into real backend later. </footer> ); }

// ---------- Announcements ---------- function Announcements({ notices, promos }) { return ( <div className="mb-6"> <div className="flex flex-col gap-3"> <div className="bg-indigo-50 border border-indigo-100 text-indigo-900 rounded-2xl p-3 flex items-center gap-2"> <Bell className="h-4 w-4" /> <div className="text-sm"> <strong>Announcements:</strong> <span className="ml-2"> {notices.map(n => n.message).join(" • ")} </span> </div> </div> <div className="bg-emerald-50 border border-emerald-100 text-emerald-900 rounded-2xl p-3 text-sm"> <strong>Promos:</strong> {promos.map(p => p.text).join(" • ")} </div> </div> </div> ); }

// ---------- Customer View ---------- function CustomerView({ q, setQ, category, setCategory, city, setCity, minRating, setMinRating, categories, cities, vendors }) { return ( <div> <Section title="Find local services" subtitle="Search, filter and chat on WhatsApp to book"> <Toolbar> <div className="relative flex-1 min-w-[220px]"> <input value={q} onChange={(e) => setQ(e.target.value)} placeholder="Search electricians, cleaning..." className="w-full rounded-2xl border pl-10 pr-3 py-2 focus:ring focus:ring-indigo-200 outline-none" /> <Search className="h-4 w-4 absolute left-3 top-2.5 text-gray-400" /> </div> <Select label="Category" value={category} onChange={e=>setCategory(e.target.value)} className="min-w-[180px]"> <option value="">All</option> {categories.map(c => <option key={c} value={c}>{c}</option>)} </Select> <Select label="City" value={city} onChange={e=>setCity(e.target.value)} className="min-w-[160px]"> <option value="">Any</option> {cities.map(c => <option key={c} value={c}>{c}</option>)} </Select> <Select label="Min Rating" value={String(minRating)} onChange={e=>setMinRating(Number(e.target.value))} className="min-w-[140px]"> {[0,3,3.5,4,4.5].map(r => <option key={r} value={r}>{r === 0 ? "Any" : ${r}+}</option>)} </Select> </Toolbar> <VendorGrid vendors={vendors} /> </Section> <Section title="Become a vendor" subtitle="List your service and get customers on WhatsApp"> <div className="flex flex-col md:flex-row items-center justify-between gap-4"> <p className="text-gray-600">Register in minutes. Approval takes ~1 working day in production (instant in this demo).</p> <a href="#vendor" onClick={() => window.scrollTo({ top: 0 })} className="inline-flex items-center gap-2 text-indigo-600 font-medium">Go to Vendor Portal <ChevronRight className="h-4 w-4" /></a> </div> </Section> </div> ); }

function VendorGrid({ vendors }) { if (!vendors.length) return <p className="text-gray-500">No matching services yet.</p>; return ( <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-5"> {vendors.map(v => <VendorCard key={v.id} v={v} />)} </div> ); }

function VendorCard({ v }) { const cover = v.images?.[0] || https://images.unsplash.com/photo-1554647286-f365d7defc54?q=80&w=1200&auto=format&fit=crop; const waLink = https://wa.me/91${(v.whatsapp||"").replace(/\D/g,"")}?text=${encodeURIComponent("Hi "+v.serviceName+" — found you on Servezy. I need help with...")}; return ( <motion.div layout className="bg-white rounded-2xl shadow overflow-hidden border"> <div className="aspect-[16/9] bg-gray-100"> <img src={cover} alt={v.serviceName} className="w-full h-full object-cover" /> </div> <div className="p-4"> <div className="flex items-center justify-between"> <h3 className="font-semibold">{v.serviceName}</h3> <Pill>★ {v.rating.toFixed(1)}</Pill> </div> <p className="text-sm text-gray-600 line-clamp-2 mt-1">{v.description}</p> <div className="flex items-center gap-2 text-sm text-gray-600 mt-2"> <MapPin className="h-4 w-4" /> {v.address.city} • {v.category} </div> <div className="flex items-center gap-2 mt-3"> <a href={waLink} target="_blank" rel="noreferrer"> <Button className="flex items-center gap-2"><MessageCircle className="h-4 w-4" /> WhatsApp</Button> </a> <a href={tel:+91${v.phone}}> <Button variant="outline" className="flex items-center gap-2"><Phone className="h-4 w-4" /> Call</Button> </a> </div> </div> </motion.div> ); }

// ---------- Vendor View ---------- function VendorView({ vendors, setVendors, categories, cities, txns, setTxns }) { const [step, setStep] = useState("register"); // register | myapp | payments | txnhistory const [lookupPhone, setLookupPhone] = useState(""); const me = vendors.find(v => v.phone === lookupPhone);

return ( <div id="vendor"> <Section title="Vendor Portal" subtitle="Register, track status, manage payments"> <Toolbar> <Button variant={step === "register" ? "primary" : "outline"} onClick={()=>setStep("register")}>Register</Button> <Button variant={step === "myapp" ? "primary" : "outline"} onClick={()=>setStep("myapp")}>My Application</Button> <Button variant={step === "payments" ? "primary" : "outline"} onClick={()=>setStep("payments")}>Payments</Button> <Button variant={step === "txnhistory" ? "primary" : "outline"} onClick={()=>setStep("txnhistory")}>Transactions</Button> </Toolbar>

{step !== "register" && (
      <div className="mb-4 grid grid-cols-1 md:grid-cols-3 gap-3">
        <TextInput label="Enter your phone to fetch your application" value={lookupPhone} onChange={(e)=>setLookupPhone(e.target.value)} placeholder="e.g. 9000000001" />
        <div className="md:col-span-2 flex items-end gap-2">
          {me ? <Pill>Found: {me.serviceName} ({me.status})</Pill> : <Pill>No record yet</Pill>}
        </div>
      </div>
    )}

    {step === "register" && (
      <VendorRegisterForm categories={categories} cities={cities} onSubmit={(v) => setVendors(prev => [v, ...prev])} />
    )}
    {step === "myapp" && <VendorMyApp me={me} />}
    {step === "payments" && <VendorPayments me={me} setTxns={setTxns} />}
    {step === "txnhistory" && <TxnHistory txns={txns} me={me} />}
  </Section>
</div>

); }

function VendorRegisterForm({ categories, cities, onSubmit }) { const [form, setForm] = useState({ ownerName: "", phone: "", whatsapp: "", serviceName: "", category: categories[0], description: "", images: [], address: { line1: "", area: "", city: cities[0], pincode: "" }, });

const fileInputRef = useRef(null);

const handleImage = async (file) => { const reader = new FileReader(); return new Promise((res) => { reader.onload = () => res(reader.result); reader.readAsDataURL(file); }); };

const addImages = async (files) => { const arr = []; for (const f of files) { const url = await handleImage(f); arr.push(url); } setForm((s) => ({ ...s, images: [...s.images, ...arr].slice(0, 6) })); };

const submit = (e) => { e.preventDefault(); const v = { id: uid(), createdAt: todayISO(), rating: 4 + Math.random(), status: "pending", ...form, }; onSubmit(v); window.scrollTo({ top: 0, behavior: "smooth" }); alert("Application submitted! Track status in 'My Application'. (Auto-approval is available in Admin view.)"); setForm({ ownerName: "", phone: "", whatsapp: "", serviceName: "", category: categories[0], description: "", images: [], address: { line1: "", area: "", city: cities[0], pincode: "" } }); };

return ( <form onSubmit={submit} className="grid grid-cols-1 md:grid-cols-2 gap-4"> <TextInput label="Owner Name" value={form.ownerName} onChange={(e)=>setForm({...form, ownerName:e.target.value})} required /> <TextInput label="Phone" value={form.phone} onChange={(e)=>setForm({...form, phone:e.target.value})} required /> <TextInput label="WhatsApp" value={form.whatsapp} onChange={(e)=>setForm({...form, whatsapp:e.target.value})} required /> <TextInput label="Service/Shop Name" value={form.serviceName} onChange={(e)=>setForm({...form, serviceName:e.target.value})} required /> <Select label="Category" value={form.category} onChange={(e)=>setForm({...form, category:e.target.value})}> {categories.map(c => <option key={c} value={c}>{c}</option>)} </Select> <TextArea label="Description" value={form.description} onChange={(e)=>setForm({...form, description:e.target.value})} className="md:col-span-2" rows={4} />

<div className="md:col-span-2 grid grid-cols-1 md:grid-cols-4 gap-3">
    <TextInput label="Address Line" value={form.address.line1} onChange={(e)=>setForm({...form, address: { ...form.address, line1:e.target.value }})} />
    <TextInput label="Area/Landmark" value={form.address.area} onChange={(e)=>setForm({...form, address: { ...form.address, area:e.target.value }})} />
    <Select label="City" value={form.address.city} onChange={(e)=>setForm({...form, address: { ...form.address, city:e.target.value }})}>
      {["Chennur","Mancherial","Karimnagar","Warangal","Hyderabad"].map(c => <option key={c}>{c}</option>)}
    </Select>
    <TextInput label="Pincode" value={form.address.pincode} onChange={(e)=>setForm({...form, address: { ...form.address, pincode:e.target.value }})} />
  </div>

  <div className="md:col-span-2">
    <span className="text-sm text-gray-600">Upload Photos (max 6)</span>
    <div className="mt-2 flex flex-wrap gap-3 items-center">
      <input type="file" accept="image/*" multiple hidden ref={fileInputRef} onChange={(e)=>{ if(e.target.files?.length) addImages(e.target.files); e.target.value=''; }} />
      <Button type="button" variant="outline" className="flex items-center gap-2" onClick={()=>fileInputRef.current?.click()}>
        <Upload className="h-4 w-4" /> Upload
      </Button>
      <div className="flex gap-3 flex-wrap">
        {form.images.map((src, i) => (
          <div key={i} className="relative h-20 w-28 rounded-xl overflow-hidden border">
            <img src={src} alt="upload" className="w-full h-full object-cover" />
          </div>
        ))}
      </div>
    </div>
  </div>

  <div className="md:col-span-2 flex items-center justify-between">
    <div className="text-sm text-gray-500 flex items-center gap-2"><AlertCircle className="h-4 w-4" /> By submitting, you agree to Servezy listing guidelines.</div>
    <Button type="submit" className="">Submit Application</Button>
  </div>
</form>

); }

function StatusBadge({ status }) { const map = { pending: { label: "Pending", cls: "bg-amber-100 text-amber-800" }, approved: { label: "Approved", cls: "bg-emerald-100 text-emerald-800" }, rejected: { label: "Rejected", cls: "bg-rose-100 text-rose-800" }, }; const s = map[status]; return <span className={px-2 py-1 rounded-full text-xs ${s.cls}}>{s.label}</span>; }

function VendorMyApp({ me }) { if (!me) return <p className="text-gray-500">Enter your phone above to see your application.</p>; return ( <div className="grid grid-cols-1 md:grid-cols-3 gap-4"> <div className="md:col-span-2 bg-white rounded-2xl shadow p-4"> <div className="flex items-center justify-between gap-3"> <div> <h3 className="text-lg font-semibold">{me.serviceName}</h3> <div className="text-sm text-gray-500">Owner: {me.ownerName} • <a className="underline" href={tel:+91${me.phone}}>+91 {me.phone}</a></div> </div> <StatusBadge status={me.status} /> </div> {me.statusNote && <p className="mt-2 text-sm text-gray-600">Note: {me.statusNote}</p>} <div className="mt-3 flex gap-2 flex-wrap"> <Pill>{me.category}</Pill> <Pill><MapPin className="h-3 w-3 inline mr-1" /> {me.address.city}</Pill> {me.plan && <Pill>Plan: {me.plan}</Pill>} </div> <div className="mt-4 grid grid-cols-2 md:grid-cols-3 gap-3"> {(me.images?.length ? me.images : [ "https://images.unsplash.com/photo-1582719478250-c89cae4dc85b?q=80&w=1200&auto=format&fit=crop", "https://images.unsplash.com/photo-1581092918367-3a673497d34a?q=80&w=1200&auto=format&fit=crop", ]).map((src, i) => ( <img key={i} src={src} alt="img" className="h-28 w-full object-cover rounded-xl border" /> ))} </div> </div> <div className="bg-white rounded-2xl shadow p-4"> <h4 className="font-semibold mb-2">Quick Links</h4> <div className="flex flex-col gap-2"> <a className="underline" target="_blank" rel="noreferrer" href={https://wa.me/91${me.whatsapp}?text=${encodeURIComponent("Hi, customer from Servezy!")}}>Open WhatsApp Chat</a> <a className="underline" href={tel:+91${me.phone}}>Call Customer Hotline</a> </div> <div className="mt-4 text-sm text-gray-600"> Last payment: {me.lastPaymentAt ? new Date(me.lastPaymentAt).toLocaleString() : "—"} </div> </div> </div> ); }

function VendorPayments({ me, setTxns }) { const [plan, setPlan] = useState("monthly"); const price = plan === "monthly" ? 1000 : plan === "quarterly" ? 2500 : 9000; if (!me) return <p className="text-gray-500">Enter your phone above to manage payments.</p>;

const pay = (method) => { const t = { id: uid(), vendorId: me.id, method, amount: price, createdAt: todayISO(), note: ${plan} plan }; setTxns(prev => [t, ...prev]); // store last payment date on vendor in LS const vs = loadLS(STORAGE_KEYS.vendors, []); const idx = vs.findIndex(v => v.id === me.id); if (idx >= 0) { vs[idx].plan = plan; vs[idx].lastPaymentAt = t.createdAt; saveLS(STORAGE_KEYS.vendors, vs); } alert(Payment recorded (${method}). This is a demo – integrate real gateway later.); };

return ( <div className="grid grid-cols-1 md:grid-cols-2 gap-4"> <div className="bg-white rounded-2xl shadow p-4"> <h4 className="font-semibold mb-2">Choose Plan</h4> <Select label="Billing Plan" value={plan} onChange={(e)=>setPlan(e.target.value)}> <option value="monthly">Monthly – {currencyINR(1000)}</option> <option value="quarterly">Quarterly – {currencyINR(2500)}</option> <option value="yearly">Yearly – {currencyINR(9000)}</option> </Select> <div className="mt-4 flex items-center justify-between"> <div className="text-sm text-gray-600">Amount due</div> <div className="text-lg font-semibold">{currencyINR(price)}</div> </div> <div className="mt-4 grid grid-cols-2 gap-3"> <Button onClick={()=>pay("UPI")} className="flex items-center gap-2"><Wallet className="h-4 w-4" /> Pay via UPI</Button> <Button onClick={()=>pay("Razorpay")} className="flex items-center gap-2" variant="outline"><IndianRupee className="h-4 w-4" /> Pay via Razorpay</Button> </div> <p className="text-xs text-gray-500 mt-3">Demo only: these buttons just record a transaction locally.</p> </div> <div className="bg-white rounded-2xl shadow p-4"> <h4 className="font-semibold mb-2">How billing works</h4> <ul className="list-disc pl-5 text-sm text-gray-600 space-y-1"> <li>Paid vendors appear higher and get promo slots.</li> <li>Invoices & GST calculations can be added in production.</li> <li>We support UPI and Razorpay integration (to be wired server-side).</li> </ul> </div> </div> ); }

function TxnHistory({ txns, me }) { const rows = me ? txns.filter(t => t.vendorId === me.id) : txns; if (!rows.length) return <p className="text-gray-500">No transactions yet.</p>; return ( <div className="overflow-auto bg-white rounded-2xl shadow"> <table className="w-full text-sm"> <thead className="bg-gray-50"> <tr> <th className="text-left p-3">Date</th> <th className="text-left p-3">Vendor</th> <th className="text-left p-3">Method</th> <th className="text-left p-3">Amount</th> <th className="text-left p-3">Note</th> </tr> </thead> <tbody> {rows.map(r => ( <tr key={r.id} className="border-t"> <td className="p-3">{new Date(r.createdAt).toLocaleString()}</td> <td className="p-3">{r.vendorId || "—"}</td> <td className="p-3">{r.method}</td> <td className="p-3">{currencyINR(r.amount)}</td> <td className="p-3">{r.note || ""}</td> </tr> ))} </tbody> </table> </div> ); }

// ---------- Admin View ---------- function AdminView({ vendors, setVendors, notices, setNotices, promos, setPromos, txns, featured, setFeatured }) { const [showPending, setShowPending] = useState(true); const [noticeText, setNoticeText] = useState(""); const [promoText, setPromoText] = useState("");

const approve = (id) => setVendors(prev => prev.map(v => v.id === id ? { ...v, status: "approved", statusNote: "Approved by admin" } : v)); const reject = (id) => setVendors(prev => prev.map(v => v.id === id ? { ...v, status: "rejected", statusNote: "Does not meet guidelines" } : v)); const remove = (id) => setVendors(prev => prev.filter(v => v.id !== id));

const toggleFeatured = (id) => setFeatured(prev => prev.includes(id) ? prev.filter(x => x !== id) : [...prev, id]);

const pending = vendors.filter(v => v.status === "pending");

return ( <div> <Section title="Approvals" subtitle="Review new vendor applications"> <Toolbar> <label className="flex items-center gap-2 text-sm"> <input type="checkbox" checked={showPending} onChange={(e)=>setShowPending(e.target.checked)} /> Show only pending </label> </Toolbar> <div className="grid grid-cols-1 md:grid-cols-2 gap-4"> {(showPending ? pending : vendors).map(v => ( <div key={v.id} className="bg-white rounded-2xl shadow p-4 border"> <div className="flex items-center justify-between"> <div> <div className="font-semibold">{v.serviceName} <span className="text-gray-400 font-normal">• {v.category}</span></div> <div className="text-sm text-gray-600">Owner: {v.ownerName} • +91 {v.phone}</div> </div> <StatusBadge status={v.status} /> </div> <p className="text-sm text-gray-600 mt-2 line-clamp-2">{v.description}</p> <div className="mt-3 flex items-center gap-2 flex-wrap"> <Pill><MapPin className="h-3 w-3 inline mr-1" /> {v.address.city}</Pill> <Pill>Rating {v.rating.toFixed(1)}</Pill> <Pill>ID {v.id}</Pill> <Pill>{new Date(v.createdAt).toLocaleDateString()}</Pill> </div> <div className="mt-3 flex flex-wrap gap-2"> <Button onClick={()=>approve(v.id)} className="flex items-center gap-2"><CheckCircle2 className="h-4 w-4" /> Approve</Button> <Button onClick={()=>reject(v.id)} variant="outline" className="flex items-center gap-2"><XCircle className="h-4 w-4" /> Reject</Button> <Button onClick={()=>toggleFeatured(v.id)} variant="ghost" className="flex items-center gap-2">{featured.includes(v.id) ? "Unfeature" : "Feature"}</Button> <Button onClick={()=>remove(v.id)} variant="danger" className="flex items-center gap-2"><Trash2 className="h-4 w-4" /> Delete</Button> </div> </div> ))} </div> </Section>

<Section title="Notifications" subtitle="Send announcements to vendors and customers">
    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
      <div className="md:col-span-2">
        <div className="flex gap-2">
          <input value={noticeText} onChange={(e)=>setNoticeText(e.target.value)} placeholder="Write an announcement…" className="flex-1 rounded-2xl border px-3 py-2 outline-none" />
          <Button onClick={() => { if(!noticeText.trim()) return; const n = { id: uid(), audience: "all", message: noticeText.trim(), createdAt: todayISO() }; setNotices(prev => [n, ...prev]); setNoticeText(""); }}>Send</Button>
        </div>
        <ul className="mt-4 divide-y bg-white rounded-2xl border">
          {notices.map(n => (
            <li key={n.id} className="p-3 text-sm flex items-center justify-between">
              <div>
                <div>{n.message}</div>
                <div className="text-xs text-gray-500">{new Date(n.createdAt).toLocaleString()}</div>
              </div>
              <Pill>{n.audience}</Pill>
            </li>
          ))}
        </ul>
      </div>
      <div className="bg-white rounded-2xl shadow p-4">
        <h4 className="font-semibold mb-2">Promotions banner</h4>
        <div className="flex gap-2">
          <input value={promoText} onChange={(e)=>setPromoText(e.target.value)} placeholder="Add a promo…" className="flex-1 rounded-2xl border px-3 py-2 outline-none" />
          <Button variant="outline" onClick={() => { if(!promoText.trim()) return; setPromos(prev => [{ id: uid(), text: promoText.trim() }, ...prev]); setPromoText(""); }}>Add</Button>
        </div>
        <ul className="mt-3 space-y-2">
          {promos.map(p => (
            <li key={p.id} className="text-sm bg-gray-50 rounded-xl p-2 border">{p.text}</li>
          ))}
        </ul>
      </div>
    </div>
  </Section>

  <Section title="Transactions" subtitle="All recorded payments (demo)">
    <TxnHistory txns={txns} />
  </Section>
</div>

); }

