<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>anhtubantool</title>

<style>
*{box-sizing:border-box}
body{
    margin:0;
    font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Arial;
    background:linear-gradient(270deg,red,orange,yellow,green,cyan,blue,violet);
    background-size:1400% 1400%;
    animation:rainbow 18s ease infinite;
    color:#fff;
    min-height:100vh;
}
@keyframes rainbow{
    0%{background-position:0% 50%}
    50%{background-position:100% 50%}
    100%{background-position:0% 50%}
}

.app{max-width:420px;margin:auto;padding:12px}

.header{
    text-align:center;
    font-size:22px;
    font-weight:bold;
    padding:12px;
    border:2px solid #fff;
    border-radius:16px;
    margin-bottom:12px;
}

.tabs{display:flex;gap:6px;margin-bottom:10px}
.tab{
    flex:1;
    padding:10px;
    text-align:center;
    border:2px solid #fff;
    border-radius:12px;
    cursor:pointer;
    font-weight:bold;
}
.tab.active{background:rgba(255,255,255,0.25)}

.section{
    display:none;
    border:2px solid #fff;
    border-radius:16px;
    padding:14px;
    background:rgba(0,0,0,0.35);
}
.section.active{display:block}

input,button{
    width:100%;
    padding:12px;
    margin-top:8px;
    border-radius:12px;
    border:none;
    font-size:15px;
}
button{font-weight:bold;cursor:pointer}

.reset{
    margin-top:10px;
    background:#000;
    color:#fff;
    border:2px solid #fff;
}

/* ===== TABLE ===== */
table{
    width:100%;
    border-collapse:collapse;
    margin-top:10px;
    font-size:13px;
}
th,td{
    border:1px solid #fff;
    padding:6px;
    text-align:center;
}
th{background:rgba(255,255,255,0.2)}
</style>
</head>

<body>
<div class="app">

<div class="header">🌈 anhtubantool 🌈</div>

<div class="tabs">
    <div class="tab active" onclick="showTab(0)">SICBO</div>
    <div class="tab" onclick="showTab(1)">SUNWIN</div>
    <div class="tab" onclick="showTab(2)">CHẴN / LẺ</div>
</div>

<!-- ================= SICBO ================= -->
<div class="section active">
<h3>🎲 SICBO AI</h3>
<input id="md5Input" placeholder="Nhập MD5 32 ký tự">
<button onclick="analyzeSicbo()">PHÂN TÍCH</button>

<table>
<thead>
<tr><th>#</th><th>Xúc xắc</th><th>Tổng</th><th>KQ</th></tr>
</thead>
<tbody id="sicboHist"></tbody>
</table>

<button class="reset" onclick="resetHistory('sicbo_hist',renderSicbo)">♻️ RESET LỊCH SỬ</button>
</div>

<!-- ================= SUNWIN ================= -->
<div class="section">
<h3>🎲 SUNWIN AI</h3>
<div>⏳ Làm mới sau <b><span id="countdown">60</span></b> giây</div>
<p>Phiên: <span id="phien"></span></p>
<p>Xúc xắc: <span id="dice"></span></p>
<p>Tổng: <span id="tong"></span></p>
<p>Kết quả: <span id="ketqua"></span></p>
<p>Dự đoán: <span id="dudoan"></span></p>
<p>Cầu: <span id="cau"></span></p>
</div>

<!-- ================= CHẴN LẺ ================= -->
<div class="section">
<h3>🔐 MD5 → CHẴN / LẺ</h3>
<input id="md5CL" placeholder="Nhập MD5 32 ký tự">
<button onclick="analyzeCL()">PHÂN TÍCH</button>

<table>
<thead>
<tr><th>#</th><th>MD5</th><th>KQ</th></tr>
</thead>
<tbody id="clHist"></tbody>
</table>

<button class="reset" onclick="resetHistory('cl_hist',renderCL)">♻️ RESET LỊCH SỬ</button>
</div>

</div>

<script>
/* ===== TAB ===== */
function showTab(i){
    document.querySelectorAll('.tab').forEach((t,idx)=>t.classList.toggle('active',idx===i))
    document.querySelectorAll('.section').forEach((s,idx)=>s.classList.toggle('active',idx===i))
}

/* ===== RESET ===== */
function resetHistory(key,cb){
    localStorage.removeItem(key);
    cb();
}

/* ===== SICBO ===== */
/* ====== HẰNG SỐ ====== */
const AI_RAND_BOOST_THRESHOLD = 3.5;
const AI_RAND_BOOST_FACTOR = 15.0;
const AI_MAX_SCORE = 100.0;

const DEEP_ENTROPY_WEIGHT = 10.0;
const DEEP_ENTROPY_DIFF_BASELINE = 50.0;
const DEEP_ENTROPY_DIFF_PENALTY = 20.0;
const DEEP_SYMMETRY_WEIGHT = 30.0;
const DEEP_REPEAT_PENALTY_WEIGHT = 15.0;
const DEEP_MIN_SCORE = 0.0;
const DEEP_MAX_SCORE = 100.0;

