<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Billing — Simple Web App</title>
  <link rel="stylesheet" href="css/style.css" />
</head>
<body>
  <header class="topbar">
    <h1>Bhavik Digital — Billing</h1>
    <nav>
      <button id="nav-new">New Invoice</button>
      <button id="nav-history">History</button>
      <button id="nav-customers">Customers</button>
    </nav>
  </header>

  <main class="container">
    <!-- NEW INVOICE -->
    <section id="page-new" class="page">
      <div class="panel">
        <div class="row">
          <div>
            <label>Customer name</label>
            <input id="cust-name" placeholder="Customer name" />
          </div>
          <div>
            <label>Invoice No.</label>
            <input id="inv-number" placeholder="Auto or custom" />
          </div>
          <div>
            <label>Invoice Date</label>
            <input id="inv-date" type="date" />
          </div>
        </div>

        <hr/>

        <div class="row">
          <input id="item-name" placeholder="Item name" />
          <input id="item-price" type="number" placeholder="Price" />
          <input id="item-qty" type="number" placeholder="Qty" value="1" />
          <button id="btn-add-item">Add</button>
        </div>

        <table id="items-table">
          <thead>
            <tr><th>#</th><th>Item</th><th>Price</th><th>Qty</th><th>Total</th><th></th></tr>
          </thead>
          <tbody></tbody>
          <tfoot>
            <tr><td colspan="4" class="right">Subtotal</td><td id="subtotal">0.00</td><td></td></tr>
            <tr><td colspan="4" class="right">GST (%)</td><td>
              <input id="gst-rate" type="number" value="18" min="0" style="width:70px"/>
            </td><td></td></tr>
            <tr><td colspan="4" class="right">Total</td><td id="grandtotal">0.00</td><td></td></tr>
          </tfoot>
        </table>

        <div class="actions">
          <button id="btn-save-inv">Save Invoice</button>
          <button id="btn-print">Print / Download</button>
          <button id="btn-clear">Clear</button>
        </div>
      </div>
    </section>

    <!-- HISTORY -->
    <section id="page-history" class="page hidden">
      <div class="panel">
        <h2>Invoice History</h2>
        <div id="history-list"></div>
      </div>
    </section>

    <!-- CUSTOMERS -->
    <section id="page-customers" class="page hidden">
      <div class="panel">
        <h2>Customers</h2>
        <div class="row">
          <input id="new-cust-name" placeholder="Customer name" />
          <input id="new-cust-email" placeholder="Email (optional)" />
          <button id="btn-add-customer">Add Customer</button>
        </div>
        <ul id="cust-list"></ul>
      </div>
    </section>

    <!-- INVOICE PREVIEW (modal-like) -->
    <div id="preview-modal" class="modal hidden">
      <div class="modal-inner">
        <button id="close-preview" class="close">✕</button>
        <div id="preview-content"></div>
        <div class="actions">
          <button id="preview-print">Print</button>
          <button id="download-pdf">Download (Print to PDF)</button>
        </div>
      </div>
    </div>
  </main>

  <footer class="footer">Built for demo • Save to browser storage</footer>

  <script src="js/app.js"></script>
