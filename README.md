import tkinter as tk
from tkinter import messagebox, simpledialog, filedialog, ttk, Toplevel
from datetime import datetime
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, Table, TableStyle
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib import colors as rl_colors
from PIL import Image, ImageTk
import time
import threading
import os

try:
    import cv2
    CAMERA_AVAILABLE = True
except Exception:
    CAMERA_AVAILABLE = False

try:
    import speech_recognition as sr
    MIC_AVAILABLE = True
except Exception:
    MIC_AVAILABLE = False

# -----------------------------
# Color palette (requested)
# -----------------------------
C1 = "#80A1BA"  # blue-gray
C2 = "#91C4C3"  # light cyan
C3 = "#FFF7DD"  # cream
C4 = "#043915"  # dark green
C5 = "#4B352A"  # brown
C6 = "#000000"  # black
ACCENT_BTN = "#A0B5C5"

# -----------------------------
# Data classes
# -----------------------------
class Customer:
    def __init__(self, name, email, phone):
        self.name = name
        self.email = email
        self.phone = phone
        self.id = id(self)

class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price
        self.id = id(self)

class Invoice:
    def __init__(self, customer, items, gst_rate, template, payment_mode="Cash", terms="Default Terms"):
        self.customer = customer
        self.items = items  # list of (Product, qty)
        self.gst_rate = gst_rate
        self.subtotal = sum(product.price * qty for product, qty in items)
        self.gst_amount = self.subtotal * (gst_rate / 100)
        self.total = self.subtotal + self.gst_amount
        self.paid = False
        self.date = datetime.now()
        self.template = template
        self.payment_mode = payment_mode
        self.terms = terms
        self.id = int(time.time() * 1000)  # stable-ish id for saving

# -----------------------------
# Permission manager (keeps as before)
# -----------------------------
class PermissionManager:
    def __init__(self):
        self.permissions = {
            "storage": False,
            "media": False,
            "camera": False,
            "microphone": False,
            "notifications": False
        }

    def request_permissions(self, root):
        dialog = Toplevel(root)
        dialog.title("Grant Permissions")
        dialog.configure(bg=C1)
        tk.Label(dialog, text="Grant permissions for a better experience:", font=("Arial", 14, "bold"), bg=C1, fg="#333").pack(pady=10)

        vars = {}
        for perm in self.permissions:
            var = tk.BooleanVar(value=self.permissions[perm])
            vars[perm] = var
            tk.Checkbutton(dialog, text=f"Allow {perm.capitalize()} Access", variable=var, bg=C1, fg="#333", font=("Arial", 10)).pack(anchor="w", padx=20)

        def submit():
            for perm, var in vars.items():
                self.permissions[perm] = var.get()
            dialog.destroy()

        tk.Button(dialog, text="Grant Selected", command=submit, bg=ACCENT_BTN, fg="#333", font=("Arial", 10)).pack(pady=10)
        dialog.transient(root)
        dialog.grab_set()
        root.wait_window(dialog)

    def check_permission(self, perm):
        return self.permissions.get(perm, False)

