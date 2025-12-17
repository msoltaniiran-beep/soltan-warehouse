<!DOCTYPE html>
<html dir="rtl">
<head>
  <meta charset="UTF-8">
  <title>اتصال به گوگل شیت</title>
  <style>
    body { font-family: Vazirmatn, sans-serif; padding: 20px; max-width: 900px; margin: 0 auto; }
    input, button { padding: 8px; margin: 5px 0; }
    label { display: block; margin-top: 15px; font-weight: bold; }
    #tableContainer { margin-top: 20px; }
    table { width: 100%; border-collapse: collapse; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: right; }
    .actions button { margin: 0 3px; font-size: 12px; padding: 4px 8px; }
    .status { margin-top: 10px; padding: 8px; background: #f0f0f0; }
    .error { color: red; }
    .success { color: green; }
  </style>
</head>
<body>

<h2>تنظیمات ارتباط با گوگل شیت</h2>
<label for="sheetUrl">لینک وب‌اپلیکیشن گوگل اپس اسکریپت (مثل <code>.../exec</code>)</label>
<input type="url" id="sheetUrl" style="width:100%" 
       placeholder="https://script.google.com/macros/s/ABC.../exec" />
<button onclick="saveUrl()">ذخیره لینک</button>
<div id="urlStatus" class="status"></div>

<hr>

<h2>افزودن رکورد جدید</h2>
<label for="inputText">متن:</label>
<input type="text" id="inputText" style="width:100%" placeholder="متن را وارد کنید...">
<label for="inputContact">لینک ارتباط (اختیاری):</label>
<input type="url" id="inputContact" style="width:100%" placeholder="https://t.me/... یا ایمیل">

<button onclick="addRecord()" id="addBtn">افزودن به شیت</button>
<div id="addStatus" class="status"></div>

<div id="tableContainer">
  <h3>داده‌های شیت</h3>
  <button onclick="loadData()" id="refreshBtn">🔄 بارگذاری مجدد</button>
  <table id="dataTable">
    <thead><tr><th>ID</th><th>متن</th><th>ارتباط</th><th>زمان</th><th>عملیات</th></tr></thead>
    <tbody id="tableBody"></tbody>
  </table>
</div>

<script>
let SHEET_URL = localStorage.getItem('sheetUrl') || '';

if (SHEET_URL) {
  document.getElementById('sheetUrl').value = SHEET_URL;
  document.getElementById('urlStatus').innerHTML = '<span class="success">✅ لینک ذخیره‌شده بارگذاری شد.</span>';
  setTimeout(loadData, 500);
}

function saveUrl() {
  const url = document.getElementById('sheetUrl').value.trim();
  if (!url) return showAlert('لطفاً لینک را وارد کنید.', 'error', 'urlStatus');
  if (!url.includes('/macros/s/') || !url.endsWith('/exec')) {
    if (!confirm('⚠️ لینک شبیه لینک وب‌اپلیکیشن نیست. ادامه دهیم؟')) return;
  }
  SHEET_URL = url;
  localStorage.setItem('sheetUrl', url);
  showAlert('✅ لینک ذخیره شد. داده‌ها بارگذاری می‌شوند...', 'success', 'urlStatus');
  loadData();
}

async function callApi(action, data = {}) {
  if (!SHEET_URL) throw new Error('لینک شیت تنظیم نشده است.');
  
  const url = SHEET_URL + '?action=' + action;
  const res = await fetch(url, {
    method: 'POST',
    mode: 'no-cors', // فقط برای ارسال — پاسخ را دریافت نمی‌کند!
    // برای دریافت پاسخ، باید mode: 'cors' باشد — اما گوگل فقط در صورتی CORS را فعال می‌کند که شما در Apps Script از doGet استفاده کنید.
    // راه‌حل: از no-cors برای ارسال، و یک درخواست جداگانه برای بارگذاری داده‌ها استفاده می‌کنیم.
  });

  // برای no-cors، res.json() ممکن است خطا بدهد — پس فقط delay قائل می‌شویم و دوباره لیست را بارگیری می‌کنیم
  if (action !== 'list') await new Promise(r => setTimeout(r, 600));
  
  if (action === 'list') {
    // برای دریافت داده‌ها، حتماً باید از doGet با خروجی JSON استفاده کنیم — یا از روش زیر
    const proxy = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`;
    const proxyRes = await fetch(proxy);
    const proxyData = await proxyRes.json();
    return JSON.parse(proxyData.contents);
  }
  
  return { status: 'ok' };
}

// راه‌حل عملی: فقط عملیات 'list' را با proxy بخوانیم، بقیه را با no-cors ارسال کنیم
async function loadData() {
  try {
    document.getElementById('refreshBtn').disabled = true;
    const result = await callApi('list');
    if (result.status === 'error') throw new Error(result.message);
    
    const tbody = document.getElementById('tableBody');
    tbody.innerHTML = '';
    result.data?.forEach(row => {
      const tr = tbody.insertRow();
      tr.insertCell().textContent = row.id.substring(0,6) + '…';
      tr.insertCell().textContent = row.text;
      const contactCell = tr.insertCell();
      if (row.contact) {
        contactCell.innerHTML = `<a href="${row.contact}" target="_blank" class="link">${row.contact}</a>`;
      } else {
        contactCell.textContent = '—';
      }
      tr.insertCell().textContent = row.timestamp;
      const actions = tr.insertCell();
      actions.className = 'actions';
      actions.innerHTML = `
        <button onclick="editRow('${row.id}', \`${row.text.replace(/`/g, '\\`')}\`, \`${(row.contact || '').replace(/`/g, '\\`')}\`)">ویرایش</button>
        <button onclick="deleteRow('${row.id}')">حذف</button>
      `;
    });
    document.getElementById('refreshBtn').disabled = false;
  } catch (e) {
    showAlert('❌ خطا در بارگذاری: ' + e.message, 'error', 'urlStatus');
    document.getElementById('refreshBtn').disabled = false;
  }
}

async function addRecord() {
  const text = document.getElementById('inputText').value.trim();
  const contact = document.getElementById('inputContact').value.trim();
  if (!text) return showAlert('لطفاً متن را وارد کنید.', 'error', 'addStatus');
  
  try {
    await callApi('add', { text, contact });
    showAlert('✅ رکورد افزوده شد.', 'success', 'addStatus');
    document.getElementById('inputText').value = '';
    document.getElementById('inputContact').value = '';
    loadData();
  } catch (e) {
    showAlert('❌ خطا در افزودن: ' + e.message, 'error', 'addStatus');
  }
}

async function editRow(id, text, contact) {
  const newText = prompt('متن جدید:', text);
  if (newText === null) return;
  const newContact = prompt('لینک ارتباط جدید:', contact || '');
  if (newContact === null) return;

  try {
    await callApi('update', { id, text: newText, contact: newContact });
    showAlert('✅ رکورد به‌روز شد.', 'success', 'urlStatus');
    loadData();
  } catch (e) {
    showAlert('❌ خطا در به‌روزرسانی: ' + e.message, 'error', 'urlStatus');
  }
}

async function deleteRow(id) {
  if (!confirm('آیا مطمئنید؟')) return;
  try {
    await callApi('delete', { id });
    showAlert('🗑️ رکورد حذف شد.', 'success', 'urlStatus');
    loadData();
  } catch (e) {
    showAlert('❌ خطا در حذف: ' + e.message, 'error', 'urlStatus');
  }
}

function showAlert(msg, type, targetId) {
  const el = document.getElementById(targetId);
  el.innerHTML = msg;
  el.className = 'status ' + type;
  setTimeout(() => {
    if (el.innerHTML === msg) el.textContent = '';
  }, 4000);
}
</script>

</body>
</html>