</body>
</html>



                  :root{
  --c-bg: #FFF7DD;
  --c-accent: #A0B5C5;
  --c-dark: #4B352A;
  --c-mid: #80A1BA;
  --card: #fff;
}
*{box-sizing:border-box}
body{
  margin:0;font-family:Inter,Segoe UI,Arial,sans-serif;background:var(--c-bg);color:var(--c-dark);
}
.topbar{background:var(--c-mid);color:white;padding:14px 18px;display:flex;justify-content:space-between;align-items:center}
.topbar h1{margin:0;font-size:18px}
.topbar nav button{margin-left:8px;background:var(--c-accent);border:0;padding:8px 10px;border-radius:6px;cursor:pointer}
.container{max-width:1000px;margin:18px auto;padding:8px}
.panel{background:var(--card);padding:16px;border-radius:10px;box-shadow:0 6px 20px rgba(0,0,0,0.06)}
.row{display:flex;gap:10px;margin-bottom:10px;flex-wrap:wrap}
.row > *{flex:1}
input[type="number"], input[type="date"], input[type="text"], input{padding:8px;border-radius:6px;border:1px solid #ddd}
#items-table{width:100%;border-collapse:collapse;margin-top:10px}
#items-table th,#items-table td{border:1px solid #eee;padding:8px;text-align:center}
.right{text-align:right;padding-right:12px}
.actions{display:flex;justify-content:flex-end;gap:8px;margin-top:12px}
button{background:var(--c-accent);border:0;color:#333;padding:8px 12px;border-radius:8px;cursor:pointer}
button:hover{opacity:0.95}
.hidden{display:none}
.footer{margin:18px;text-align:center;color:#666}

/* modal */
.modal{position:fixed;inset:0;background:rgba(0,0,0,0.35);display:flex;align-items:center;justify-content:center;padding:12px}
.modal-inner{background:white;padding:18px;border-radius:8px;max-width:760px;min-width:320px;max-height:80vh;overflow:auto;position:relative}
.close{position:absolute;right:10px;top:10px;border:0;background:#eee;padding:6px;border-radius:6px;cursor:pointer}



// Simple billing app using localStorage
(function(){
  // DOM refs
  const navNew = document.getElementById('nav-new');
  const navHistory = document.getElementById('nav-history');
  const navCustomers = document.getElementById('nav-customers');
  const pages = { new: document.getElementById('page-new'), history: document.getElementById('page-history'), customers: document.getElementById('page-customers') };

  const custName = document.getElementById('cust-name');
  const invNumber = document.getElementById('inv-number');
  const invDate = document.getElementById('inv-date');
  const itemName = document.getElementById('item-name');
  const itemPrice = document.getElementById('item-price');
  const itemQty = document.getElementById('item-qty');
  const btnAddItem = document.getElementById('btn-add-item');
  const itemsTableBody = document.querySelector('#items-table tbody');
  const subtotalEl = document.getElementById('subtotal');
  const gstRateEl = document.getElementById('gst-rate');
  const grandtotalEl = document.getElementById('grandtotal');
  const btnSaveInv = document.getElementById('btn-save-inv');
  const btnPrint = document.getElementById('btn-print');
  const btnClear = document.getElementById('btn-clear');

  const historyList = document.getElementById('history-list');

  const custList = document.getElementById('cust-list');
  const newCustName = document.getElementById('new-cust-name');
  const newCustEmail = document.getElementById('new-cust-email');
  const btnAddCustomer = document.getElementById('btn-add-customer');

  const previewModal = document.getElementById('preview-modal');
  const previewContent = document.getElementById('preview-content');
  const closePreview = document.getElementById('close-preview');
  const previewPrint = document.getElementById('preview-print');

  // state
  let items = [];
  let invoices = JSON.parse(localStorage.getItem('invoices') || '[]');
  let customers = JSON.parse(localStorage.getItem('customers') || '[]');

  // nav
  function show(pageKey){
    Object.values(pages).forEach(p => p.classList.add('hidden'));
    if(pageKey === 'new') pages.new.classList.remove('hidden');
    if(pageKey === 'history') { pages.history.classList.remove('hidden'); renderHistory(); }
    if(pageKey === 'customers') { pages.customers.classList.remove('hidden'); renderCustomers(); }
  }
  navNew.addEventListener('click', ()=> show('new'));
  navHistory.addEventListener('click', ()=> show('history'));
  navCustomers.addEventListener('click', ()=> show('customers'));

  // helpers
  function format(n){ return parseFloat(n || 0).toFixed(2); }

  function renderItems(){
    itemsTableBody.innerHTML = '';
    let subtotal = 0;
    items.forEach((it, i) => {
      const tr = document.createElement('tr');
      const total = it.price * it.qty;
      subtotal += total;
      tr.innerHTML = `<td>${i+1}</td><td>${it.name}</td><td>${format(it.price)}</td><td>${it.qty}</td><td>${format(total)}</td>
        <td><button data-i="${i}" class="remove-item">Remove</button></td>`;
      itemsTableBody.appendChild(tr);
    });
    subtotalEl.innerText = format(subtotal);
    const gst = subtotal * (Number(gstRateEl.value || 0) / 100);
    grandtotalEl.innerText = format(subtotal + gst);
    attachRemoveHandlers();
  }

  function attachRemoveHandlers(){
    document.querySelectorAll('.remove-item').forEach(btn => {
      btn.onclick = () => {
        const idx = Number(btn.dataset.i);
        items.splice(idx,1);
        renderItems();
      };
    });
  }

  // add item
  btnAddItem.addEventListener('click', ()=>{
    const name = itemName.value.trim();
    const price = Number(itemPrice.value);
    const qty = Number(itemQty.value) || 1;
    if(!name || !price) return alert('Enter item name and price');
    items.push({name, price, qty});
    itemName.value = ''; itemPrice.value = ''; itemQty.value = 1;
    renderItems();
  });

  // GST change
  gstRateEl.addEventListener('input', renderItems);

  // save invoice
  btnSaveInv.addEventListener('click', ()=>{
    if(!custName.value.trim()) return alert('Add customer name');
    if(items.length === 0) return alert('Add at least one item');
    const id = Date.now();
    const invNo = invNumber.value.trim() || `INV-${id}`;
    const date = invDate.value || new Date().toISOString().slice(0,10);
    const subtotal = Number(subtotalEl.innerText);
    const gstRate = Number(gstRateEl.value) || 0;
    const gstAmount = subtotal * (gstRate/100);
    const total = subtotal + gstAmount;

    const invoice = {
      id, invNo, date, customer: custName.value.trim(),
      items, subtotal, gstRate, gstAmount, total
    };
    invoices.unshift(invoice);
    localStorage.setItem('invoices', JSON.stringify(invoices));
    items = [];
    renderItems();
    invNumber.value = '';
    alert('Invoice saved to local history');
    show('history');
  });

  // print current invoice (quick)
  btnPrint.addEventListener('click', ()=>{
    openPreview(buildInvoiceHtml({
      invNo: invNumber.value || 'Preview',
      date: invDate.value || (new Date().toISOString().slice(0,10)),
      customer: custName.value || 'Customer',
      items: items,
      gstRate: Number(gstRateEl.value || 0),
      subtotal: Number(subtotalEl.innerText)
    }));
  });

  // clear
  btnClear.addEventListener('click', ()=>{
    if(confirm('Clear all items?')){ items = []; renderItems(); }
  });

  // history
  function renderHistory(){
    if(!invoices.length){ historyList.innerHTML = '<p>No invoices saved yet.</p>'; return; }
    historyList.innerHTML = '';
    invoices.forEach(inv => {
      const el = document.createElement('div');
      el.className = 'panel';
      el.style.marginBottom = '8px';
      el.innerHTML = `<div style="display:flex;justify-content:space-between;align-items:center">
        <div><strong>${inv.invNo}</strong> • ${inv.customer} • ${inv.date}</div>
        <div>
          <button data-id="${inv.id}" class="view-inv">View</button>
          <button data-id="${inv.id}" class="del-inv">Delete</button>
        </div>
      </div>`;
      historyList.appendChild(el);
    });
    // handlers
    document.querySelectorAll('.view-inv').forEach(b=>{
      b.onclick = () => {
        const id = Number(b.dataset.id);
        const inv = invoices.find(i=>i.id===id);
        openPreview(buildInvoiceHtml(inv));
      };
    });
    document.querySelectorAll('.del-inv').forEach(b=>{
      b.onclick = () => {
        const id = Number(b.dataset.id);
        if(!confirm('Delete invoice?')) return;
        invoices = invoices.filter(i=>i.id!==id);
        localStorage.setItem('invoices', JSON.stringify(invoices));
        renderHistory();
      };
    });
  }

  // customers
  function renderCustomers(){
    custList.innerHTML = '';
    if(!customers.length){ custList.innerHTML = '<li>No customers</li>'; return; }
    customers.forEach((c, i) => {
      const li = document.createElement('li');
      li.innerHTML = `${c.name} ${c.email?`• ${c.email}`:''} <button data-i="${i}" class="use-cust">Use</button>`;
      custList.appendChild(li);
    });
    document.querySelectorAll('.use-cust').forEach(b=>{
      b.onclick = () => {
        const idx = Number(b.dataset.i);
        custName.value = customers[idx].name;
        show('new');
      };
    });
  }
  btnAddCustomer.addEventListener('click', ()=>{
    const name = newCustName.value.trim();
    const email = newCustEmail.value.trim();
    if(!name) return alert('Enter name');
    customers.push({name,email});
    localStorage.setItem('customers', JSON.stringify(customers));
    newCustName.value=''; newCustEmail.value='';
    renderCustomers();
  });

  // modal preview helpers
  function buildInvoiceHtml(inv){
    // inv could be an invoice saved or build from current state
    const itemsHtml = (inv.items || []).map((it, idx)=>`<tr><td>${idx+1}</td><td>${it.name}</td><td>${format(it.price)}</td><td>${it.qty||''}</td><td>${format((it.price||it.total||0)* (it.qty||1))}</td></tr>`).join('');
    const gstAmount = inv.gstAmount !== undefined ? format(inv.gstAmount) : format((inv.subtotal||0)*(inv.gstRate||0)/100);
    const total = inv.total !== undefined ? format(inv.total) : format((inv.subtotal||0) + Number(gstAmount));
    return `
      <div>
        <h2>Invoice: ${inv.invNo}</h2>
        <div><strong>Date:</strong> ${inv.date}</div>
        <div><strong>Customer:</strong> ${inv.customer}</div>
        <table style="width:100%;border-collapse:collapse;margin-top:10px;border:1px solid #ddd">
          <thead><tr><th>#</th><th>Item</th><th>Price</th><th>Qty</th><th>Total</th></tr></thead>
          <tbody>${itemsHtml}</tbody>
          <tfoot>
            <tr><td colspan="3"></td><td>Subtotal</td><td>${format(inv.subtotal||0)}</td></tr>
            <tr><td colspan="3"></td><td>GST (${inv.gstRate||0}%)</td><td>${gstAmount}</td></tr>
            <tr><td colspan="3"></td><td><strong>Total</strong></td><td><strong>${total}</strong></td></tr>
          </tfoot>
        </table>
      </div>`;
  }

  function openPreview(html){
    previewContent.innerHTML = html;
    previewModal.classList.remove('hidden');
  }
  closePreview.onclick = ()=> previewModal.classList.add('hidden');
  previewPrint.onclick = ()=> { previewModal.classList.add('hidden'); setTimeout(()=> window.print(), 200); };

  // init
  (function init(){
    invDate.value = new Date().toISOString().slice(0,10);
    renderItems(); renderCustomers();
    show('new');
  })();

})();


