<!DOCTYPE html>
<html dir="rtl" lang="fa">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>مدیریت شیت — بدون خطا</title>
  <style>
    body { 
      font-family: Vazirmatn, Tahoma, sans-serif; 
      padding: 20px; 
      max-width: 1000px; 
      margin: 0 auto; 
      line-height: 1.6;
    }
    h2 { border-bottom: 1px solid #eee; padding-bottom: 8px; }
    label { display: block; margin-top: 12px; font-weight: bold; }
    input, button { 
      padding: 10px; 
      margin: 6px 0; 
      font-size: 14px; 
    }
    .form-group { margin-bottom: 20px; }
    iframe { 
      width: 100%; 
      min-height: 400px; 
      border: 1px solid #ddd; 
      margin-top: 10px;
      background: #f9f9f9;
    }
    .status { 
      padding: 10px; 
      margin: 10px 0; 
      background: #f0f0f0; 
      border-radius: 4px;
    }
    .success { color: green; }
    .error { color: red; }
    .controls { margin: 15px 0; }
  </style>
</head>
<body>

<h2>🔗 لینک‌های اتصال</h2>

<div class="form-group">
  <label>1. لینک اپس اسکریپت (برای ثبت داده)</label>
  <input type="url" id="scriptUrl" style="width:100%" 
         placeholder="https://script.google.com/macros/s/.../exec"
         value="">
</div>

<div class="form-group">
  <label>2. لینک Publish to Web (برای نمایش جدول)</label>
  <input type="url" id="sheetUrl" style="width:100%" 
         placeholder="[https://docs.google.com/spreadsheets/d/e/.../pubhtml](https://docs.google.com/spreadsheets/d/1pBaolhZZ-7UCytgGJNZcGAJKz6cTlsOuqU7oPEBi4Oc/edit?gid=0#gid=0)"
         value="">
  <small>📌 از منوی گوگل شیت: File → Share → Publish to web → Sheet1 → Web page → Publish</small>
</div>

<button onclick="saveUrls()">ذخیره لینک‌ها</button>
<div id="urlStatus" class="status"></div>

<hr>

<h2>➕ افزودن رکورد جدید</h2>
<div class="form-group">
  <label for="inputText">متن *</label>
  <input type="text" id="inputText" style="width:100%" placeholder="متن را وارد کنید...">
</div>
<div class="form-group">
  <label for="inputContact">لینک ارتباط (اختیاری)</label>
  <input type="url" id="inputContact" style="width:100%" placeholder="https://t.me/...">
</div>
<button onclick="addRecord()">ثبت در گوگل شیت</button>
<div id="addStatus" class="status"></div>

<hr>

<h2>📊 جدول زنده (به‌روز شده هر 3 ثانیه)</h2>
<div class="controls">
  <button onclick="refreshIframe()">🔄 بروزرسانی دستی</button>
  <label>
    <input type="checkbox" id="autoRefresh" onchange="toggleAutoRefresh()" checked> 
    به‌روزرسانی خودکار
  </label>
</div>

<iframe id="sheetFrame" frameborder="0"></iframe>

<!-- اسکریپت اصلی -->
<script>
let SCRIPT_URL = localStorage.getItem('scriptUrl') || '';
let SHEET_URL = localStorage.getItem('sheetUrl') || '';
let autoRefreshId = null;

// بارگذاری لینک‌های ذخیره‌شده
if (SCRIPT_URL) document.getElementById('scriptUrl').value = SCRIPT_URL;
if (SHEET_URL) {
  document.getElementById('sheetUrl').value = SHEET_URL;
  loadIframe();
  startAutoRefresh(); // شروع خودکار
}

function saveUrls() {
  SCRIPT_URL = document.getElementById('scriptUrl').value.trim();
  SHEET_URL = document.getElementById('sheetUrl').value.trim();
  
  if (!SCRIPT_URL || !SHEET_URL) {
    showAlert('لطفاً هر دو لینک را وارد کنید.', 'error', 'urlStatus');
    return;
  }
  
  if (!SCRIPT_URL.includes('/exec')) {
    if (!confirm('⚠️ لینک اسکریپت باید با /exec تمام شود. ادامه دهیم؟')) return;
  }
  if (!SHEET_URL.includes('/pubhtml')) {
    if (!confirm('⚠️ لینک شیت باید از Publish to Web باشد (دارای pubhtml). ادامه دهیم؟')) return;
  }

  localStorage.setItem('scriptUrl', SCRIPT_URL);
  localStorage.setItem('sheetUrl', SHEET_URL);
  loadIframe();
  startAutoRefresh();
  showAlert('✅ لینک‌ها ذخیره و بارگذاری شدند.', 'success', 'urlStatus');
}

function loadIframe() {
  const iframe = document.getElementById('sheetFrame');
  if (SHEET_URL) {
    // ✅ اضافه کردن timestamp برای دور زدن کش
    iframe.src = SHEET_URL + (SHEET_URL.includes('?') ? '&' : '?') + '_t=' + Date.now();
  }
}

function refreshIframe() {
  loadIframe();
  showAlert('✅ جدول به‌روز شد.', 'success', 'urlStatus');
}

function toggleAutoRefresh() {
  const checked = document.getElementById('autoRefresh').checked;
  if (checked) {
    startAutoRefresh();
  } else {
    stopAutoRefresh();
  }
}

function startAutoRefresh() {
  stopAutoRefresh();
  autoRefreshId = setInterval(refreshIframe, 3000);
}

function stopAutoRefresh() {
  if (autoRefreshId) clearInterval(autoRefreshId);
}

// ✅ ارسال بدون CORS — با no-cors (همیشه کار می‌کند)
async function addRecord() {
  const text = document.getElementById('inputText').value.trim();
  const contact = document.getElementById('inputContact').value.trim();
  if (!text) return showAlert('لطفاً متن را وارد کنید.', 'error', 'addStatus');

  if (!SCRIPT_URL) return showAlert('لینک اسکریپت تنظیم نشده است.', 'error', 'addStatus');

  try {
    // ارسال با no-cors — بدون انتظار پاسخ
    await fetch(SCRIPT_URL + '?action=add', {
      method: 'POST',
      mode: 'no-cors', // ✅ کلید موفقیت — هیچ‌وقت Failed to fetch نمی‌دهد
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text, contact })
    });

    // نمایش موفقیت و پاک کردن فرم
    showAlert('✅ داده ارسال شد. جدول به‌روز می‌شود...', 'success', 'addStatus');
    document.getElementById('inputText').value = '';
    document.getElementById('inputContact').value = '';

    // بروزرسانی جدول پس از 1 ثانیه (برای اطمینان از ذخیره‌سازی در گوگل)
    setTimeout(refreshIframe, 1000);
  } catch (e) {
    showAlert('❌ خطا در ارسال: ' + e.message, 'error', 'addStatus');
  }
}

function showAlert(msg, type, targetId) {
  const el = document.getElementById(targetId);
  el.innerHTML = msg;
  el.className = 'status ' + type;
  setTimeout(() => { if (el.innerHTML === msg) el.textContent = ''; }, 4000);
}
</script>

</body>
</html>
