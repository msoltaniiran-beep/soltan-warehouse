<!DOCTYPE html>
<html dir="rtl" lang="fa">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>مدیریت شیت</title>
  <style>
    body { font-family: Vazirmatn, Tahoma, sans-serif; padding: 20px; max-width: 1000px; margin: 0 auto; }
    label { display: block; margin-top: 12px; font-weight: bold; }
    input, button { padding: 8px; margin: 4px 0; }
    .form-group { margin-bottom: 15px; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 10px; text-align: right; }
    th { background: #f5f5f5; }
    .actions button { margin: 0 3px; padding: 4px 8px; font-size: 12px; }
    .link { color: #1a0dab; text-decoration: underline; }
    .status { padding: 8px; margin: 8px 0; background: #f0f0f0; }
    .error { color: red; }
    .success { color: green; }
    #autoRefresh { margin-top: 10px; }
  </style>
</head>
<body>

<h2>🔗 تنظیم لینک گوگل اپس اسکریپت</h2>
<div class="form-group">
  <label for="sheetUrl">لینک وب‌اپلیکیشن (مثل <code>.../exec</code>)</label>
  <input type="url" id="sheetUrl" style="width:100%" 
         placeholder="https://script.google.com/macros/s/ABC123/exec" />
  <button onclick="saveUrl()">ذخیره و اتصال</button>
  <div id="urlStatus" class="status"></div>
</div>

<hr>

<h2>➕ افزودن رکورد جدید</h2>
<div class="form-group">
  <label for="inputText">متن *</label>
  <input type="text" id="inputText" style="width:100%" placeholder="متن را وارد کنید...">
</div>
<div class="form-group">
  <label for="inputContact">لینک ارتباط (اختیاری)</label>
  <input type="url" id="inputContact" style="width:100%" placeholder="https://t.me/... یا ایمیل">
</div>
<button onclick="addRecord()" id="addBtn">افزودن به شیت</button>
<div id="addStatus" class="status"></div>

<div id="autoRefresh">
  <label>
    <input type="checkbox" id="autoRefreshCheck" onchange="toggleAutoRefresh()"> به‌روزرسانی خودکار هر 3 ثانیه
  </label>
</div>

<div id="tableContainer">
  <h3>📋 داده‌های شیت (به‌روزشده در <span id="lastUpdate">—</span>)</h3>
  <table id="dataTable">
    <thead><tr><th>ID</th><th>متن</th><th>ارتباط</th><th>زمان</th><th>عملیات</th></tr></thead>
    <tbody id="tableBody"></tbody>
  </table>
</div>

<script>
let SHEET_URL = localStorage.getItem('sheetUrl') || '';
let autoRefreshId = null;

if (SHEET_URL) {
  document.getElementById('sheetUrl').value = SHEET_URL;
  document.getElementById('urlStatus').innerHTML = '<span class="success">✅ لینک ذخیره‌شده بارگذاری شد.</span>';
  setTimeout(loadData, 300);
  if (localStorage.getItem('autoRefresh') === 'true') {
    document.getElementById('autoRefreshCheck').checked = true;
    startAutoRefresh();
  }
}

function saveUrl() {
  const url = document.getElementById('sheetUrl').value.trim();
  if (!url) return showAlert('لطفاً لینک را وارد کنید.', 'error', 'urlStatus');
  if (!url.includes('/macros/s/') || !url.endsWith('/exec')) {
    if (!confirm('لینک شبیه لینک وب‌اپلیکیشن نیست. ادامه دهیم؟')) return;
  }
  SHEET_URL = url;
  localStorage.setItem('sheetUrl', url);
  showAlert('✅ لینک ذخیره شد. داده‌ها بارگذاری می‌شوند...', 'success', 'urlStatus');
  loadData();
}

async function callApi(action, data = {}) {
  if (!SHEET_URL) throw new Error('لینک شیت تنظیم نشده است.');

  const url = SHEET_URL + '?action=' + action;
  const options = {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  };

  if (action === 'list') {
    options.method = 'GET';
    delete options.body;
    delete options.headers;
  }

  const res = await fetch(url, options);
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`HTTP ${res.status}: ${text.substring(0, 200)}`);
  }
  return await res.json();
}

async function loadData() {
  try {
    document.getElementById('addBtn').disabled = true;
    const result = await callApi('list');
    if (result.status !== 'ok') throw new Error(result.message || 'خطا در دریافت داده');

    const tbody = document.getElementById('tableBody');
    tbody.innerHTML = '';
    result.data?.forEach(row => {
      const tr = tbody.insertRow();
      tr.insertCell().textContent = row.id.substring(0,8) + '…';
      tr.insertCell().textContent = row.text || '—';
      const contactCell = tr.insertCell();
      if (row.contact) {
        contactCell.innerHTML = `<a href="${row.contact}" target="_blank" class="link">${row.contact}</a>`;
      } else {
        contactCell.textContent = '—';
      }
      tr.insertCell().textContent = row.timestamp || '';
      
      const actions = tr.insertCell();
      actions.className = 'actions';
      actions.innerHTML = `
        <button onclick="editRow('${row.id}', \`${row.text?.replace(/`/g, '\\`') || ''}\`, \`${row.contact?.replace(/`/g, '\\`') || ''}\`)">ویرایش</button>
        <button onclick="deleteRow('${row.id}')">حذف</button>
      `;
    });
    document.getElementById('lastUpdate').textContent = new Date().toLocaleTimeString('fa-IR');
  } catch (e) {
    showAlert('❌ خطا در بارگذاری: ' + e.message, 'error', 'urlStatus');
  } finally {
    document.getElementById('addBtn').disabled = false;
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
    loadData(); // بروزرسانی فوری لیست
  } catch (e) {
    showAlert('❌ خطا در افزودن: ' + e.message, 'error', 'addStatus');
  }
}

