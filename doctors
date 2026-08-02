/**
 * DocSlot — Combined Worker (Public Site + Admin Panel)
 *
 * KV binding used: KV_BINDING (already set in your Cloudflare dashboard)
 */

// ===== EDIT YOUR PAYMENT DETAILS HERE =====
const PAYMENT_CONFIG = {
  upiId: "8677004540@ybl",
  upiName: "Pkfuturegkgs",
  paypalEmail: "your-paypal@email.com",
  bankDetails: "Bank: Your Bank Name, A/C: XXXXXXXXX, IFSC/SWIFT: XXXXXXX",
  monthlyFeeINR: 299,
  monthlyFeeUSD: 5,
  yearlyFeeINR: 2499,
  yearlyFeeUSD: 45,
};
// ============================================

const DAY_MS = 24 * 60 * 60 * 1000;

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname;

    // =========================================================
    // PUBLIC API
    // =========================================================

    if (path === "/api/doctors" && request.method === "GET") {
      const list = await env.KV_BINDING.list({ prefix: "doctor:" });
      const all = await Promise.all(
        list.keys.map(async (k) => {
          const v = await env.KV_BINDING.get(k.name);
          return v ? JSON.parse(v) : null;
        })
      );
      const now = Date.now();
      const visible = all.filter((d) => d && d.status === "approved" && d.expiresAt > now);
      return json(visible.sort((a, b) => b.createdAt - a.createdAt));
    }

    if (path === "/api/doctors" && request.method === "POST") {
      const body = await request.json().catch(() => null);
      if (!body) return json({ error: "Invalid data" }, 400);

      const required = ["name", "spec", "clinic", "city", "country", "fee", "currency", "phone", "plan", "txnId"];
      for (const field of required) {
        if (!body[field] || String(body[field]).trim() === "") {
          return json({ error: `${field} is required` }, 400);
        }
      }

      const id = crypto.randomUUID();
      const planDays = body.plan === "yearly" ? 365 : 30;
      const doctor = {
        id,
        name: body.name.trim(),
        spec: body.spec.trim(),
        exp: body.exp ? body.exp.trim() : "",
        clinic: body.clinic.trim(),
        city: body.city.trim(),
        country: body.country.trim(),
        fee: Number(body.fee),
        currency: body.currency.trim(),
        phone: body.phone.replace(/[^\d+]/g, ""),
        rating: 5.0,
        token: "T-" + Math.floor(Math.random() * 30 + 1),
        plan: body.plan,
        txnId: body.txnId.trim(),
        status: "pending",
        createdAt: Date.now(),
        approvedAt: null,
        expiresAt: null,
        planDays,
      };

      await env.KV_BINDING.put(`doctor:${id}`, JSON.stringify(doctor));
      return json({ success: true, doctor });
    }

    if (path === "/api/payment-config" && request.method === "GET") {
      return json(PAYMENT_CONFIG);
    }

    // =========================================================
    // ADMIN API (password protected)
    // =========================================================

    if (path === "/api/admin/login" && request.method === "POST") {
      const body = await request.json().catch(() => ({}));
      if (body.password === env.ADMIN_PASSWORD) return json({ success: true });
      return json({ success: false, error: "Wrong password" }, 401);
    }

    if (path.startsWith("/api/admin/") && path !== "/api/admin/login") {
      const authHeader = request.headers.get("x-admin-password") || "";
      if (authHeader !== env.ADMIN_PASSWORD) {
        return json({ error: "Unauthorized" }, 401);
      }
    }

    if (path === "/api/admin/listings" && request.method === "GET") {
      const list = await env.KV_BINDING.list({ prefix: "doctor:" });
      const all = await Promise.all(
        list.keys.map(async (k) => {
          const v = await env.KV_BINDING.get(k.name);
          return v ? JSON.parse(v) : null;
        })
      );
      return json(all.filter(Boolean).sort((a, b) => b.createdAt - a.createdAt));
    }

    if (path === "/api/admin/approve" && request.method === "POST") {
      const body = await request.json().catch(() => null);
      if (!body || !body.id) return json({ error: "Missing id" }, 400);
      const key = `doctor:${body.id}`;
      const existing = await env.KV_BINDING.get(key);
      if (!existing) return json({ error: "Not found" }, 404);
      const doctor = JSON.parse(existing);
      doctor.status = "approved";
      doctor.approvedAt = Date.now();
      doctor.expiresAt = Date.now() + (doctor.planDays || 30) * DAY_MS;
      await env.KV_BINDING.put(key, JSON.stringify(doctor));
      return json({ success: true, doctor });
    }

    if (path === "/api/admin/reject" && request.method === "POST") {
      const body = await request.json().catch(() => null);
      if (!body || !body.id) return json({ error: "Missing id" }, 400);
      const key = `doctor:${body.id}`;
      const existing = await env.KV_BINDING.get(key);
      if (!existing) return json({ error: "Not found" }, 404);
      const doctor = JSON.parse(existing);
      doctor.status = "rejected";
      await env.KV_BINDING.put(key, JSON.stringify(doctor));
      return json({ success: true });
    }

    if (path === "/api/admin/delete" && request.method === "POST") {
      const body = await request.json().catch(() => null);
      if (!body || !body.id) return json({ error: "Missing id" }, 400);
      await env.KV_BINDING.delete(`doctor:${body.id}`);
      return json({ success: true });
    }

    if (path === "/api/admin/renew" && request.method === "POST") {
      const body = await request.json().catch(() => null);
      if (!body || !body.id) return json({ error: "Missing id" }, 400);
      const key = `doctor:${body.id}`;
      const existing = await env.KV_BINDING.get(key);
      if (!existing) return json({ error: "Not found" }, 404);
      const doctor = JSON.parse(existing);
      const base = doctor.expiresAt && doctor.expiresAt > Date.now() ? doctor.expiresAt : Date.now();
      doctor.expiresAt = base + (doctor.planDays || 30) * DAY_MS;
      doctor.status = "approved";
      await env.KV_BINDING.put(key, JSON.stringify(doctor));
      return json({ success: true, doctor });
    }

    // =========================================================
    // FRONTEND PAGES
    // =========================================================

    if (path === "/admin" || path === "/admin/") {
      return new Response(ADMIN_HTML, { headers: { "content-type": "text/html;charset=UTF-8" } });
    }

    if (path === "/" || path === "") {
      return new Response(PUBLIC_HTML, { headers: { "content-type": "text/html;charset=UTF-8" } });
    }

    return new Response("Not found", { status: 404 });
  },
};

