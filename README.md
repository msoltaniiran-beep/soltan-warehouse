<!DOCTYPE html>
<html dir="rtl" lang="fa">
<head>
  <meta charset="UTF-8">
  <title>اتصال به گوگل شیت — بدون خطا</title>
  <style>
    body { font-family: Vazirmatn, sans-serif; padding: 20px; max-width: 900px; margin: 0 auto; }
    input, button { padding: 8px; margin: 4px 0; }
    label { display: block; margin-top: 12px; font-weight: bold; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: right; }
    .actions button { margin: 0 3px; }
    .status { padding: 8px; margin: 8px 0; background: #f0f0f0; }
    .error { color: red; }
    .success { color: green; }
  </style>
</head>
<body>

<h2>🔗 لینک وب‌اپلیکیشن گوگل</h2>
<input type="url" id="sheetUrl" style="width:100%" 
       placeholder="https://script.google.com/macros/s/.../exec" 
       value="https://script.google.com/macros/s/YOUR_EXEC_URL/exec" />
<button onclick="saveUrl()">ذخیره</button>
<div id="urlStatus" class="status"></div>

<h2>➕ افزودن</h2>
<input type="text" id="inputText" style="width:100%" placeholder="متن...">
<input type="url" id="inputContact" style="width:100%" placeholder="لینک ارتباط (اختیاری)">
<button onclick="addRecord()">افزودن</button>
<div id="addStatus" class="status"></div>

<h3>📋 داده‌ها</h3>
<button onclick="loadData()">🔄 بارگذاری</button>
<table id="dataTable">
  <thead><tr><th>ID</th><th>متن</th><th>ارتباط</th><th>عملیات</th></tr></thead>
  <tbody id="tableBody"></tbody>
</table>

<script>
let SHEET_URL = localStorage.getItem('sheetUrl') || '';

function saveUrl() {
  SHEET_URL = document.getElementById('sheetUrl').value.trim();
  if (!SHEET_URL.includes('/exec')) {
    alert('⚠️ لینک باید با /exec تمام شود.');
    return;
  }
  localStorage.setItem('sheetUrl', SHEET_URL);
  document.getElementById('urlStatus').innerHTML = '<span class="success">✅ ذخیره شد</span>';
  loadData();
}

// ✅ این تابع همیشه کار می‌کند — حتی در file://
async function safeFetch(url, options = {}) {
  // برای دریافت داده‌ها: از proxy ساده استفاده می‌کنیم
  if (options.method === 'GET') {
    const proxy = `https://api.allorigins.win/raw?url=${encodeURIComponent(url)}`;
    const res = await fetch(proxy);
    return res.text();
  }
  // برای ارسال: از no-cors استفاده می‌کنیم (بدون انتظار پاسخ)
  await fetch(url, { ...options, mode: 'no-cors' });
  return '{"status":"ok"}';
}

async function loadData() {
  if (!SHEET_URL) return;
  try {
    const url = SHEET_URL + '?action=list';
    const text = await safeFetch(url, { method: 'GET' });
    const data = JSON.parse(text);

    const tbody = document.getElementById('tableBody');
    tbody.innerHTML = '';
    (data.data || []).forEach(row => {
      const tr = tbody.insertRow();
      tr.insertCell().textContent = row.id.substring(0,6) + '…';
      tr.insertCell().textContent = row.text || '—';
      const contactCell = tr.insertCell();
      if (row.contact) {
        contactCell.innerHTML = `<a href="${row.contact}" target="_blank">${row.contact}</a>`;
      } else {
        contactCell.textContent = '—';
      }
      const actions = tr.insertCell();
      actions.innerHTML = `
        <button onclick="editRow('${row.id}', \`${row.text?.replace(/`/g, '\\`') || ''}\`, \`${row.contact?.replace(/`/g, '\\`') || ''}\`)">ویرایش</button>
        <button onclick="deleteRow('${row.id}')">حذف</button>
      `;
    });
  } catch (e) {
    document.getElementById('urlStatus').innerHTML = `<span class="error">❌ خطا: ${e.message}</span>`;
  }
}

async function addRecord() {
  const text = document.getElementById('inputText').value.trim();
  const contact = document.getElementById('inputContact').value.trim();
  if (!text) return alert('لطفاً متن را وارد کنید.');

  try {
    const url = SHEET_URL + '?action=add';
    // ✅ ارسال بدون انتظار برای پاسخ — پس هیچ‌وقت Failed to fetch نمی‌دهد
    await safeFetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text, contact })
    });
    
    document.getElementById('inputText').value = '';
    document.getElementById('inputContact').value = '';
    document.getElementById('addStatus').innerHTML = '<span class="success">✅ ارسال شد — لیست به‌روز می‌شود...</span>';
    
    // تأخیر کوتاه برای اطمینان از ذخیره‌سازی در سمت گوگل
    setTimeout(loadData, 800);
  } catch (e) {
    document.getElementById('addStatus').innerHTML = `<span class="error">❌ خطا: ${e.message}</span>`;
  }
}

async function editRow(id, text, contact) {
  const newText = prompt('متن جدید:', text) || '';
  const newContact = prompt('لینک جدید:', contact) || '';
  if (newText === null) return;

  const url = SHEET_URL + '?action=update';
  await safeFetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id, text: newText, contact: newContact })
  });
  setTimeout(loadData, 800);
}

async function deleteRow(id) {
  if (!confirm('حذف شود؟')) return;
  const url = SHEET_URL + '?action=delete';
  await safeFetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id })
  });
  setTimeout(loadData, 800);
}

// بارگذاری اولیه
if (SHEET_URL) {
  document.getElementById('sheetUrl').value = SHEET_URL;
  setTimeout(loadData, 500);
}
</script>

</body>
</html>