# -----------------------------
# Main application
# -----------------------------
class BillingApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Bhavik Mate powered By Bhavik digital")
        self.root.geometry("1000x700")
        self.root.configure(bg=C3)
        self.customers = []
        self.products = []
        self.invoices = []
        self.revenue = 0.0
        self.notifications = []
        self.perm_manager = PermissionManager()

        # Request permissions on startup
        self.perm_manager.request_permissions(root)

        # Load optional icons/images
        self.add_invoice_img = None
        try:
            if os.path.exists("add_invoice.png"):
                img = Image.open("add_invoice.png").resize((24, 24))
                self.add_invoice_img = ImageTk.PhotoImage(img)
        except Exception:
            self.add_invoice_img = None

        # Top three-line bar (using provided colors)
        self.create_three_line_bar()

        # Menu bar
        self.create_menu_bar()

        # Header and progress
        header_frame = tk.Frame(root, bg=C3)
        header_frame.pack(fill="x", padx=10, pady=(10,0))
        tk.Label(header_frame, text="Bhavik Mate powered By Bhavik digital", font=("Arial", 18, "bold"), bg=C3, fg=C5).pack(side="left")
        self.progress = ttk.Progressbar(header_frame, orient="horizontal", mode="determinate", length=200)
        self.progress.pack(side="right", padx=10)

        # Notebook for main sections
        self.notebook = ttk.Notebook(root)
        self.notebook.pack(fill="both", expand=True, padx=10, pady=10)

        # Tabs
        self.tab_home = tk.Frame(self.notebook, bg=C3)
        self.tab_dashboard = tk.Frame(self.notebook, bg=C3)
        self.tab_customers = tk.Frame(self.notebook, bg=C3)
        self.tab_create_invoice = tk.Frame(self.notebook, bg=C3)
        self.tab_history = tk.Frame(self.notebook, bg=C3)
        self.tab_settings = tk.Frame(self.notebook, bg=C3)

        self.notebook.add(self.tab_home, text="Home")
        self.notebook.add(self.tab_dashboard, text="Dashboard")
        self.notebook.add(self.tab_customers, text="Customers")
        self.notebook.add(self.tab_create_invoice, text="Create Invoice")
        self.notebook.add(self.tab_history, text="History")
        self.notebook.add(self.tab_settings, text="Settings")

        # Build contents for each tab
        self.build_home_tab()
        self.build_dashboard_tab()
        self.build_customers_tab()
        self.build_create_invoice_tab()
        self.build_history_tab()
        self.build_settings_tab()

        # Auto-check updates periodically
        self.root.after(300000, self.auto_check_updates)

    # -----------------------------
    # UI building helpers
    # -----------------------------
    def create_three_line_bar(self):
        top = tk.Frame(self.root, bg=C6)
        top.pack(fill="x")
        # three horizontal lines stacked
        tk.Frame(self.root, bg=C1, height=6).pack(fill="x")
        tk.Frame(self.root, bg=C2, height=6).pack(fill="x")
        tk.Frame(self.root, bg=C4, height=6).pack(fill="x")

    def create_menu_bar(self):
        menubar = tk.Menu(self.root)
        self.root.config(menu=menubar)
        # Add top-level menus
        menubar.add_command(label="Home", command=lambda: self.notebook.select(self.tab_home))
        menubar.add_command(label="Dashboard", command=lambda: self.notebook.select(self.tab_dashboard))
        menubar.add_command(label="Customers", command=lambda: self.notebook.select(self.tab_customers))
        menubar.add_command(label="History", command=lambda: self.notebook.select(self.tab_history))
        menubar.add_command(label="Settings", command=lambda: self.notebook.select(self.tab_settings))

    def create_button(self, parent, text, command, img=None):
        if img:
            btn = tk.Button(parent, text=text, image=img, compound="left", command=command, bg=ACCENT_BTN, fg="#333",
                            font=("Arial", 10), relief="flat", padx=8, pady=6)
        else:
            btn = tk.Button(parent, text=text, command=command, bg=ACCENT_BTN, fg="#333", font=("Arial", 10), relief="flat", padx=8, pady=6)
        btn.bind("<Enter>", lambda e: btn.config(bg="#90A5B5"))
        btn.bind("<Leave>", lambda e: btn.config(bg=ACCENT_BTN))
        return btn

    # -----------------------------
    # Tab: Home
    # -----------------------------
    def build_home_tab(self):
        frame = self.tab_home
        tk.Label(frame, text="Welcome to Bhavik Digital Billing", bg=C3, fg=C4, font=("Arial", 14, "bold")).pack(pady=20)
        tk.Label(frame, text="Use the menu or tabs to navigate.", bg=C3, fg="#333").pack()

    # -----------------------------
    # Tab: Dashboard
    # -----------------------------
    def build_dashboard_tab(self):
        frame = self.tab_dashboard
        tk.Label(frame, text="Dashboard", bg=C3, fg=C4, font=("Arial", 14, "bold")).pack(pady=6)
        stats_frame = tk.Frame(frame, bg=C3)
        stats_frame.pack(pady=10)
        self.rev_label = tk.Label(stats_frame, text=f"Revenue: ${self.revenue:.2f}", bg=C3, fg="#333", font=("Arial", 12))
        self.rev_label.pack()

    # -----------------------------
    # Tab: Customers
    # -----------------------------
    def build_customers_tab(self):
        frame = self.tab_customers
        header = tk.Frame(frame, bg=C3)
        header.pack(fill="x", pady=8)
        tk.Label(header, text="Customers", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(side="left", padx=10)
        add_btn = self.create_button(header, "Add Customer", self.add_customer)
        add_btn.pack(side="right", padx=10)
        self.customer_listbox = tk.Listbox(frame, height=15)
        self.customer_listbox.pack(fill="both", expand=True, padx=10, pady=10)

    # -----------------------------
    # Tab: Create Invoice
    # -----------------------------
    def build_create_invoice_tab(self):
        frame = self.tab_create_invoice
        left = tk.Frame(frame, bg=C3)
        left.pack(side="left", fill="y", padx=10, pady=10)

        right = tk.Frame(frame, bg=C3)
        right.pack(side="left", fill="both", expand=True, padx=10, pady=10)

        # Customer Details (search + add)
        tk.Label(left, text="Customer Details", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(anchor="w")
        self.cust_search_var = tk.StringVar()
        search_entry = tk.Entry(left, textvariable=self.cust_search_var, width=30)
        search_entry.pack(pady=4)
        tk.Button(left, text="Search", command=self.search_customer, bg=ACCENT_BTN).pack(pady=2)
        tk.Button(left, text="+ Add New Customer", command=self.add_customer, bg=ACCENT_BTN).pack(pady=4)

        # Invoice Type
        tk.Label(left, text="Invoice Type", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(anchor="w", pady=(10,0))
        self.invoice_type_var = tk.StringVar(value="GST")
        types = [("GST Invoice", "GST"), ("Non-GST Invoice", "NON_GST"), ("Estimate", "ESTIMATE")]
        for txt, val in types:
            tk.Radiobutton(left, text=txt, variable=self.invoice_type_var, value=val, bg=C3).pack(anchor="w")

        # Items area
        tk.Label(right, text="Items", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(anchor="w")
        self.items_tree = ttk.Treeview(right, columns=("name", "price", "qty"), show="headings", height=8)
        self.items_tree.heading("name", text="Name")
        self.items_tree.heading("price", text="Price")
        self.items_tree.heading("qty", text="Qty")
        self.items_tree.pack(fill="both", expand=True, pady=6)
        btn_frame = tk.Frame(right, bg=C3)
        btn_frame.pack(fill="x")
        self.create_button(btn_frame, "Add Item", self.add_product).pack(side="left", padx=4)
        tk.Button(btn_frame, text="Add Selected Product to Items", command=self.add_selected_product_to_items, bg=ACCENT_BTN).pack(side="left", padx=4)

        # Payment & Terms
        tk.Label(right, text="Payment & Terms", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(anchor="w", pady=(8,0))
        self.payment_mode_var = tk.StringVar(value="Cash")
        modes = [("Cash","Cash"), ("UPI","UPI"), ("Bank Transfer","Bank Transfer")]
        for txt, val in modes:
            tk.Radiobutton(right, text=txt, variable=self.payment_mode_var, value=val, bg=C3).pack(anchor="w")

        # Terms (simple)
        self.terms_var = tk.StringVar(value="Payment due within 30 days of invoice date.")
        tk.Label(right, text="Terms & Conditions", bg=C3).pack(anchor="w")
        tk.Entry(right, textvariable=self.terms_var, width=80).pack(fill="x")

        # Create/Preview
        action_frame = tk.Frame(right, bg=C3)
        action_frame.pack(fill="x", pady=10)
        gen_btn = self.create_button(action_frame, "Generate PDF", self.generate_pdf, img=self.add_invoice_img)
        gen_btn.pack(side="right", padx=6)
        self.create_button(action_frame, "Preview", self.preview_invoice).pack(side="right", padx=6)

        # Quick product list to select from when adding items
        tk.Label(left, text="Products", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(anchor="w", pady=(10,0))
        self.product_listbox_small = tk.Listbox(left, height=8)
        self.product_listbox_small.pack()

    def add_selected_product_to_items(self):
        sel = self.product_listbox_small.curselection()
        if not sel:
            messagebox.showerror("Error", "Select product from list")
            return
        idx = sel[0]
        product = self.products[idx]
        qty = simpledialog.askinteger("Quantity", "Enter quantity:", minvalue=1)
        if not qty:
            return
        self.items_tree.insert("", "end", values=(product.name, f"{product.price:.2f}", qty))

    # -----------------------------
    # Tab: History
    # -----------------------------
    def build_history_tab(self):
        frame = self.tab_history
        top = tk.Frame(frame, bg=C3)
        top.pack(fill="x", pady=6, padx=6)
        tk.Label(top, text="Invoice History", bg=C3, fg=C4, font=("Arial", 12, "bold")).pack(side="left")

        # Search toggle
        self.show_search = tk.BooleanVar(value=False)
        tk.Checkbutton(top, text="Show Search", variable=self.show_search, bg=C3, command=self.toggle_search).pack(side="right", padx=6)

        # Filters
        filter_frame = tk.Frame(frame, bg=C3)
        filter_frame.pack(fill="x", padx=6)
        tk.Label(filter_frame, text="Filter:", bg=C3).pack(side="left")
        self.filter_var = tk.StringVar(value="All")
        tk.OptionMenu(filter_frame, self.filter_var, "All", "Today", "This Month", "This Year").pack(side="left")

        # Search entry (hidden initially)
        self.history_search_var = tk.StringVar()
        self.history_search_entry = tk.Entry(filter_frame, textvariable=self.history_search_var)
        # populate list
        self.history_listbox = tk.Listbox(frame, height=18)
        self.history_listbox.pack(fill="both", expand=True, padx=6, pady=6)

        btns = tk.Frame(frame, bg=C3)
        btns.pack(fill="x", padx=6, pady=6)
        tk.Button(btns, text="Open Selected", command=self.open_selected_history, bg=ACCENT_BTN).pack(side="left", padx=4)
        tk.Button(btns, text="Delete Selected", command=self.delete_selected_history, bg=ACCENT_BTN).pack(side="left", padx=4)
        tk.Button(btns, text="Download Selected", command=self.download_selected_history, bg=ACCENT_BTN).pack(side="left", padx=4)
        tk.Button(btns, text="Refresh", command=self.refresh_history_list, bg=ACCENT_BTN).pack(side="right", padx=4)

    def toggle_search(self):
        if self.show_search.get():
            self.history_search_entry.pack(side="right", padx=6)
        else:
            self.history_search_entry.pack_forget()

    def refresh_history_list(self):
        self.history_listbox.delete(0, tk.END)
        f = self.filter_var.get()
        q = self.history_search_var.get().lower()
        for inv in self.invoices:
            if f == "Today":
                if inv.date.date() != datetime.now().date():
                    continue
            elif f == "This Month":
                now = datetime.now()
                if not (inv.date.year == now.year and inv.date.month == now.month):
                    continue
            elif f == "This Year":
                now = datetime.now()
                if inv.date.year != now.year:
                    continue
            label = f"{inv.id}: {inv.customer.name} - ${inv.total:.2f} - {inv.date.strftime('%Y-%m-%d')}"
            if q and q not in label.lower():
                continue
            self.history_listbox.insert(tk.END, label)

    def open_selected_history(self):
        sel = self.history_listbox.curselection()
        if not sel:
            messagebox.showerror("Error", "Select an invoice to open.")
            return
        idx = sel[0]
        label = self.history_listbox.get(idx)
        inv_id = int(label.split(":")[0])
        inv = next((i for i in self.invoices if i.id == inv_id), None)
        if not inv:
            messagebox.showerror("Error", "Invoice not found.")
            return
        # show detail popup
        popup = Toplevel(self.root)
        popup.title(f"Invoice {inv.id}")
        tk.Label(popup, text=f"Invoice ID: {inv.id}", font=("Arial", 12, "bold")).pack(pady=4)
        tk.Label(popup, text=f"Customer: {inv.customer.name}").pack(anchor="w")
        tk.Label(popup, text=f"Date: {inv.date.strftime('%Y-%m-%d %H:%M:%S')}").pack(anchor="w")
        tk.Label(popup, text=f"Subtotal: ${inv.subtotal:.2f}").pack(anchor="w")
        tk.Label(popup, text=f"GST ({inv.gst_rate}%): ${inv.gst_amount:.2f}").pack(anchor="w")
        tk.Label(popup, text=f"Total: ${inv.total:.2f}").pack(anchor="w")
        tk.Button(popup, text="Close", command=popup.destroy, bg=ACCENT_BTN).pack(pady=6)

    def delete_selected_history(self):
        sel = self.history_listbox.curselection()
        if not sel:
            messagebox.showerror("Error", "Select an invoice to delete.")
            return
        idx = sel[0]
        label = self.history_listbox.get(idx)
        inv_id = int(label.split(":")[0])
        self.invoices = [i for i in self.invoices if i.id != inv_id]
        self.refresh_history_list()
        self.add_notification("Selected invoice deleted.")

    def download_selected_history(self):
        sel = self.history_listbox.curselection()
        if not sel:
            messagebox.showerror("Error", "Select an invoice to download.")
            return
        idx = sel[0]
        label = self.history_listbox.get(idx)
        inv_id = int(label.split(":")[0])
        inv = next((i for i in self.invoices if i.id == inv_id), None)
        if not inv:
            messagebox.showerror("Error", "Invoice not found.")
            return
        if not self.perm_manager.check_permission("storage") or not self.perm_manager.check_permission("media"):
            messagebox.showerror("Permission Denied", "Storage and Media permissions required for PDFs.")
            return
        path = filedialog.asksaveasfilename(defaultextension=".pdf", filetypes=[("PDF files","*.pdf")])
        if path:
            threading.Thread(target=self.create_pdf, args=(inv, path)).start()

    # -----------------------------
    # Tab: Settings
    # -----------------------------
    def build_settings_tab(self):
        frame = self.tab_settings
        tk.Label(frame, text="Settings", bg=C3, fg=C4, font=("Arial", 14, "bold")).pack(anchor="w", padx=8, pady=6)
        # sub-notebook for settings sections
        settings_nb = ttk.Notebook(frame)
        settings_nb.pack(fill="both", expand=True, padx=8, pady=8)

        # General
        general = tk.Frame(settings_nb, bg=C3)
        settings_nb.add(general, text="General")
        tk.Label(general, text="Choose Language", bg=C3).pack(anchor="w", padx=8, pady=4)
        self.lang_var = tk.StringVar(value="English")
        tk.OptionMenu(general, self.lang_var, "English", "मराठी", "हिन्दी", "ગુજરાતી").pack(anchor="w", padx=8)
        tk.Label(general, text="Theme", bg=C3).pack(anchor="w", padx=8, pady=(10,0))
        self.theme_var = tk.StringVar(value="Light")
        tk.Radiobutton(general, text="Light Theme", variable=self.theme_var, value="Light", bg=C3).pack(anchor="w", padx=8)
        tk.Radiobutton(general, text="Dark Theme", variable=self.theme_var, value="Dark", bg=C3).pack(anchor="w", padx=8)
        tk.Label(general, text="Edit Name", bg=C3).pack(anchor="w", padx=8, pady=(10,0))
        self.name_var = tk.StringVar(value="Bhavik Mate")
        tk.Entry(general, textvariable=self.name_var, width=40).pack(anchor="w", padx=8)
        tk.Button(general, text="Save", command=self.save_settings, bg=ACCENT_BTN).pack(anchor="w", padx=8, pady=8)

        # Invoice Preference
        inv_pref = tk.Frame(settings_nb, bg=C3)
        settings_nb.add(inv_pref, text="Invoice Preference")
        tk.Label(inv_pref, text="Payment Terms", bg=C3).pack(anchor="w", padx=8, pady=(8,0))
        self.terms_list = tk.Listbox(inv_pref, height=6)
        self.terms_list.pack(fill="x", padx=8)
        for t in [
            "Payment due within 30 days of invoice date.",
            "Payment due immediately upon receipt.",
            "50% advance payment required.",
            "Payment due within 15 days; interest @2%/month."
        ]:
            self.terms_list.insert(tk.END, t)
        tk.Label(inv_pref, text="Template", bg=C3).pack(anchor="w", padx=8, pady=(8,0))
        self.template_var = tk.StringVar(value="Template 1")
        tk.OptionMenu(inv_pref, self.template_var, "Template 1", "Template 2", "Template 3").pack(anchor="w", padx=8)
        # Auto-save toggle
        self.autosave_var = tk.BooleanVar(value=False)
        tk.Checkbutton(inv_pref, text="Auto Save Invoices", variable=self.autosave_var, bg=C3).pack(anchor="w", padx=8, pady=6)

        # Tax Settings
        tax_tab = tk.Frame(settings_nb, bg=C3)
        settings_nb.add(tax_tab, text="Tax Settings")
        tk.Label(tax_tab, text="Manage GST Rates", bg=C3).pack(anchor="w", padx=8, pady=8)
        self.gst_listbox = tk.Listbox(tax_tab, height=6)
        self.gst_listbox.pack(fill="x", padx=8)
        for rate in [0, 5, 12, 18, 28]:
            self.gst_listbox.insert(tk.END, f"{rate}%")
        gst_ctrl = tk.Frame(tax_tab, bg=C3)
        gst_ctrl.pack(fill="x", padx=8, pady=6)
        tk.Button(gst_ctrl, text="Add Rate", command=self.add_gst_rate, bg=ACCENT_BTN).pack(side="left", padx=4)
        tk.Button(gst_ctrl, text="Remove Selected", command=self.remove_gst_rate, bg=ACCENT_BTN).pack(side="left", padx=4)

        # Security
        security = tk.Frame(settings_nb, bg=C3)
        settings_nb.add(security, text="Security")
        self.biometric_var = tk.BooleanVar(value=False)
        self.notify_var = tk.BooleanVar(value=self.perm_manager.permissions.get("notifications", False))
        tk.Checkbutton(security, text="Biometric Lock", variable=self.biometric_var, bg=C3).pack(anchor="w", padx=8, pady=6)
        tk.Checkbutton(security, text="Notifications", variable=self.notify_var, bg=C3, command=self.toggle_notifications).pack(anchor="w", padx=8, pady=6)

    def save_settings(self):
        name = self.name_var.get()
        messagebox.showinfo("Saved", f"Settings saved. Name updated to: {name}")
        # apply theme lightly
        if self.theme_var.get() == "Dark":
            self.root.configure(bg="#222")
        else:
            self.root.configure(bg=C3)

    def add_gst_rate(self):
        rate = simpledialog.askinteger("Add GST", "Enter GST % (integer):", minvalue=0, maxvalue=100)
        if rate is not None:
            self.gst_listbox.insert(tk.END, f"{rate}%")

    def remove_gst_rate(self):
        sel = self.gst_listbox.curselection()
        if not sel:
            return
        self.gst_listbox.delete(sel[0])

    def toggle_notifications(self):
        val = self.notify_var.get()
        self.perm_manager.permissions["notifications"] = val
        messagebox.showinfo("Notifications", f"Notifications {'enabled' if val else 'disabled'}.")

    # -----------------------------
    # Utilities: Notifications/updates
    # -----------------------------
    def add_notification(self, message):
        if self.perm_manager.check_permission("notifications"):
            self.notifications.append(message)
            # for simplicity, print to console and show small popup
            self.notification_popup(message)
        else:
            # silent fail or small messagebox
            print("Notification (blocked):", message)

    def notification_popup(self, message):
        popup = Toplevel(self.root)
        popup.title("Notification")
        popup.configure(bg=C3)
        tk.Label(popup, text=message, bg=C3).pack(padx=10, pady=10)
        tk.Button(popup, text="OK", command=popup.destroy, bg=ACCENT_BTN).pack(pady=(0,10))
        popup.after(4000, popup.destroy)  # auto-close after 4s

    def check_updates(self):
        if self.perm_manager.check_permission("notifications"):
            self.progress.start()
            threading.Thread(target=self.simulate_update_check).start()
        else:
            messagebox.showerror("Permission Denied", "Notifications permission required.")

    def simulate_update_check(self):
        time.sleep(1.5)
        self.progress.stop()
        self.add_notification("Update check complete: You're on the latest version.")

    def auto_check_updates(self):
        self.check_updates()
        self.root.after(300000, self.auto_check_updates)

    # -----------------------------
    # Customer/Product/Invoice actions
    # -----------------------------
    def add_customer(self):
        name = simpledialog.askstring("Add Customer", "Name:")
        email = simpledialog.askstring("Add Customer", "Email:")
        phone = simpledialog.askstring("Add Customer", "Phone:")
        if name and email and phone:
            customer = Customer(name, email, phone)
            self.customers.append(customer)
            # update UI lists
            if hasattr(self, "customer_listbox"):
                self.customer_listbox.insert(tk.END, f"{customer.name} ({customer.email}, {customer.phone})")
            self.add_notification("Customer added successfully!")
        else:
            messagebox.showerror("Error", "All fields required!")

    def search_customer(self):
        q = self.cust_search_var.get().lower()
        found = None
        for c in self.customers:
            if q in c.name.lower() or q in c.email.lower():
                found = c
                break
        if found:
            messagebox.showinfo("Found", f"Found: {found.name} ({found.email})")
        else:
            messagebox.showinfo("Not Found", "No matching customer.")

    def add_product(self):
        # re-used as Add Product dialog
        name = simpledialog.askstring("Add Product", "Name:")
        price = simpledialog.askfloat("Add Product", "Price:")
        if name and price is not None:
            product = Product(name, price)
            self.products.append(product)
            # update small product list
            if hasattr(self, "product_listbox_small"):
                self.product_listbox_small.insert(tk.END, f"{product.name} - {product.price:.2f}")
            self.add_notification("Product added successfully!")
        else:
            messagebox.showerror("Error", "Name and Price required!")

    def create_invoice(self):
        # kept for backward compatibility; prefer using Create Invoice tab UI
        if not self.customers or not self.products:
            messagebox.showerror("Error", "Add customers and products first!")
            return
        customer_idx = simpledialog.askinteger("Create Invoice", "Customer Index (0-based):")
        if customer_idx is None or customer_idx >= len(self.customers):
            return
        customer = self.customers[customer_idx]
        gst_rate = simpledialog.askfloat("GST Rate", "Select GST % (0, 5, 12, 18, 28):")
        if gst_rate not in [0, 5, 12, 18, 28]:
            messagebox.showerror("Error", "Invalid GST rate!")
            return
        template = simpledialog.askinteger("Template", "Select Template (1-5):")
        if template not in range(1, 6):
            return
        items = []
        while True:
            product_idx = simpledialog.askinteger("Add Item", "Product Index (0-based, -1 to finish):")
            if product_idx == -1:
                break
            if product_idx < len(self.products):
                qty = simpledialog.askinteger("Add Item", "Quantity:")
                if qty:
                    items.append((self.products[product_idx], qty))
        if items:
            invoice = Invoice(customer, items, gst_rate, template)
            self.invoices.append(invoice)
            self.add_notification(f"Invoice created! Total: ${invoice.total:.2f}")

    def process_payment(self):
        if not self.invoices:
            messagebox.showerror("Error", "No invoices!")
            return
        invoice_idx = simpledialog.askinteger("Process Payment", "Invoice Index (0-based):")
        if invoice_idx is None or invoice_idx >= len(self.invoices):
            return
        amount = simpledialog.askfloat("Process Payment", "Amount:")
        if amount:
            result = self.process_payment_logic(self.invoices[invoice_idx].id, amount)
            self.add_notification(result)

    def process_payment_logic(self, invoice_id, amount):
        invoice = next((i for i in self.invoices if i.id == invoice_id), None)
        if not invoice:
            return "Invoice not found"
        if amount >= invoice.total:
            invoice.paid = True
            self.revenue += invoice.total
            self.rev_label.config(text=f"Revenue: ${self.revenue:.2f}")
            return f"Payment successful. Change: ${amount - invoice.total:.2f}"
        return "Insufficient payment"

    # -----------------------------
    # Camera / Microphone features
    # -----------------------------
    def scan_receipt(self):
        if not self.perm_manager.check_permission("camera"):
            messagebox.showerror("Permission Denied", "Camera permission required for scanning.")
            return
        if not CAMERA_AVAILABLE:
            messagebox.showerror("Error", "Camera not available. Install OpenCV.")
            return
        cap = cv2.VideoCapture(0)
        ret, frame = cap.read()
        if ret:
            path = "scanned_receipt.png"
            cv2.imwrite(path, frame)
            cap.release()
            self.add_notification(f"Receipt scanned and saved as {path}!")
        else:
            messagebox.showerror("Error", "Failed to capture image.")

    def voice_note(self):
        if not self.perm_manager.check_permission("microphone"):
            messagebox.showerror("Permission Denied", "Microphone permission required for voice notes.")
            return
        if not MIC_AVAILABLE:
            messagebox.showerror("Error", "Microphone not available. Install speech_recognition.")
            return
        recognizer = sr.Recognizer()
        with sr.Microphone() as source:
            messagebox.showinfo("Voice Note", "Speak now...")
            audio = recognizer.listen(source)
            try:
                note = recognizer.recognize_google(audio)
                self.add_notification(f"Voice Note: {note}")
            except sr.UnknownValueError:
                messagebox.showerror("Error", "Could not understand audio.")
            except sr.RequestError:
                messagebox.showerror("Error", "Speech service unavailable.")

    # -----------------------------
    # PDF generation (simple)
    # -----------------------------
    def generate_pdf(self):
        # Used by Create Invoice tab: gather details from UI and create invoice and PDF
        if not self.perm_manager.check_permission("storage") or not self.perm_manager.check_permission("media"):
            messagebox.showerror("Permission Denied", "Storage and Media permissions required for PDFs.")
            return
        # For simplicity, pick first customer if exists
        if not self.customers:
            messagebox.showerror("Error", "Add a customer first.")
            return
        customer = self.customers[0]
        # gather items from treeview
        items = []
        for child in self.items_tree.get_children():
            name, price_str, qty = self.items_tree.item(child, "values")
            price = float(price_str)
            prod = Product(name, price)
            items.append((prod, int(qty)))
        if not items:
            messagebox.showerror("Error", "Add items to invoice.")
            return
        gst_rate = 18  # default; you may choose from gst_listbox selection
        template = self.template_var.get() if hasattr(self, "template_var") else "Template 1"
        inv = Invoice(customer, items, gst_rate, template, payment_mode=self.payment_mode_var.get(), terms=self.terms_var.get())
        self.invoices.append(inv)
        self.add_notification(f"Invoice created: {inv.id} - Total ${inv.total:.2f}")
        # ask where to save pdf
        path = filedialog.asksaveasfilename(defaultextension=".pdf", filetypes=[("PDF files","*.pdf")])
        if path:
            threading.Thread(target=self.create_pdf, args=(inv, path)).start()

    def create_pdf(self, invoice, file_path):
        # create a simple PDF representation
        try:
            doc = SimpleDocTemplate(file_path, pagesize=letter)
            styles = getSampleStyleSheet()
            story = []
            story.append(Paragraph("NEW BHAVIK DIGITAL", styles['Title']))
            story.append(Spacer(1, 12))
            story.append(Paragraph(f"Invoice ID: {invoice.id}", styles['Normal']))
            story.append(Paragraph(f"Date: {invoice.date.strftime('%Y-%m-%d %H:%M:%S')}", styles['Normal']))
            story.append(Paragraph(f"Customer: {invoice.customer.name} ({invoice.customer.email})", styles['Normal']))
            story.append(Spacer(1, 8))
            data = [["Item", "Price", "Qty", "Total"]]
            for prod, qty in invoice.items:
                data.append([prod.name, f"{prod.price:.2f}", str(qty), f"{prod.price * qty:.2f}"])
            data.append(["", "", "Subtotal", f"{invoice.subtotal:.2f}"])
            data.append(["", "", f"GST ({invoice.gst_rate}%)", f"{invoice.gst_amount:.2f}"])
            data.append(["", "", "Total", f"{invoice.total:.2f}"])
            table = Table(data, hAlign='LEFT')
            table.setStyle(TableStyle([
                ('GRID', (0,0), (-1,-1), 0.5, rl_colors.grey),
                ('BACKGROUND', (0,0), (-1,0), rl_colors.lightblue),
                ('FONTNAME', (0,0), (-1,0), 'Helvetica-Bold'),
            ]))
            story.append(table)
            story.append(Spacer(1, 12))
            story.append(Paragraph(f"Payment Mode: {invoice.payment_mode}", styles['Normal']))
            story.append(Paragraph(f"Terms: {invoice.terms}", styles['Normal']))
            doc.build(story)
            self.progress.stop()
            self.add_notification(f"PDF generated and saved to {file_path}!")
        except Exception as e:
            messagebox.showerror("Error", f"Failed to create PDF: {e}")

    def preview_invoice(self):
        # Quick preview using tree items + first customer
        if not self.customers:
            messagebox.showerror("Error", "No customers.")
            return
        if not self.items_tree.get_children():
            messagebox.showerror("Error", "No items.")
            return
        # build text
        customer = self.customers[0]
        subtotal = 0.0
        out = [f"Preview - Customer: {customer.name}", ""]
        for child in self.items_tree.get_children():
            name, price_str, qty = self.items_tree.item(child, "values")
            line_total = float(price_str) * int(qty)
            subtotal += line_total
            out.append(f"{name} x {qty} = {line_total:.2f}")
        gst_rate = 18
        gst_amt = subtotal * (gst_rate / 100)
        total = subtotal + gst_amt
        out.append("")
        out.append(f"Subtotal: {subtotal:.2f}")
        out.append(f"GST: {gst_amt:.2f}")
        out.append(f"Total: {total:.2f}")
        messagebox.showinfo("Invoice Preview", "\n".join(out))

# ----------------------------
# Run App
# ----------------------------
if __name__ == "__main__":
    root = tk.Tk()
    app = BillingApp(root)
    root.mainloop()