async function editRow(id, text, contact) {
  const newText = prompt('ویرایش متن:', text) || '';
  const newContact = prompt('لینک ارتباط جدید:', contact) || '';
  if (newText === null || newContact === null) return;

  try {
    await callApi('update', { id, text: newText, contact: newContact });
    showAlert('✅ رکورد به‌روز شد.', 'success', 'urlStatus');
    loadData();
  } catch (e) {
    showAlert('❌ خطا در به‌روزرسانی: ' + e.message, 'error', 'urlStatus');
  }
}

async function deleteRow(id) {
  if (!confirm('آیا مطمئنید می‌خواهید این رکورد حذف شود؟')) return;
  try {
    await callApi('delete', { id });
    showAlert('🗑️ رکورد حذف شد.', 'success', 'urlStatus');
    loadData();
  } catch (e) {
    showAlert('❌ خطا در حذف: ' + e.message, 'error', 'urlStatus');
  }
}

function toggleAutoRefresh() {
  const checked = document.getElementById('autoRefreshCheck').checked;
  localStorage.setItem('autoRefresh', checked);
  if (checked) {
    startAutoRefresh();
  } else {
    stopAutoRefresh();
  }
}

function startAutoRefresh() {
  stopAutoRefresh();
  autoRefreshId = setInterval(loadData, 3000);
  console.log('✅ Auto-refresh فعال شد.');
}

function stopAutoRefresh() {
  if (autoRefreshId) clearInterval(autoRefreshId);
  autoRefreshId = null;
}

function showAlert(msg, type, targetId) {
  const el = document.getElementById(targetId);
  el.innerHTML = msg;
  el.className = 'status ' + type;
  setTimeout(() => {
    if (el.innerHTML === msg) el.textContent = '';
  }, 4000);
}

// بارگذاری اولیه
if (SHEET_URL) {
  loadData();
}
</script>

</body>
</html>