function json(data, status = 200) {
  return new Response(JSON.stringify(data), { status, headers: { "content-type": "application/json" } });
}

// =========================================================
// PUBLIC SITE HTML
// =========================================================
const PUBLIC_HTML = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DocSlot — Find & Book Doctors Worldwide</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,500;9..144,600;9..144,700&family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@500;600&display=swap" rel="stylesheet">
<style>
  :root{
    --bg:#F7F4EC; --card:#FFFDF8; --ink:#16211F;
    --teal:#0F3D3E; --teal-deep:#0A2B2C;
    --amber:#C9971B; --amber-soft:#F1E3BE; --line:#DCD5C2;
  }
  *{box-sizing:border-box; margin:0; padding:0;}
  body{ background:var(--bg); color:var(--ink); font-family:'IBM Plex Sans', sans-serif; -webkit-font-smoothing:antialiased; }
  h1,h2,h3,.display{ font-family:'Fraunces', serif; }
  .mono{ font-family:'IBM Plex Mono', monospace; }

  header{ background:var(--teal); color:#F7F4EC; padding:18px 20px; position:sticky; top:0; z-index:50; box-shadow:0 2px 12px rgba(10,43,44,0.15); }
  .header-row{ display:flex; align-items:center; justify-content:space-between; max-width:900px; margin:0 auto; flex-wrap:wrap; gap:10px; }
  .logo{ font-size:22px; font-weight:600; display:flex; align-items:center; gap:8px; }
  .logo .dot{ width:9px; height:9px; background:var(--amber); border-radius:50%; display:inline-block; }
  .tagline{ font-size:11px; color:#B9CFC9; margin-top:2px; }

  .tabs{ display:flex; gap:6px; }
  .tab-btn{ background:rgba(255,255,255,0.08); color:#E8F0EE; border:1px solid rgba(255,255,255,0.2); padding:8px 14px; border-radius:999px; font-size:12.5px; font-weight:600; cursor:pointer; }
  .tab-btn.active{ background:var(--amber); color:#3A2C05; border-color:var(--amber); }

  .view{ display:none; }
  .view.active{ display:block; }

  .hero{ max-width:900px; margin:0 auto; padding:36px 20px 20px; }
  .hero h1{ font-size:clamp(26px,6vw,38px); font-weight:600; line-height:1.15; color:var(--teal-deep); }
  .hero p{ margin-top:10px; font-size:15px; color:#4A5754; max-width:540px; }

  .search-wrap{ max-width:900px; margin:22px auto 0; padding:0 20px; display:flex; gap:10px; flex-wrap:wrap; }
  .search-box{ flex:1; min-width:200px; display:flex; align-items:center; background:var(--card); border:1.5px solid var(--line); border-radius:14px; padding:12px 16px; gap:10px; }
  .search-box input{ border:none; outline:none; background:transparent; font-size:15px; width:100%; color:var(--ink); }

  .chips{ display:flex; gap:8px; overflow-x:auto; padding:16px 20px 4px; max-width:900px; margin:0 auto; scrollbar-width:none; }
  .chips::-webkit-scrollbar{ display:none; }
  .chip{ flex-shrink:0; padding:8px 16px; border-radius:999px; border:1.5px solid var(--line); background:var(--card); font-size:13px; font-weight:500; color:#3E4A47; cursor:pointer; white-space:nowrap; }
  .chip.active{ background:var(--teal); border-color:var(--teal); color:#fff; }

  .results-line{ max-width:900px; margin:18px auto 0; padding:0 24px; font-size:12.5px; color:#7A8582; }

  .grid{ max-width:900px; margin:0 auto; padding:14px 20px 60px; display:grid; grid-template-columns:1fr; gap:16px; }
  @media(min-width:640px){ .grid{ grid-template-columns:1fr 1fr; } }

  .card{ background:var(--card); border:1px solid var(--line); border-radius:16px; padding:18px; animation:fadeUp 0.4s ease both; transition:transform 0.15s ease, box-shadow 0.15s ease; }
  .card:hover{ transform:translateY(-2px); box-shadow:0 8px 24px rgba(15,61,62,0.08); }
  @keyframes fadeUp{ from{ opacity:0; transform:translateY(8px);} to{ opacity:1; transform:translateY(0);} }

  .card-top{ display:flex; gap:12px; align-items:flex-start; }
  .avatar{ width:52px; height:52px; border-radius:50%; background:var(--teal); color:#fff; display:flex; align-items:center; justify-content:center; font-family:'Fraunces'; font-weight:600; font-size:18px; flex-shrink:0; }
  .doc-name{ font-size:16.5px; font-weight:600; color:var(--teal-deep); }
  .doc-spec{ font-size:12.5px; color:var(--amber); font-weight:600; margin-top:1px; }
  .doc-meta{ font-size:12px; color:#7A8582; margin-top:4px; }
  .stars{ font-size:11.5px; color:#C9971B; margin-top:5px; }

  .token{ margin-top:14px; border-top:1px dashed var(--line); padding-top:12px; display:flex; align-items:center; justify-content:space-between; }
  .token-left{ display:flex; align-items:center; gap:8px; }
  .token-num{ background:var(--amber-soft); color:#7A5A0A; font-family:'IBM Plex Mono'; font-weight:600; font-size:12px; padding:4px 9px; border-radius:6px; }
  .token-label{ font-size:11px; color:#8A948F; }
  .fee{ font-family:'IBM Plex Mono'; font-weight:600; font-size:14px; color:var(--teal-deep); }

  .book-btn{ margin-top:14px; width:100%; background:var(--teal); color:#fff; border:none; padding:12px; border-radius:10px; font-weight:600; font-size:14px; cursor:pointer; display:flex; align-items:center; justify-content:center; gap:8px; }
  .book-btn:hover{ background:var(--teal-deep); }

  .empty{ text-align:center; padding:60px 20px; color:#8A948F; grid-column:1/-1; }
  .empty .display{ font-size:20px; color:var(--teal-deep); margin-bottom:6px; }

  .form-wrap{ max-width:540px; margin:0 auto; padding:30px 20px 60px; }
  .form-card{ background:var(--card); border:1px solid var(--line); border-radius:18px; padding:24px; }
  .form-card h2{ font-size:20px; color:var(--teal-deep); }
  .form-card .sub{ font-size:13px; color:#7A8582; margin-top:6px; margin-bottom:18px; }
  .field{ margin-top:14px; }
  .field.two-col{ display:grid; grid-template-columns:1fr 1fr; gap:10px; }
  .field label{ font-size:12px; font-weight:600; color:#4A5754; display:block; margin-bottom:5px; }
  .field input, .field select{ width:100%; padding:11px 12px; border-radius:9px; border:1.5px solid var(--line); background:#fff; font-family:'IBM Plex Sans'; font-size:14.5px; color:var(--ink); }
  .field input:focus, .field select:focus{ outline:2px solid var(--teal); border-color:var(--teal); }
  .submit-btn{ margin-top:20px; width:100%; background:var(--amber); color:#3A2C05; border:none; padding:13px; border-radius:10px; font-weight:700; font-size:14.5px; cursor:pointer; }
  .submit-btn:hover{ filter:brightness(0.95); }
  .success-msg{ display:none; background:#E5F3E9; border:1px solid #9FCFAE; color:#1F5E32; padding:14px; border-radius:10px; font-size:13.5px; margin-top:14px; }
  .success-msg.show{ display:block; }
  .form-note{ font-size:11.5px; color:#9CA6A2; margin-top:16px; text-align:center; }

  .plan-toggle{ display:flex; gap:8px; margin-top:6px; }
  .plan-btn{ flex:1; padding:12px; border-radius:10px; border:1.5px solid var(--line); background:#fff; text-align:center; cursor:pointer; }
  .plan-btn.active{ border-color:var(--teal); background:var(--amber-soft); }
  .plan-btn .plan-name{ font-weight:700; font-size:13.5px; color:var(--teal-deep); }
  .plan-btn .plan-price{ font-size:12px; color:#7A8582; margin-top:2px; }

  .pay-box{ margin-top:16px; background:var(--amber-soft); border:1px solid #E4CE8E; border-radius:12px; padding:16px; font-size:13px; color:#5C4813; line-height:1.6; }
  .pay-box b{ color:#3A2C05; }

  .modal-overlay{ display:none; position:fixed; inset:0; background:rgba(10,20,20,0.55); z-index:100; align-items:flex-end; justify-content:center; }
  .modal-overlay.open{ display:flex; }
  .modal{ background:var(--bg); width:100%; max-width:480px; border-radius:20px 20px 0 0; padding:24px 22px 30px; animation:slideUp 0.25s ease both; max-height:88vh; overflow-y:auto; }
  @keyframes slideUp{ from{ transform:translateY(30px); opacity:0;} to{ transform:translateY(0); opacity:1;} }
  .modal h2{ font-size:20px; color:var(--teal-deep); }
  .modal .doc-spec{ margin-bottom:16px; }
  .modal-actions{ display:flex; gap:10px; margin-top:22px; }
  .modal-actions button{ flex:1; padding:13px; border-radius:10px; border:none; font-weight:600; font-size:14px; cursor:pointer; }
  .cancel-btn{ background:transparent; border:1.5px solid var(--line); color:#5A655F; }
  .confirm-btn{ background:#25D366; color:#fff; display:flex; align-items:center; justify-content:center; gap:7px; }

  footer{ text-align:center; padding:26px 20px 40px; font-size:11.5px; color:#9CA6A2; }
</style>
</head>
<body>

<header>
  <div class="header-row">
    <div>
      <div class="logo"><span class="dot"></span>DocSlot</div>
      <div class="tagline">Find doctors anywhere in the world</div>
    </div>
    <div class="tabs">
      <div class="tab-btn active" id="tabBrowse" onclick="switchTab('browse')">Find doctors</div>
      <div class="tab-btn" id="tabRegister" onclick="switchTab('register')">List your clinic</div>
    </div>
  </div>
</header>

<div class="view active" id="browseView">
  <div class="hero">
    <h1 class="display">Find a doctor nearby, book straight on WhatsApp</h1>
    <p>Search by specialty, country, or city — and book an appointment in one tap, anywhere in the world.</p>
  </div>
  <div class="search-wrap">
    <div class="search-box">
      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><line x1="21" y1="21" x2="16.65" y2="16.65"/></svg>
      <input type="text" id="searchInput" placeholder="Search by doctor name, specialty, city or country...">
    </div>
  </div>
  <div class="chips" id="chips"></div>
  <div class="results-line" id="resultsLine"></div>
  <div class="grid" id="grid"><div class="empty"><div class="display">Loading...</div></div></div>
</div>

<div class="view" id="registerView">
  <div class="form-wrap">
    <div class="form-card">
      <h2 class="display">List your clinic</h2>
      <div class="sub">Choose a plan, complete payment, and submit your details. Your listing goes live once we verify your payment (usually within a few hours).</div>

      <label style="font-size:12px;font-weight:600;color:#4A5754;">Choose your plan</label>
      <div class="plan-toggle">
        <div class="plan-btn active" id="planMonthly" onclick="selectPlan('monthly')">
          <div class="plan-name">Monthly</div>
          <div class="plan-price" id="monthlyPriceLabel">Loading...</div>
        </div>
        <div class="plan-btn" id="planYearly" onclick="selectPlan('yearly')">
          <div class="plan-name">Yearly</div>
          <div class="plan-price" id="yearlyPriceLabel">Loading...</div>
        </div>
      </div>

      <div class="pay-box" id="payBox">Loading payment details...</div>

      <div class="field"><label>Payment transaction / reference ID</label><input type="text" id="regTxnId" placeholder="Paste your UPI/PayPal/bank transaction ID"></div>

      <div class="field"><label>Doctor's name</label><input type="text" id="regName" placeholder="Dr. Full Name"></div>
      <div class="field"><label>Specialty</label><input type="text" id="regSpec" placeholder="e.g. General Physician, Dentist, Dermatologist"></div>
      <div class="field"><label>Experience</label><input type="text" id="regExp" placeholder="e.g. 10 years of experience"></div>
      <div class="field"><label>Clinic / Hospital name</label><input type="text" id="regClinic" placeholder="Clinic name"></div>
      <div class="field two-col">
        <div><label>City</label><input type="text" id="regCity" placeholder="City"></div>
        <div><label>Country</label><input type="text" id="regCountry" placeholder="Country"></div>
      </div>
      <div class="field two-col">
        <div><label>Consultation fee (for patients)</label><input type="number" id="regFee" placeholder="e.g. 30"></div>
        <div><label>Currency</label>
          <select id="regCurrency">
            <option value="USD">USD $</option>
            <option value="EUR">EUR €</option>
            <option value="GBP">GBP £</option>
            <option value="INR">INR ₹</option>
            <option value="AUD">AUD $</option>
            <option value="CAD">CAD $</option>
            <option value="AED">AED د.إ</option>
            <option value="SGD">SGD $</option>
            <option value="NGN">NGN ₦</option>
            <option value="ZAR">ZAR R</option>
            <option value="JPY">JPY ¥</option>
            <option value="OTHER">Other</option>
          </select>
        </div>
      </div>
      <div class="field"><label>WhatsApp number (with country code, e.g. +1, +44, +91)</label><input type="tel" id="regPhone" placeholder="+countrycode number"></div>

      <button class="submit-btn" onclick="submitDoctor()">Submit for review</button>
      <div class="success-msg" id="successMsg">✓ Submitted! Your listing will go live once your payment is verified.</div>
      <div class="form-note">Listings are reviewed manually before going live — this protects patients from fake entries.</div>
    </div>
  </div>
</div>

<div class="modal-overlay" id="modalOverlay">
  <div class="modal">
    <h2 class="display" id="modalDocName"></h2>
    <div class="doc-spec" id="modalDocSpec"></div>
    <div class="field"><label>Your name</label><input type="text" id="patientName" placeholder="Full name"></div>
    <div class="field"><label>Your phone number</label><input type="tel" id="patientPhone" placeholder="Your number"></div>
    <div class="field"><label>Appointment date</label><input type="date" id="apptDate"></div>
    <div class="field"><label>Time slot</label>
      <select id="apptSlot">
        <option>Morning 9:00 - 11:00</option>
        <option>Midday 11:00 - 1:00</option>
        <option>Afternoon 4:00 - 6:00</option>
        <option>Evening 6:00 - 8:00</option>
      </select>
    </div>
    <div class="modal-actions">
      <button class="cancel-btn" onclick="closeModal()">Cancel</button>
      <button class="confirm-btn" onclick="confirmBooking()">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M12.04 2C6.58 2 2.13 6.45 2.13 11.91c0 1.75.46 3.44 1.32 4.94L2 22l5.29-1.39a9.9 9.9 0 0 0 4.75 1.21h.01c5.46 0 9.9-4.45 9.9-9.91 0-2.65-1.03-5.14-2.9-7.01A9.82 9.82 0 0 0 12.04 2z"/></svg>
        Send on WhatsApp
      </button>
    </div>
  </div>
</div>

<footer>DocSlot — real doctors, real bookings, anywhere in the world.</footer>

<script>
  const CURRENCY_SYMBOLS = { USD:'$', EUR:'€', GBP:'£', INR:'₹', AUD:'$', CAD:'$', AED:'د.إ', SGD:'$', NGN:'₦', ZAR:'R', JPY:'¥', OTHER:'' };

  let doctors = [];
  let activeSpec = "All";
  let selectedDoc = null;
  let selectedPlan = "monthly";
  let paymentConfig = null;

  function switchTab(tab){
    document.getElementById('browseView').classList.toggle('active', tab==='browse');
    document.getElementById('registerView').classList.toggle('active', tab==='register');
    document.getElementById('tabBrowse').classList.toggle('active', tab==='browse');
    document.getElementById('tabRegister').classList.toggle('active', tab==='register');
  }

  function initials(name){ return name.replace("Dr. ","").split(" ").map(w=>w[0]).join("").slice(0,2).toUpperCase(); }

  async function loadDoctors(){
    try{
      const res = await fetch('/api/doctors');
      doctors = await res.json();
    }catch(e){ doctors = []; }
    renderChips();
    renderGrid();
  }

  async function loadPaymentConfig(){
    try{
      const res = await fetch('/api/payment-config');
      paymentConfig = await res.json();
    }catch(e){ paymentConfig = {}; }
    document.getElementById('monthlyPriceLabel').textContent = '₹' + paymentConfig.monthlyFeeINR + ' / $' + paymentConfig.monthlyFeeUSD;
    document.getElementById('yearlyPriceLabel').textContent = '₹' + paymentConfig.yearlyFeeINR + ' / $' + paymentConfig.yearlyFeeUSD;
    renderPayBox();
  }

  function selectPlan(plan){
    selectedPlan = plan;
    document.getElementById('planMonthly').classList.toggle('active', plan==='monthly');
    document.getElementById('planYearly').classList.toggle('active', plan==='yearly');
    renderPayBox();
  }

  function renderPayBox(){
    if(!paymentConfig) return;
    const inr = selectedPlan==='monthly' ? paymentConfig.monthlyFeeINR : paymentConfig.yearlyFeeINR;
    const usd = selectedPlan==='monthly' ? paymentConfig.monthlyFeeUSD : paymentConfig.yearlyFeeUSD;
    document.getElementById('payBox').innerHTML =
      '<b>Pay via UPI (India):</b> ' + paymentConfig.upiId + ' — ₹' + inr + '<br>' +
      '<b>Pay via PayPal (International):</b> ' + paymentConfig.paypalEmail + ' — $' + usd + '<br>' +
      '<b>Bank transfer:</b> ' + paymentConfig.bankDetails + '<br><br>' +
      'After paying, paste your transaction/reference ID below.';
  }

  function renderChips(){
    const specs = ["All", ...new Set(doctors.map(d=>d.spec))];
    document.getElementById("chips").innerHTML = specs.map(s =>
      \`<div class="chip \${s===activeSpec?'active':''}" onclick="setSpec('\${s.replace(/'/g,"\\\\'")}')">\${s}</div>\`
    ).join("");
  }

  function setSpec(s){ activeSpec = s; renderChips(); renderGrid(); }

  function getFiltered(){
    const query = document.getElementById("searchInput").value.toLowerCase().trim();
    return doctors.filter(d=>{
      const matchesSpec = activeSpec==="All" || d.spec===activeSpec;
      const haystack = (d.name+" "+d.spec+" "+d.city+" "+d.country).toLowerCase();
      const matchesQuery = !query || haystack.includes(query);
      return matchesSpec && matchesQuery;
    });
  }

  function renderGrid(){
    const filtered = getFiltered();
    document.getElementById("resultsLine").textContent = filtered.length + " doctor" + (filtered.length===1?'':'s') + " found";
    const grid = document.getElementById("grid");
    if(filtered.length===0){
      grid.innerHTML = '<div class="empty"><div class="display">No doctors yet</div><div>Be the first — register from the "List your clinic" tab.</div></div>';
      return;
    }
    grid.innerHTML = filtered.map((d,i)=>{
      const sym = CURRENCY_SYMBOLS[d.currency] !== undefined ? CURRENCY_SYMBOLS[d.currency] : '';
      return \`
      <div class="card" style="animation-delay:\${i*0.04}s">
        <div class="card-top">
          <div class="avatar">\${initials(d.name)}</div>
          <div>
            <div class="doc-name">\${d.name}</div>
            <div class="doc-spec">\${d.spec}</div>
            <div class="doc-meta">\${d.exp || ''} \${d.exp ? '·' : ''} \${d.clinic}, \${d.city}, \${d.country}</div>
            <div class="stars">★★★★★ <span style="color:#7A8582">\${d.rating}</span></div>
          </div>
        </div>
        <div class="token">
          <div class="token-left">
            <div class="token-num mono">\${d.token}</div>
            <div class="token-label">today's token</div>
          </div>
          <div class="fee">\${sym}\${d.fee} \${d.currency==='OTHER'?'':d.currency}</div>
        </div>
        <button class="book-btn" onclick='openModal(\${i})'>
          <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="4" width="18" height="18" rx="2"/><line x1="16" y1="2" x2="16" y2="6"/><line x1="8" y1="2" x2="8" y2="6"/><line x1="3" y1="10" x2="21" y2="10"/></svg>
          Book appointment
        </button>
      </div>
    \`;}).join("");
  }

  function openModal(idx){
    selectedDoc = getFiltered()[idx];
    const sym = CURRENCY_SYMBOLS[selectedDoc.currency] !== undefined ? CURRENCY_SYMBOLS[selectedDoc.currency] : '';
    document.getElementById("modalDocName").textContent = selectedDoc.name;
    document.getElementById("modalDocSpec").textContent = selectedDoc.spec + " · " + sym + selectedDoc.fee + " " + (selectedDoc.currency==='OTHER'?'':selectedDoc.currency) + " consultation fee";
    document.getElementById("modalOverlay").classList.add("open");
  }
  function closeModal(){ document.getElementById("modalOverlay").classList.remove("open"); }

  function confirmBooking(){
    const name = document.getElementById("patientName").value.trim();
    const phone = document.getElementById("patientPhone").value.trim();
    const date = document.getElementById("apptDate").value;
    const slot = document.getElementById("apptSlot").value;
    if(!name || !phone || !date){ alert("Please fill in your name, phone number and date."); return; }
    const msg = "Hello " + selectedDoc.name + ", I'm " + name + " and I'd like to book an appointment.\\n\\nSpecialty: " + selectedDoc.spec + "\\nDate: " + date + "\\nTime slot: " + slot + "\\nMy number: " + phone + "\\n\\nPlease confirm.";
    const waNumber = selectedDoc.phone.replace(/[^\\d]/g, "");
    window.open("https://wa.me/" + waNumber + "?text=" + encodeURIComponent(msg), "_blank");
    closeModal();
  }

  async function submitDoctor(){
    const payload = {
      name: document.getElementById("regName").value.trim(),
      spec: document.getElementById("regSpec").value.trim(),
      exp: document.getElementById("regExp").value.trim(),
      clinic: document.getElementById("regClinic").value.trim(),
      city: document.getElementById("regCity").value.trim(),
      country: document.getElementById("regCountry").value.trim(),
      fee: document.getElementById("regFee").value.trim(),
      currency: document.getElementById("regCurrency").value,
      phone: document.getElementById("regPhone").value.trim(),
      plan: selectedPlan,
      txnId: document.getElementById("regTxnId").value.trim(),
    };
    if(!payload.name || !payload.spec || !payload.clinic || !payload.city || !payload.country || !payload.fee || !payload.phone || !payload.txnId){
      alert("Please fill in all fields, including your payment transaction ID.");
      return;
    }
    if(!payload.phone.startsWith('+')){
      alert("Please include your country code, starting with + (e.g. +1, +44, +91).");
      return;
    }
    try{
      const res = await fetch('/api/doctors', {
        method: 'POST',
        headers: {'Content-Type':'application/json'},
        body: JSON.stringify(payload)
      });
      const data = await res.json();
      if(data.success){
        document.getElementById("successMsg").classList.add("show");
        ['regName','regSpec','regExp','regClinic','regCity','regCountry','regFee','regPhone','regTxnId'].forEach(id=>document.getElementById(id).value='');
        await loadDoctors();
      } else {
        alert(data.error || "Something went wrong, please try again.");
      }
    }catch(e){
      alert("Network error, please try again.");
    }
  }

  document.getElementById("searchInput").addEventListener("input", renderGrid);
  document.getElementById("modalOverlay").addEventListener("click", (e)=>{ if(e.target.id==="modalOverlay") closeModal(); });

  loadDoctors();
  loadPaymentConfig();
</script>
</body>
</html>`;

// =========================================================
// ADMIN PANEL HTML
// =========================================================
const ADMIN_HTML = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>DocSlot Admin</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,600;9..144,700&family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@500;600&display=swap" rel="stylesheet">
<style>
  :root{ --bg:#F7F4EC; --card:#FFFDF8; --ink:#16211F; --teal:#0F3D3E; --teal-deep:#0A2B2C; --amber:#C9971B; --amber-soft:#F1E3BE; --line:#DCD5C2; --red:#C0392B; --green:#1F5E32; }
  *{box-sizing:border-box; margin:0; padding:0;}
  body{ background:var(--bg); color:var(--ink); font-family:'IBM Plex Sans', sans-serif; }
  .mono{ font-family:'IBM Plex Mono', monospace; }
  h1,h2{ font-family:'Fraunces', serif; }

  header{ background:var(--teal); color:#F7F4EC; padding:16px 20px; }
  header .logo{ font-size:19px; font-weight:600; max-width:900px; margin:0 auto; }

  .login-wrap{ max-width:360px; margin:80px auto; padding:0 20px; }
  .login-card{ background:var(--card); border:1px solid var(--line); border-radius:16px; padding:26px; }
  .login-card h2{ font-size:20px; color:var(--teal-deep); margin-bottom:14px; }
  .login-card input{ width:100%; padding:11px 12px; border-radius:9px; border:1.5px solid var(--line); font-size:14.5px; margin-bottom:12px; }
  .login-card button{ width:100%; background:var(--teal); color:#fff; border:none; padding:12px; border-radius:9px; font-weight:600; cursor:pointer; }
  .login-error{ color:var(--red); font-size:12.5px; margin-top:8px; display:none; }

  #dashboard{ display:none; }
  .container{ max-width:900px; margin:0 auto; padding:22px 20px 60px; }

  .filter-tabs{ display:flex; gap:8px; margin-bottom:18px; flex-wrap:wrap; }
  .ftab{ padding:7px 14px; border-radius:999px; border:1.5px solid var(--line); background:var(--card); font-size:12.5px; font-weight:600; cursor:pointer; color:#4A5754; }
  .ftab.active{ background:var(--teal); border-color:var(--teal); color:#fff; }

  .listing{ background:var(--card); border:1px solid var(--line); border-radius:14px; padding:16px; margin-bottom:12px; }
  .listing-top{ display:flex; justify-content:space-between; align-items:flex-start; gap:10px; }
  .listing-name{ font-weight:600; font-size:15.5px; color:var(--teal-deep); }
  .listing-spec{ font-size:12.5px; color:var(--amber); font-weight:600; }
  .listing-meta{ font-size:12px; color:#7A8582; margin-top:4px; line-height:1.6; }
  .badge{ font-size:10.5px; font-weight:700; padding:3px 9px; border-radius:999px; text-transform:uppercase; letter-spacing:0.3px; }
  .badge.pending{ background:#FCEFC7; color:#8A6100; }
  .badge.approved{ background:#DCF0E1; color:var(--green); }
  .badge.rejected{ background:#F8D7D5; color:var(--red); }
  .badge.expired{ background:#E5E5E5; color:#666; }

  .txn-box{ background:var(--amber-soft); border-radius:8px; padding:8px 10px; font-size:12px; margin-top:8px; }

  .actions{ display:flex; gap:8px; margin-top:12px; flex-wrap:wrap; }
  .actions button{ padding:8px 14px; border-radius:8px; border:none; font-size:12.5px; font-weight:600; cursor:pointer; }
  .btn-approve{ background:var(--green); color:#fff; }
  .btn-reject{ background:var(--red); color:#fff; }
  .btn-renew{ background:var(--teal); color:#fff; }
  .btn-delete{ background:transparent; border:1.5px solid var(--line); color:#5A655F; }

  .empty{ text-align:center; padding:50px 20px; color:#8A948F; }
</style>
</head>
<body>

<header><div class="logo">DocSlot Admin</div></header>

<div class="login-wrap" id="loginWrap">
  <div class="login-card">
    <h2>Admin Login</h2>
    <input type="password" id="pwInput" placeholder="Admin password">
    <button onclick="login()">Login</button>
    <div class="login-error" id="loginError">Wrong password, try again.</div>
  </div>
</div>

<div id="dashboard">
  <div class="container">
    <div class="filter-tabs">
      <div class="ftab active" onclick="setFilter('pending')" id="f-pending">Pending</div>
      <div class="ftab" onclick="setFilter('approved')" id="f-approved">Approved</div>
      <div class="ftab" onclick="setFilter('rejected')" id="f-rejected">Rejected</div>
      <div class="ftab" onclick="setFilter('all')" id="f-all">All</div>
    </div>
    <div id="listings"><div class="empty">Loading...</div></div>
  </div>
</div>

<script>
  let adminPw = "";
  let allListings = [];
  let activeFilter = "pending";

  async function login(){
    const pw = document.getElementById('pwInput').value;
    const res = await fetch('/api/admin/login', { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({password: pw}) });
    const data = await res.json();
    if(data.success){
      adminPw = pw;
      document.getElementById('loginWrap').style.display = 'none';
      document.getElementById('dashboard').style.display = 'block';
      loadListings();
    } else {
      document.getElementById('loginError').style.display = 'block';
    }
  }

  function setFilter(f){
    activeFilter = f;
    ['pending','approved','rejected','all'].forEach(x => document.getElementById('f-'+x).classList.toggle('active', x===f));
    renderListings();
  }

  async function loadListings(){
    const res = await fetch('/api/admin/listings', { headers: {'x-admin-password': adminPw} });
    allListings = await res.json();
    renderListings();
  }

  function statusOf(d){
    if(d.status === 'approved' && d.expiresAt && d.expiresAt < Date.now()) return 'expired';
    return d.status;
  }

  function renderListings(){
    const filtered = activeFilter==='all' ? allListings : allListings.filter(d => statusOf(d)===activeFilter);
    const el = document.getElementById('listings');
    if(filtered.length===0){ el.innerHTML = '<div class="empty">Nothing here.</div>'; return; }

    el.innerHTML = filtered.map(d => {
      const st = statusOf(d);
      const expiry = d.expiresAt ? new Date(d.expiresAt).toLocaleDateString() : '—';
      return \`
      <div class="listing">
        <div class="listing-top">
          <div>
            <div class="listing-name">\${d.name}</div>
            <div class="listing-spec">\${d.spec}</div>
          </div>
          <span class="badge \${st}">\${st}</span>
        </div>
        <div class="listing-meta">
          \${d.clinic}, \${d.city}, \${d.country}<br>
          Fee: \${d.fee} \${d.currency} · Plan: \${d.plan} · WhatsApp: \${d.phone}<br>
          Submitted: \${new Date(d.createdAt).toLocaleDateString()} · Expires: \${expiry}
        </div>
        <div class="txn-box mono">Txn ID: \${d.txnId}</div>
        <div class="actions">
          \${st==='pending' ? '<button class="btn-approve" onclick="approve(\\''+d.id+'\\')">Approve</button><button class="btn-reject" onclick="reject(\\''+d.id+'\\')">Reject</button>' : ''}
          \${st==='approved' || st==='expired' ? '<button class="btn-renew" onclick="renew(\\''+d.id+'\\')">Renew</button>' : ''}
          <button class="btn-delete" onclick="removeListing('\${d.id}')">Delete</button>
        </div>
      </div>\`;
    }).join('');
  }

  async function approve(id){
    await fetch('/api/admin/approve', { method:'POST', headers:{'Content-Type':'application/json','x-admin-password':adminPw}, body: JSON.stringify({id}) });
    loadListings();
  }
  async function reject(id){
    await fetch('/api/admin/reject', { method:'POST', headers:{'Content-Type':'application/json','x-admin-password':adminPw}, body: JSON.stringify({id}) });
    loadListings();
  }
  async function renew(id){
    await fetch('/api/admin/renew', { method:'POST', headers:{'Content-Type':'application/json','x-admin-password':adminPw}, body: JSON.stringify({id}) });
    loadListings();
  }
  async function removeListing(id){
    if(!confirm('Delete this listing permanently?')) return;
    await fetch('/api/admin/delete', { method:'POST', headers:{'Content-Type':'application/json','x-admin-password':adminPw}, body: JSON.stringify({id}) });
    loadListings();
  }
</script>
</body>
</html>`;