/* ====== ENTROPY HEX ====== */
function hexEntropy(hex) {
    if (!hex) return 0;
    const freq = {};
    for (let c of hex) freq[c] = (freq[c] || 0) + 1;
    let entropy = 0;
    const total = hex.length;
    for (let c in freq) {
        const p = freq[c] / total;
        entropy -= p * Math.log2(p);
    }
    return entropy;
}

/* ====== DEEP SCORE ====== */
function deepScore(md5) {
    const ent = hexEntropy(md5);
    const freq = {};
    for (let c of md5) freq[c] = (freq[c] || 0) + 1;
    const repeated = Object.values(freq).filter(v => v > 2).length;
    const repPenalty = repeated / Object.keys(freq).length;
    const half = Math.floor(md5.length / 2);
    let symmetryMatches = 0;
    for (let i = 0; i < half; i++) {
        if (md5[i] === md5[i + half]) symmetryMatches++;
    }
    const symmetry = symmetryMatches / half;
    let score =
        ent * DEEP_ENTROPY_WEIGHT +
        (DEEP_ENTROPY_DIFF_BASELINE - ent * DEEP_ENTROPY_DIFF_PENALTY) +
        symmetry * DEEP_SYMMETRY_WEIGHT -
        repPenalty * DEEP_REPEAT_PENALTY_WEIGHT;
    return Math.max(DEEP_MIN_SCORE, Math.min(DEEP_MAX_SCORE, score));
}

/* ====== XÁC SUẤT 3D6 ====== */
function compute3D6() {
    const counts = {};
    for (let i = 3; i <= 18; i++) counts[i] = 0;
    for (let d1 = 1; d1 <= 6; d1++)
        for (let d2 = 1; d2 <= 6; d2++)
            for (let d3 = 1; d3 <= 6; d3++)
                counts[d1 + d2 + d3]++;
    const total = 216;
    const probs = {};
    for (let k in counts) probs[k] = counts[k] / total;
    return probs;
}

const PROBS_3D6 = compute3D6();
const XIU_RANGE = Object.fromEntries(Object.entries(PROBS_3D6).filter(([k]) => k >= 4 && k <= 10));
const TAI_RANGE = Object.fromEntries(Object.entries(PROBS_3D6).filter(([k]) => k >= 11 && k <= 17));

/* ====== RNG SEED ====== */
function seededRandom(seed) {
    let x = Math.sin(seed++) * 10000;
    return x - Math.floor(x);
}

/* ====== CHỌN VỊ LÓT ====== */
function weightedSample(weightedMap, md5, entropy, deep) {
    let items = Object.entries(weightedMap).map(([k, v]) => [Number(k), v]);
    const seed = parseInt(md5.slice(0, 16), 16);
    let localSeed = seed;

    const boost = Math.max(0, (entropy - AI_RAND_BOOST_THRESHOLD) * AI_RAND_BOOST_FACTOR) * (deep / DEEP_MAX_SCORE);
    items = items.map(([k, v]) => [k, v + boost]);

    const selected = [];
    while (selected.length < 3 && items.length > 0) {
        const totalWeight = items.reduce((s, [, w]) => s + w, 0);
        let r = seededRandom(localSeed++) * totalWeight;
        let acc = 0;
        for (let i = 0; i < items.length; i++) {
            acc += items[i][1];
            if (acc >= r) {
                selected.push(items[i][0]);
                items.splice(i, 1);
                break;
            }
        }
    }
    return selected.sort((a, b) => a - b);
}

/* ====== PHÂN TÍCH ====== */
function analyze() {
    const md5 = document.getElementById("md5Input").value.trim().toLowerCase();
    const out = document.getElementById("output");

    if (!/^[0-9a-f]{32}$/.test(md5)) {
        out.textContent = "❌ MD5 không hợp lệ!";
        return;
    }

    const seed = parseInt(md5.slice(0, 16), 16);
    let s = seed;
    const d1 = Math.floor(seededRandom(s++) * 6) + 1;
    const d2 = Math.floor(seededRandom(s++) * 6) + 1;
    const d3 = Math.floor(seededRandom(s++) * 6) + 1;
    const total = d1 + d2 + d3;
    const triple = d1 === d2 && d2 === d3;

    const entropy = hexEntropy(md5);
    const deep = deepScore(md5);

    let predicted, icon, range;
    if (triple) {
        predicted = `BỘ BA (${d1}-${d2}-${d3})`;
        icon = "👑";
        range = PROBS_3D6;
    } else {
        const threshold = 10.5 + (entropy - 4.0) + (deep / DEEP_MAX_SCORE);
        if (total <= threshold) {
            predicted = "XỈU";
            icon = "❄️";
            range = XIU_RANGE;
        } else {
            predicted = "TÀI";
            icon = "🔥";
            range = TAI_RANGE;
        }
    }

    const lot = weightedSample(range, md5, entropy, deep);

    out.textContent =
`==============================
${icon} DỰ ĐOÁN: ${predicted}
🎯 VỊ LÓT: ${lot.join(" - ")}
🔑 Entropy MD5: ${entropy.toFixed(3)}
🧠 Deep Score: ${deep.toFixed(3)}
==============================`;
}

