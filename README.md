<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Bhavik Digital — Billing</title>

<style>
:root{
  --c-bg:#FFF7DD;
  --c-accent:#A0B5C5;
  --c-dark:#4B352A;
  --c-mid:#80A1BA;
  --card:#fff;
}

*{box-sizing:border-box;}

body{
  margin:0;
  font-family:Inter,Segoe UI,Arial,sans-serif;
  background:var(--c-bg);
  color:var(--c-dark);
}

.topbar{
  background:var(--c-mid);
  color:white;
  padding:14px 18px;
  display:flex;
  justify-content:space-between;
  align-items:center;
}

.topbar h1{
  margin:0;
  font-size:20px;
}

nav button{
  margin-left:8px;
  background:var(--c-accent);
  border:0;
  padding:8px 12px;
  border-radius:6px;
  cursor:pointer;
}

.container{
  max-width:1000px;
  margin:18px auto;
  padding:10px;
}

.panel{
  background:var(--card);
  padding:16px;
  border-radius:10px;
  box-shadow:0 6px 20px rgba(0,0,0,0.06);
}

.row{
  display:flex;
  gap:10px;
  margin-bottom:10px;
  flex-wrap:wrap;
}

.row > *{
  flex:1;
}

input, select{
  padding:8px;
  border-radius:6px;
  border:1px solid #ccc;
}

button{
  background:var(--c-accent);
  border:0;
  color:#222;
  padding:8px 12px;
  border-radius:6px;
  cursor:pointer;
}

button:hover{opacity:0.9;}

#items-table{
  width:100%;
  border-collapse:collapse;
  margin-top:10px;
}

#items-table th,#items-table td{
  border:1px solid #eee;
  padding:8px;
  text-align:center;
}

.right{text-align:right;padding-right:12px;}

.hidden{display:none;}

.actions{
  display:flex;
  justify-content:flex-end;
  gap:10px;
  margin-top:10px;
}

.footer{
  text-align:center;
  color:#666;
  padding:12px;
  font-size:0.9em;
}

/* Modal */
.modal{
  position:fixed;
  inset:0;
  background:rgba(0,0,0,0.35);
  display:flex;
  justify-content:center;
  align-items:center;
  padding:12px;
}
.modal-inner{
  background:white;
  padding:18px;
  border-radius:8px;
  max-width:760px;
  width:100%;
  max-height:80vh;
  overflow:auto;
  position:relative;
}
.close{
  position:absolute;
  top:10px;
  right:10px;
  background:#eee;
  border:0;
  border-radius:6px;
  padding:6px;
  cursor:pointer;
}
</style>
</head>

<body>
<div class="topbar">
  <h1>Bhavik Digital — Billing</h1>
  <nav>
    <button id="nav-new">🧾 New Invoice</button>
    <button id="nav-history">📜 History</button>
    <button id="nav-customers">👥 Customers</button>
  </nav>
</div>

<div class="container">

  <!-- NEW INVOICE PAGE -->
  <div id="page-new" class="panel">
    <div class="row">
      <input id="cust-name" placeholder="Customer name">
      <input id="inv-number" placeholder="Invoice No. (auto)">
      <input type="date" id="inv-date">
    </div>

    <hr>
    <div class="row">
      <input id="item-name" placeholder="Item name">
      <input type="number" id="item-price" placeholder="Price ₹">
      <input type="number" id="item-qty" placeholder="Qty" value="1">
      <button id="btn-add-item">Add</button>
    </div>

    <table id="items-table">
      <thead>
        <tr><th>#</th><th>Item</th><th>Price</th><th>Qty</th><th>Total</th><th></th></tr>
      </thead>
      <tbody></tbody>
      <tfoot>
        <tr><td colspan="4" class="right">Subtotal ₹</td><td id="subtotal">0.00</td></tr>
        <tr><td colspan="4" class="right">GST %</td><td><input id="gst-rate" type="number" value="18" style="width:60px;"></td></tr>
        <tr><td colspan="4" class="right">Total ₹</td><td id="grandtotal">0.00</td></tr>
      </tfoot>
    </table>

    <div class="actions">
      <button id="btn-save-inv">💾 Save Invoice</button>
      <button id="btn-print">🖨️ Print / Preview</button>
      <button id="btn-clear">🗑️ Clear</button>
    </div>
  </div>

  <!-- HISTORY PAGE -->
  <div id="page-history" class="panel hidden">
    <h3>Invoice History</h3>
    <div id="history-list"></div>
  </div>

  <!-- CUSTOMERS PAGE -->
  <div id="page-customers" class="panel hidden">
    <h3>Customers</h3>
    <div class="row">
      <input id="new-cust-name" placeholder="Customer name">
      <input id="new-cust-email" placeholder="Email (optional)">
      <button id="btn-add-customer">Add Customer</button>
    </div>
    <ul id="cust-list"></ul>
  </div>

