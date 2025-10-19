<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Billing Software - HTML CSS JS</title>
<style>
  :root {
    --primary: #CBDCEB;
    --accent: #9DBDFF;
    --dark: #001F3F;
    --bg: #F8FAFC;
    --white: #FFFFFF;
  }

  body {
    margin: 0;
    padding: 0;
    background: var(--bg);
    font-family: "Segoe UI", sans-serif;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
  }

  .app-frame {
    width: 90%;
    max-width: 900px;
    background: var(--white);
    box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
    border-radius: 16px;
    overflow: hidden;
  }

  header {
    background: var(--dark);
    color: var(--white);
    padding: 1.2rem;
    text-align: center;
    font-size: 1.5rem;
    letter-spacing: 0.5px;
  }

  .content {
    padding: 1.5rem;
  }

  .inputs {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
    justify-content: space-between;
    margin-bottom: 1rem;
  }

  input {
    flex: 1;
    min-width: 150px;
    padding: 8px 10px;
    border: 1px solid #ccc;
    border-radius: 6px;
  }

  button {
    background: var(--accent);
    border: none;
    color: var(--dark);
    padding: 8px 14px;
    border-radius: 8px;
    cursor: pointer;
    font-weight: bold;
    transition: background 0.2s;
  }

  button:hover {
    background: var(--primary);
  }

  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 1rem;
    border-radius: 10px;
    overflow: hidden;
  }

  th, td {
    border: 1px solid #eee;
    padding: 8px;
    text-align: center;
  }

  th {
    background: var(--primary);
  }

  tfoot td {
    font-weight: bold;
    background: #f3f6f9;
  }

  .actions {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 1rem;
  }

  .footer-note {
    text-align: center;
    color: #777;
    font-size: 0.9rem;
    padding: 1rem;
    background: #f9fafb;
  }

  @media print {
    .inputs, .actions, .footer-note {
      display: none !important;
    }
    body {
      background: #fff;
    }
    header {
      color: #000;
      background: #fff;
      border-bottom: 2px solid #000;
    }
  }
</style>
</head>
<body>

<div class="app-frame">
  <header>🧾 Simple Billing Software</header>

  <div class="content">
    <div class="inputs">
      <input type="text" id="itemName" placeholder="Item name">
      <input type="number" id="itemPrice" placeholder="Price ₹">
      <input type="number" id="itemQty" placeholder="Quantity">
      <button onclick="addItem()">Add Item ➕</button>
    </div>

    <table id="invoiceTable">
      <thead>
        <tr>
          <th>#</th>
          <th>Item</th>
          <th>Price (₹)</th>
          <th>Qty</th>
          <th>Total (₹)</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody id="invoiceBody"></tbody>
      <tfoot>
        <tr>
          <td colspan="4" align="right">Grand Total (₹):</td>
          <td id="grandTotal">0.00</td>
          <td></td>
        </tr>
      </tfoot>
    </table>

    <div class="actions">
      <button onclick="printInvoice()">🖨️ Print / Save PDF</button>
      <button onclick="clearInvoice()">🗑️ Clear All</button>
    </div>
  </div>

  <div class="footer-note">
    © 2025 My Billing Software | Built with ❤️ using HTML, CSS & JS
  </div>
</div>

<script>
  let items = [];

  function addItem() {
    const name = document.getElementById('itemName').value.trim();
    const price = parseFloat(document.getElementById('itemPrice').value);
    const qty = parseInt(document.getElementById('itemQty').value);

    if (!name || isNaN(price) || isNaN(qty)) {
      alert("Please fill all fields correctly!");
      return;
    }

    const total = price * qty;
    items.push({ name, price, qty, total });
    renderTable();

    document.getElementById('itemName').value = '';
    document.getElementById('itemPrice').value = '';
    document.getElementById('itemQty').value = '';
  }

  function renderTable() {
    const body = document.getElementById('invoiceBody');
    body.innerHTML = '';
    let grand = 0;

    items.forEach((item, index) => {
      grand += item.total;
      body.innerHTML += `
        <tr>
          <td>${index + 1}</td>
          <td>${item.name}</td>
          <td>${item.price.toFixed(2)}</td>
          <td>${item.qty}</td>
          <td>${item.total.toFixed(2)}</td>
          <td><button onclick="removeItem(${index})">❌</button></td>
        </tr>`;
    });

    document.getElementById('grandTotal').innerText = grand.toFixed(2);
  }

  function removeItem(index) {
    items.splice(index, 1);
    renderTable();
  }

  function clearInvoice() {
    if (confirm("Are you sure you want to clear all items?")) {
      items = [];
      renderTable();
    }
  }

  function printInvoice() {
    window.print();
  }
</script>
</body>
</html>