/* ===== SUNWIN ===== */
let cd=60;
async function loadSun(){
    try{
        let r=await fetch("https://sunwinsaygex-tzz9.onrender.com/api/sun");
        let d=await r.json();
        phien.textContent=d.phien;
        dice.textContent=`${d.xuc_xac_1}-${d.xuc_xac_2}-${d.xuc_xac_3}`;
        tong.textContent=d.tong;
        ketqua.textContent=d.ket_qua;
        dudoan.textContent=d.du_doan;
        cau.textContent=d.cau;
    }catch{ketqua.textContent="Lỗi API";}
}
setInterval(()=>{
    countdown.textContent=cd--;
    if(cd<0){cd=60;loadSun()}
},1000);
loadSun();

/* ========= ENTROPY ========= */
function hexEntropy(hex) {
    const freq = {};
    for (let c of hex) freq[c] = (freq[c] || 0) + 1;
    let e = 0;
    for (let k in freq) {
        let p = freq[k] / hex.length;
        e -= p * Math.log2(p);
    }
    return e;
}

/* ========= DEEP SCORE ========= */
function deepScore(md5) {
    const ent = hexEntropy(md5);
    const freq = {};
    for (let c of md5) freq[c] = (freq[c] || 0) + 1;
    let rep = Object.values(freq).filter(v => v > 2).length;
    let sym = 0;
    for (let i = 0; i < 16; i++) if (md5[i] === md5[i+16]) sym++;
    let score = ent*10 + (50-ent*20) + (sym/16)*30 - (rep/Object.keys(freq).length)*15;
    return Math.max(0, Math.min(100, score));
}

/* ========= RNG ========= */
function seededRandom(seed) {
    let x = Math.sin(seed) * 10000;
    return x - Math.floor(x);
}

/* ========= VỊ LÓT ========= */
function pickLot(md5, isLe) {
    const pool = isLe ? [1,3] : [0,2,4];
    let s = parseInt(md5.slice(8,16),16);
    let a = pool[Math.floor(seededRandom(s)*pool.length)];
    let b = pool[Math.floor(seededRandom(s+1)*pool.length)];
    if (a===b) b = pool[(pool.indexOf(a)+1)%pool.length];
    return [a,b].sort((x,y)=>x-y);
}

/* ========= LỜI KHUYÊN ========= */
function advice(conf) {
    if (conf > 75) return "🔥 Kèo mạnh – có thể theo";
    if (conf > 55) return "⚠️ Kèo trung bình – cân nhắc vốn";
    return "❄️ Kèo yếu – nên bỏ qua";
}

/* ========= MAIN ========= */
function analyze() {
    const md5 = document.getElementById("md5").value.trim().toLowerCase();
    const out = document.getElementById("result");
    if (!/^[0-9a-f]{32}$/.test(md5)) {
        out.innerHTML = "❌ MD5 không hợp lệ";
        return;
    }

    const entropy = hexEntropy(md5);
    const deep = deepScore(md5);
    const seed = parseInt(md5.slice(0,8),16);
    const total = 3 + seededRandom(seed)*15;
    const threshold = 10.5 + (entropy-4) + deep/100;
    const isLe = total >= threshold;

    const result = isLe ? "LẺ" : "CHẴN";
    const icon = isLe ? "🔥" : "❄️";
    const lot = pickLot(md5,isLe);

    const confidence = Math.min(95, Math.max(35, Math.round((entropy/4 + deep/100)*50)));

    out.innerHTML = `
        <h3>${icon} KẾT QUẢ</h3>
        <div class="highlight">${result}</div>
        <p><b>Vị lót:</b> ${lot[0]} - ${lot[1]}</p>
        <p>🎯 Tỉ lệ thắng: <b>${confidence}%</b></p>
        <p>💡 ${advice(confidence)}</p>
        <p class="small">Entropy: ${entropy.toFixed(3)} | Deep: ${deep.toFixed(1)}</p>
    `;

    saveHistory(md5, result, confidence);
    loadHistory();
}

/* ========= LỊCH SỬ ========= */
function saveHistory(md5, res, conf) {
    let h = JSON.parse(localStorage.getItem("md5_history") || "[]");
    h.unshift({md5,res,conf});
    if (h.length > 10) h.pop();
    localStorage.setItem("md5_history", JSON.stringify(h));
}

function loadHistory() {
    let h = JSON.parse(localStorage.getItem("md5_history") || "[]");
    document.getElementById("historyList").innerHTML =
        h.map(i=>`<div class="history-item">${i.md5.slice(0,8)}… → ${i.res} (${i.conf}%)</div>`).join("");
}

loadHistory();

/* ========= DÁN ========= */
async function pasteMD5() {
    try {
        const text = await navigator.clipboard.readText();
        document.getElementById("md5").value = text.trim();
    } catch {
        alert("Không thể dán");
    }
}
renderCL();
</script>

</body>
</html>