</div>

<div class="footer">
  © 2025 Bhavik Digital | Saved locally (Browser Storage)
</div>

<!-- PREVIEW MODAL -->
<div id="preview-modal" class="modal hidden">
  <div class="modal-inner">
    <button class="close" id="close-preview">✕</button>
    <div id="preview-content"></div>
  </div>
</div>


<script>
(function(){
  const pages = {
    new: document.getElementById('page-new'),
    history: document.getElementById('page-history'),
    customers: document.getElementById('page-customers')
  };
  const navNew=document.getElementById('nav-new'),
        navHistory=document.getElementById('nav-history'),
        navCustomers=document.getElementById('nav-customers');

  function show(page){
    Object.values(pages).forEach(p=>p.classList.add('hidden'));
    pages[page].classList.remove('hidden');
    if(page==='history') renderHistory();
    if(page==='customers') renderCustomers();
  }
  navNew.onclick=()=>show('new');
  navHistory.onclick=()=>show('history');
  navCustomers.onclick=()=>show('customers');

  // DOM refs
  const custName=document.getElementById('cust-name'),
        invNumber=document.getElementById('inv-number'),
        invDate=document.getElementById('inv-date'),
        itemName=document.getElementById('item-name'),
        itemPrice=document.getElementById('item-price'),
        itemQty=document.getElementById('item-qty'),
        btnAddItem=document.getElementById('btn-add-item'),
        itemsTableBody=document.querySelector('#items-table tbody'),
        subtotalEl=document.getElementById('subtotal'),
        gstRateEl=document.getElementById('gst-rate'),
        grandtotalEl=document.getElementById('grandtotal'),
        btnSaveInv=document.getElementById('btn-save-inv'),
        btnPrint=document.getElementById('btn-print'),
        btnClear=document.getElementById('btn-clear');

  const historyList=document.getElementById('history-list');
  const custList=document.getElementById('cust-list'),
        newCustName=document.getElementById('new-cust-name'),
        newCustEmail=document.getElementById('new-cust-email'),
        btnAddCustomer=document.getElementById('btn-add-customer');

  const previewModal=document.getElementById('preview-modal'),
        previewContent=document.getElementById('preview-content'),
        closePreview=document.getElementById('close-preview'),
        previewPrint=document.getElementById('preview-print');

  let items=[], invoices=JSON.parse(localStorage.getItem('invoices')||'[]'),
      customers=JSON.parse(localStorage.getItem('customers')||'[]');

  function format(n){return parseFloat(n||0).toFixed(2);}

  function renderItems(){
    itemsTableBody.innerHTML='';
    let subtotal=0;
    items.forEach((it,i)=>{
      const tr=document.createElement('tr');
      const total=it.price*it.qty;
      subtotal+=total;
      tr.innerHTML=`<td>${i+1}</td><td>${it.name}</td><td>${format(it.price)}</td><td>${it.qty}</td><td>${format(total)}</td><td><button onclick="removeItem(${i})">❌</button></td>`;
      itemsTableBody.appendChild(tr);
    });
    subtotalEl.innerText=format(subtotal);
    const gst=subtotal*(Number(gstRateEl.value)/100);
    grandtotalEl.innerText=format(subtotal+gst);
  }

  window.removeItem=function(i){
    items.splice(i,1);
    renderItems();
  }

  btnAddItem.onclick=()=>{
    const name=itemName.value.trim(),
          price=Number(itemPrice.value),
          qty=Number(itemQty.value)||1;
    if(!name||!price){alert("Enter valid item and price");return;}
    items.push({name,price,qty});
    itemName.value='';itemPrice.value='';itemQty.value=1;
    renderItems();
  };

  gstRateEl.oninput=renderItems;

  btnSaveInv.onclick=()=>{
    if(!custName.value.trim()){alert("Enter customer name");return;}
    if(!items.length){alert("Add at least one item");return;}
    const id=Date.now();
    const invoice={
      id,
      invNo:invNumber.value||("INV-"+id),
      date:invDate.value||new Date().toISOString().slice(0,10),
      customer:custName.value,
      items,
      subtotal:Number(subtotalEl.innerText),
      gstRate:Number(gstRateEl.value),
      total:Number(grandtotalEl.innerText)
    };
    invoices.unshift(invoice);
    localStorage.setItem('invoices',JSON.stringify(invoices));
    alert("Invoice saved!");
    items=[];
    renderItems();
    show('history');
  };

function openPreview(html){
  previewContent.innerHTML=html;
  previewModal.classList.remove('hidden');
  setTimeout(()=>window.print(),400); // auto open print after showing
}


  btnClear.onclick=()=>{
    if(confirm("Clear all items?")){items=[];renderItems();}
  };

  function renderHistory(){
    historyList.innerHTML=invoices.length? '' : '<p>No invoices saved yet.</p>';
    invoices.forEach(inv=>{
      const div=document.createElement('div');
      div.className='panel';
      div.style.marginBottom='8px';
      div.innerHTML=`<div style="display:flex;justify-content:space-between">
      <div><strong>${inv.invNo}</strong> • ${inv.customer} • ${inv.date}</div>
      <div><button onclick='viewInvoice(${inv.id})'>View</button>
      <button onclick='deleteInvoice(${inv.id})'>Delete</button></div></div>`;
      historyList.appendChild(div);
    });
  }

  window.viewInvoice=function(id){
    const inv=invoices.find(i=>i.id===id);
    openPreview(buildInvoiceHtml(inv));
  }
  window.deleteInvoice=function(id){
    if(!confirm("Delete invoice?"))return;
    invoices=invoices.filter(i=>i.id!==id);
    localStorage.setItem('invoices',JSON.stringify(invoices));
    renderHistory();
  }

  function renderCustomers(){
    custList.innerHTML='';
    if(!customers.length){custList.innerHTML='<li>No customers yet.</li>';return;}
    customers.forEach((c,i)=>{
      const li=document.createElement('li');
      li.innerHTML=`${c.name} ${c.email?('('+c.email+')'):''} <button onclick="useCustomer(${i})">Use</button>`;
      custList.appendChild(li);
    });
  }
  window.useCustomer=function(i){
    custName.value=customers[i].name;
    show('new');
  }

  btnAddCustomer.onclick=()=>{
    const name=newCustName.value.trim(), email=newCustEmail.value.trim();
    if(!name){alert("Enter name");return;}
    customers.push({name,email});
    localStorage.setItem('customers',JSON.stringify(customers));
    newCustName.value=''; newCustEmail.value='';
    renderCustomers();
  };

  function buildInvoiceHtml(inv){
    let rows=inv.items.map((it,i)=>`<tr><td>${i+1}</td><td>${it.name}</td><td>${format(it.price)}</td><td>${it.qty}</td><td>${format(it.price*it.qty)}</td></tr>`).join('');
    return `<h2>${inv.invNo}</h2>
    <div>Date: ${inv.date}</div><div>Customer: ${inv.customer}</div>
    <table style="width:100%;border-collapse:collapse;margin-top:10px" border="1">
      <thead><tr><th>#</th><th>Item</th><th>Price</th><th>Qty</th><th>Total</th></tr></thead>
      <tbody>${rows}</tbody>
      <tfoot>
        <tr><td colspan="4" class="right">Subtotal</td><td>${format(inv.subtotal)}</td></tr>
        <tr><td colspan="4" class="right">GST (${inv.gstRate}%)</td><td>${format(inv.subtotal*inv.gstRate/100)}</td></tr>
        <tr><td colspan="4" class="right">Total</td><td>${format(inv.total)}</td></tr>
      </tfoot>
    </table>`;
  }

  function openPreview(html){
    previewContent.innerHTML=html;
    previewModal.classList.remove('hidden');
  }
  closePreview.onclick=()=>previewModal.classList.add('hidden');
 const previewPrint=document.getElementById('preview-print');


  (function init(){
    invDate.value=new Date().toISOString().slice(0,10);
    renderItems();
    renderCustomers();
    show('new');
  })();

})();
</script>
</body>
</html>
