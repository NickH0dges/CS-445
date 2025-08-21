# Created by Nick Hodges and Alex Boehne for CS 445 with Dr. Suja
# at Southeast Missouri State University
#
# Please see the README file for usage information.
# This software is released under the MIT License.


import json
import os
import csv
from datetime import datetime
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, filedialog

APP_DIR = os.path.dirname(os.path.abspath(__file__))
USERS_FILE = os.path.join(APP_DIR, 'pos_users.json')
ITEMS_FILE = os.path.join(APP_DIR, 'pos_items.json')
TX_LOG    = os.path.join(APP_DIR, 'pos_transactions.csv')

TAX_RATE = 0.0825  # fixed tax inside the program (8.25%). Change as needed.

# --------------------- data helpers ---------------------
def load_json(path, default):
    if not os.path.exists(path):
        with open(path, 'w', encoding='utf-8') as f:
            json.dump(default, f, indent=2)
        return default.copy()
    try:
        with open(path, 'r', encoding='utf-8') as f:
            return json.load(f)
    except Exception:
        return default.copy()

def save_json(path, data):
    with open(path, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2)

# Bootstrap minimal data on first run
DEFAULT_USERS = {
    "0001": {"name": "Admin", "pin": "1234", "is_admin": True},
}
DEFAULT_ITEMS = {
    # sku: {name, price}
    "100001": {"name": "Bottle Water", "price": 1.00},
    "100002": {"name": "Chips", "price": 1.50},
}

USERS = load_json(USERS_FILE, DEFAULT_USERS)
ITEMS = load_json(ITEMS_FILE, DEFAULT_ITEMS)

if not os.path.exists(TX_LOG):
    with open(TX_LOG, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow([
            'timestamp','cashier_id','cashier_name','payment_type','card_txn','subtotal','tax','total','lines_json'
        ])

# --------------------- domain logic ---------------------
class Cart:
    def __init__(self):
        self.lines = []  # each: {sku, name, price, qty}

    def add(self, sku, name, price, qty=1):
        # if already in cart, increase qty
        for line in self.lines:
            if line['sku'] == sku:
                line['qty'] += qty
                return
        self.lines.append({'sku': sku, 'name': name, 'price': float(price), 'qty': int(qty)})

    def remove_index(self, idx):
        if 0 <= idx < len(self.lines):
            self.lines.pop(idx)

    def set_qty(self, idx, qty):
        if 0 <= idx < len(self.lines):
            self.lines[idx]['qty'] = max(1, int(qty))

    def clear(self):
        self.lines.clear()

    def subtotal(self):
        return sum(l['price'] * l['qty'] for l in self.lines)

    def tax(self):
        return round(self.subtotal() * TAX_RATE, 2)

    def total(self):
        return round(self.subtotal() + self.tax(), 2)

# --------------------- UI: App Shell ---------------------
class POSApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title('Simple POS')
        self.geometry('1100x700')
        self.minsize(960, 600)
        self.resizable(True, True)

        # ---- Theming (inviting slate + green accent) ----
        self.BG = '#0f172a'       # slate-900
        self.CARD_BG = '#111827'  # slate-800
        self.ACCENT = '#22c55e'   # green-500
        self.FG = '#e5e7eb'       # gray-200

        style = ttk.Style(self)
        try:
            style.theme_use('clam')
        except Exception:
            pass
        self.configure(bg=self.BG)
        style.configure('App.TFrame', background=self.BG)
        style.configure('Card.TFrame', background=self.CARD_BG)
        style.configure('Big.TLabel', background=self.BG, foreground=self.FG, font=('Segoe UI', 36, 'bold'))
        style.configure('Muted.TLabel', background=self.BG, foreground='#94a3b8', font=('Segoe UI', 11))
        style.configure('CardLabel.TLabel', background=self.CARD_BG, foreground=self.FG, font=('Segoe UI', 12))
        style.configure('Accent.TButton', padding=(16,12), font=('Segoe UI', 12, 'bold'))
        style.map('Accent.TButton',
                  background=[('!disabled', self.ACCENT), ('active', '#16a34a')],
                  foreground=[('!disabled', 'white')])
        style.configure('Key.TButton', padding=(10,14), font=('Segoe UI', 16))
        # Apply theme to default ttk widgets so unlabeled frames pick up the colors
        style.configure('TFrame', background=self.BG)
        style.configure('TLabel', background=self.BG, foreground=self.FG)
        style.configure('TLabelframe', background=self.CARD_BG)
        style.configure('TLabelframe.Label', background=self.CARD_BG, foreground=self.FG)

        self.active_user_id = None
        self.active_user = None
        self.cart = Cart()
        self.show_login()

    # ---------- Screens ----------
    def show_login(self):
        self.clear_root()
        LoginFrame(self, on_success=self.on_login_ok).pack(fill='both', expand=True)

    def show_main(self):
        self.clear_root()
        MainPOSFrame(self).pack(fill='both', expand=True)

    def on_login_ok(self, user_id):
        self.active_user_id = user_id
        self.active_user = USERS.get(user_id)
        self.show_main()

    def clear_root(self):
        for w in self.winfo_children():
            w.destroy()

# --------------------- UI: Login ---------------------
class LoginFrame(ttk.Frame):
    """Touch-friendly login with on-screen numeric keypad.
    Default admin: ID 0001 / PIN 1234
    """
    def __init__(self, master, on_success):
        super().__init__(master, style='App.TFrame')
        self.on_success = on_success
        self.active_entry = None  # which entry receives keypad input

        # Center card
        container = ttk.Frame(self, style='App.TFrame')
        container.pack(fill='both', expand=True)
        card = ttk.Frame(container, style='Card.TFrame', padding=24)
        card.place(relx=0.5, rely=0.5, anchor='center')
        card.columnconfigure(0, weight=0, uniform='cols')
        card.columnconfigure(1, weight=1, uniform='cols')

        # ----- Keypad (left) -----
        kp = ttk.Frame(card, style='Card.TFrame')
        kp.grid(row=1, column=0, rowspan=3, sticky='n', padx=(0,20))
        kp.columnconfigure((0,1,2), weight=1, uniform='kp')
        kp.rowconfigure((0,1,2,3), weight=1, uniform='kp')
        buttons = [
            '1','2','3',
            '4','5','6',
            '7','8','9',
            '←','0','×'
        ]
        def put(ch):
            widget = self.active_entry or self.id_entry
            if ch == '←':
                cur = widget.get()
                widget.delete(0, tk.END)
                widget.insert(0, cur[:-1])
            elif ch == '×':
                widget.delete(0, tk.END)
            else:
                widget.insert(tk.END, ch)
        r=c=0
        for label in buttons:
            b = ttk.Button(kp, text=label, style='Key.TButton', command=lambda l=label: put(l), width=4)
            b.grid(row=r, column=c, padx=6, pady=6, sticky='nsew', ipady=8)
            c += 1
            if c==3:
                c=0; r+=1

        # ----- Fields + actions (right) -----
        title = ttk.Label(card, text='Simple POS', style='Big.TLabel')
        title.grid(row=0, column=0, columnspan=2, sticky='w', pady=(0,10))

        fields = ttk.Frame(card, style='Card.TFrame')
        fields.grid(row=1, column=1, sticky='nsew')
        for i in range(2):
            fields.columnconfigure(i, weight=1)

        ttk.Label(fields, text='User ID (4 digits)', style='CardLabel.TLabel').grid(row=0, column=0, sticky='w')
        self.id_var = tk.StringVar()
        self.id_entry = ttk.Entry(fields, textvariable=self.id_var, font=('Segoe UI', 18), width=14)
        self.id_entry.grid(row=1, column=0, sticky='ew', pady=(2,12))
        self.id_entry.bind('<FocusIn>', lambda e: self.set_active(self.id_entry))

        ttk.Label(fields, text='PIN (4 digits)', style='CardLabel.TLabel').grid(row=2, column=0, sticky='w')
        self.pin_var = tk.StringVar()
        self.pin_entry = ttk.Entry(fields, textvariable=self.pin_var, show='•', font=('Segoe UI', 18), width=14)
        self.pin_entry.grid(row=3, column=0, sticky='ew', pady=(2,16))
        self.pin_entry.bind('<FocusIn>', lambda e: self.set_active(self.pin_entry))

        action_row = ttk.Frame(card, style='Card.TFrame')
        action_row.grid(row=2, column=1, sticky='w')
        ttk.Button(action_row, text='LOGIN', style='Accent.TButton', command=self.try_login).pack(anchor='w')

        ttk.Label(card, text='Tap a field, then use the keypad. Default admin: 0001 / 1234', style='CardLabel.TLabel').grid(row=3, column=1, sticky='w', pady=(12,0))

        self.id_entry.focus_set()
        self.set_active(self.id_entry)

    def set_active(self, widget):
        self.active_entry = widget

    def try_login(self):
        uid = self.id_var.get().strip()
        pin = self.pin_var.get().strip()
        user = USERS.get(uid)
        if not user or user.get('pin') != pin:
            messagebox.showerror('Login failed', 'Invalid ID or PIN')
            return
        self.on_success(uid)

# --------------------- UI: Main POS ---------------------
class MainPOSFrame(ttk.Frame):
    def __init__(self, master: POSApp):
        super().__init__(master)
        self.app = master
        self.checkout_win = None

        # top bar
        top = ttk.Frame(self)
        top.pack(fill='x', pady=6, padx=8)
        ttk.Label(top, text=f"Logged in: {self.app.active_user.get('name')} ({self.app.active_user_id})").pack(side='left')
        ttk.Button(top, text='Sign out', command=self.sign_out).pack(side='right')

        # main area split
        body = ttk.Frame(self)
        body.pack(fill='both', expand=True, padx=8, pady=8)
        body.columnconfigure(1, weight=1)
        body.rowconfigure(0, weight=1)

        # Left: receipt list
        left = ttk.LabelFrame(body, text='Receipt')
        left.grid(row=0, column=0, sticky='nsw', padx=(0,8))
        left.rowconfigure(1, weight=1)

        self.listbox = tk.Listbox(left, width=45, height=20, font=('Consolas', 12),
                                   bg=self.app.CARD_BG, fg=self.app.FG,
                                   selectbackground=self.app.ACCENT, highlightthickness=0, relief='flat')
        self.listbox.grid(row=1, column=0, padx=8, pady=8, sticky='ns')

        btns = ttk.Frame(left)
        btns.grid(row=2, column=0, pady=(0,8))
        ttk.Button(btns, text='Change Qty', command=self.change_qty).pack(side='left', padx=4)
        ttk.Button(btns, text='Remove', command=self.remove_line).pack(side='left', padx=4)
        ttk.Button(btns, text='Clear', command=self.clear_cart).pack(side='left', padx=4)

        # Right: actions
        right = ttk.LabelFrame(body, text='Scan / Lookup')
        right.grid(row=0, column=1, sticky='nsew')
        right.columnconfigure(1, weight=1)

        ttk.Label(right, text='Scan or enter SKU:').grid(row=0, column=0, padx=8, pady=6, sticky='e')
        self.sku_var = tk.StringVar()
        sku_entry = ttk.Entry(right, textvariable=self.sku_var, font=('Segoe UI', 14))
        sku_entry.grid(row=0, column=1, padx=8, pady=6, sticky='ew')
        sku_entry.bind('<Return>', lambda e: self.add_by_sku())
        ttk.Button(right, text='Add', command=self.add_by_sku).grid(row=0, column=2, padx=8, pady=6)
        ttk.Button(right, text='Lookup…', command=self.open_lookup).grid(row=0, column=3, padx=(0,8), pady=6)

        ttk.Separator(right).grid(row=1, column=0, columnspan=4, sticky='ew', pady=6)

        self.total_var = tk.StringVar(value='Total: $0.00')
        total_lbl = ttk.Label(right, textvariable=self.total_var, font=('Segoe UI', 24, 'bold'))
        total_lbl.grid(row=2, column=0, columnspan=4, padx=8, pady=(6,2), sticky='w')

        foot = ttk.Frame(right)
        foot.grid(row=3, column=0, columnspan=4, sticky='ew', padx=8, pady=8)
        foot.columnconfigure(0, weight=1)
        self.checkout_btn = ttk.Button(foot, text='Checkout', command=self.checkout)
        self.checkout_btn.grid(row=0, column=1, sticky='e')

        # Admin button only for admins
        if self.app.active_user.get('is_admin'):
            ttk.Button(foot, text='Admin', command=self.open_admin).grid(row=0, column=2, padx=(8,0))

        self.refresh_list()

    def sign_out(self):
        self.app.active_user = None
        self.app.active_user_id = None
        self.app.cart.clear()
        self.app.show_login()

    def refresh_list(self):
        self.listbox.delete(0, tk.END)
        for line in self.app.cart.lines:
            self.listbox.insert(tk.END, f"{line['qty']} x {line['name']:<20} @ ${line['price']:.2f}")
        self.total_var.set(f"Total: ${self.app.cart.total():.2f}")

    def add_by_sku(self):
        sku = self.sku_var.get().strip()
        if not sku:
            return
        item = ITEMS.get(sku)
        if not item:
            messagebox.showerror('Not found', f'SKU {sku} not in system')
        else:
            self.app.cart.add(sku, item['name'], item['price'], qty=1)
            self.refresh_list()
        self.sku_var.set('')

    def open_lookup(self):
        ItemLookupWindow(self.app, on_pick=self.add_item_from_lookup)

    def add_item_from_lookup(self, sku, qty):
        item = ITEMS.get(sku)
        if not item:
            return
        self.app.cart.add(sku, item['name'], item['price'], qty=qty)
        self.refresh_list()

    def change_qty(self):
        idx = self.listbox.curselection()
        if not idx:
            return
        idx = idx[0]
        line = self.app.cart.lines[idx]
        try:
            qty = simpledialog.askinteger('Quantity', f"Set quantity for {line['name']}", minvalue=1, initialvalue=line['qty'])
            if qty:
                self.app.cart.set_qty(idx, qty)
                self.refresh_list()
        except Exception:
            pass

    def remove_line(self):
        idx = self.listbox.curselection()
        if not idx:
            return
        self.app.cart.remove_index(idx[0])
        self.refresh_list()

    def clear_cart(self):
        if self.app.cart.lines and messagebox.askyesno('Clear cart', 'Remove all items from this transaction?'):
            self.app.cart.clear()
            self.refresh_list()

    def checkout(self):
        if not self.app.cart.lines:
            messagebox.showinfo('Empty', 'Add items before checkout')
            return
        # Single-instance guard
        if self.checkout_win and self.checkout_win.winfo_exists():
            self.checkout_win.lift()
            self.checkout_win.focus_force()
            return
        self.checkout_win = CheckoutWindow(self.app, on_done=self.on_sale_done)
        try:
            self.checkout_win.protocol('WM_DELETE_WINDOW', self.on_checkout_closed)
        except Exception:
            pass

    def on_sale_done(self):
        self.app.cart.clear()
        self.refresh_list()
        self.on_checkout_closed()

    def on_checkout_closed(self):
        if self.checkout_win and self.checkout_win.winfo_exists():
            try:
                self.checkout_win.grab_release()
            except Exception:
                pass
            self.checkout_win.destroy()
        self.checkout_win = None

    def open_admin(self):
        AdminWindow(self.app)

# --------------------- Item Lookup (no UPC) ---------------------
class ItemLookupWindow(tk.Toplevel):
    """Search by name and add to cart for items without a UPC scan."""
    def __init__(self, app: POSApp, on_pick):
        super().__init__(app)
        self.app = app
        self.on_pick = on_pick
        self.title('Item Lookup')
        self.geometry('520x500')
        self.resizable(True, True)
        self.configure(bg=self.app.BG)
        self.grab_set()

        top = ttk.Frame(self)
        top.pack(fill='x', padx=10, pady=10)
        ttk.Label(top, text='Search:').pack(side='left')
        self.q = tk.StringVar()
        entry = ttk.Entry(top, textvariable=self.q)
        entry.pack(side='left', fill='x', expand=True, padx=8)
        entry.bind('<KeyRelease>', lambda e: self.refresh())

        self.tree = ttk.Treeview(self, columns=('sku','name','price'), show='headings')
        self.tree.heading('sku', text='SKU')
        self.tree.heading('name', text='Name')
        self.tree.heading('price', text='Price ($)')
        self.tree.column('sku', width=100)
        self.tree.column('name', width=260)
        self.tree.column('price', width=90, anchor='e')
        self.tree.pack(fill='both', expand=True, padx=10, pady=(0,10))
        self.tree.bind('<Double-1>', lambda e: self.add_selected())

        bottom = ttk.Frame(self)
        bottom.pack(fill='x', padx=10, pady=(0,10))
        ttk.Button(bottom, text='Add Selected', command=self.add_selected).pack(side='right')

        self.refresh()
        entry.focus_set()

    def refresh(self):
        q = self.q.get().strip().lower()
        self.tree.delete(*self.tree.get_children())
        for sku, item in ITEMS.items():
            if q and q not in item['name'].lower():
                continue
            self.tree.insert('', 'end', iid=sku, values=(sku, item['name'], f"{item['price']:.2f}"))

    def add_selected(self):
        sel = self.tree.selection()
        if not sel:
            return
        sku = sel[0]
        qty = simpledialog.askinteger('Quantity', 'Quantity:', minvalue=1, initialvalue=1)
        if not qty:
            return
        self.on_pick(sku, qty)
        self.destroy()

# --------------------- UI: Checkout ---------------------
class CheckoutWindow(tk.Toplevel):
    def __init__(self, app: POSApp, on_done):
        super().__init__(app)
        self.app = app
        self.on_done = on_done
        self.title('Checkout')
        self.geometry('560x460')
        self.resizable(False, False)
        self.configure(bg=self.app.BG)
        self.transient(app)
        self.grab_set()
        self.focus_force()

        sub = self.app.cart.subtotal()
        tax = self.app.cart.tax()
        total = self.app.cart.total()

        header = ttk.Frame(self)
        header.pack(fill='x', padx=12, pady=12)
        ttk.Label(header, text=f"Subtotal: ${sub:.2f}").pack(anchor='w')
        ttk.Label(header, text=f"Tax (@ {TAX_RATE*100:.2f}%): ${tax:.2f}").pack(anchor='w')
        ttk.Label(header, text=f"Total:", font=('Segoe UI', 11, 'bold')).pack(anchor='w', pady=(6,0))
        ttk.Label(header, text=f"${total:.2f}", font=('Segoe UI', 28, 'bold')).pack(anchor='w')

        self.pay_var = tk.StringVar(value='cash')
        pay_frame = ttk.LabelFrame(self, text='Payment')
        pay_frame.pack(fill='x', padx=12, pady=8)
        ttk.Radiobutton(pay_frame, text='Cash', variable=self.pay_var, value='cash', command=self.toggle_payment_fields).pack(anchor='w', padx=8, pady=4)
        ttk.Radiobutton(pay_frame, text='Card (standalone terminal)', variable=self.pay_var, value='card', command=self.toggle_payment_fields).pack(anchor='w', padx=8, pady=4)

        # cash fields
        self.cash_frame = ttk.Frame(pay_frame)
        self.cash_frame.pack(fill='x', padx=8, pady=(4,8))
        ttk.Label(self.cash_frame, text='Cash received:').grid(row=0, column=0, sticky='e')
        self.cash_var = tk.StringVar()
        e = ttk.Entry(self.cash_frame, textvariable=self.cash_var, width=12)
        e.grid(row=0, column=1, padx=6)
        e.bind('<KeyRelease>', lambda e: self.update_change())
        ttk.Label(self.cash_frame, text='Change due:').grid(row=0, column=2, sticky='e')
        self.change_var = tk.StringVar(value='$0.00')
        ttk.Label(self.cash_frame, textvariable=self.change_var, font=('Segoe UI', 12, 'bold')).grid(row=0, column=3, padx=6)

        # card fields
        self.card_frame = ttk.Frame(pay_frame)
        ttk.Label(self.card_frame, text='Card transaction # (from terminal):').grid(row=0, column=0, sticky='e')
        self.card_var = tk.StringVar()
        ttk.Entry(self.card_frame, textvariable=self.card_var, width=24).grid(row=0, column=1, padx=6)

        ttk.Button(self, text='Confirm Sale', command=self.confirm).pack(pady=12)
        self.toggle_payment_fields()

    def toggle_payment_fields(self):
        if self.pay_var.get() == 'cash':
            self.card_frame.forget()
            self.cash_frame.pack(fill='x', padx=8, pady=(4,8))
        else:
            self.cash_frame.forget()
            self.card_frame.pack(fill='x', padx=8, pady=(4,8))

    def update_change(self):
        try:
            received = float(self.cash_var.get()) if self.cash_var.get() else 0.0
            change = max(0.0, round(received - self.app.cart.total(), 2))
            self.change_var.set(f"${change:.2f}")
        except ValueError:
            self.change_var.set('—')

    def confirm(self):
        payment_type = self.pay_var.get()
        card_txn = self.card_var.get().strip() if payment_type == 'card' else ''
        if payment_type == 'cash':
            try:
                received = float(self.cash_var.get())
            except Exception:
                messagebox.showerror('Invalid', 'Enter cash received')
                return
            if received + 1e-9 < self.app.cart.total():
                messagebox.showerror('Underpayment', 'Cash received is less than the total. Cannot complete sale.')
                return
        else:
            if not card_txn:
                if not messagebox.askyesno('No transaction #', 'No card transaction number entered. Continue?'):
                    return

        # write audit log
        try:
            with open(TX_LOG, 'a', newline='', encoding='utf-8') as f:
                writer = csv.writer(f)
                writer.writerow([
                    datetime.now().isoformat(timespec='seconds'),
                    self.app.active_user_id,
                    self.app.active_user.get('name'),
                    payment_type,
                    card_txn,
                    f"{self.app.cart.subtotal():.2f}",
                    f"{self.app.cart.tax():.2f}",
                    f"{self.app.cart.total():.2f}",
                    json.dumps(self.app.cart.lines)
                ])
        except Exception as e:
            messagebox.showerror('Error', f'Failed to write audit log: {e}')
            return

        messagebox.showinfo('Sale complete', 'Transaction recorded.')
        self.on_done()
        self.destroy()

# --------------------- UI: Admin ---------------------
class AdminWindow(tk.Toplevel):
    def __init__(self, app: POSApp):
        super().__init__(app)
        self.app = app
        self.title('Admin')
        self.geometry('720x520')
        self.resizable(True, True)
        self.grab_set()

        nb = ttk.Notebook(self)
        nb.pack(fill='both', expand=True)

        self.items_tab = ItemsAdmin(nb)
        self.users_tab = UsersAdmin(nb)
        self.audit_tab = AuditAdmin(nb)

        nb.add(self.items_tab, text='Items (SKU)')
        nb.add(self.users_tab, text='Users')
        nb.add(self.audit_tab, text='Audit / Export')

# ---- Items Admin ----
class ItemsAdmin(ttk.Frame):
    def __init__(self, master):
        super().__init__(master)
        self.tree = ttk.Treeview(self, columns=('name','price'), show='headings')
        self.tree.heading('name', text='Name')
        self.tree.heading('price', text='Price ($)')
        self.tree.column('name', width=260)
        self.tree.column('price', width=100, anchor='e')
        self.tree.pack(fill='both', expand=True, padx=8, pady=8)

        btns = ttk.Frame(self)
        btns.pack(fill='x', padx=8, pady=(0,8))
        ttk.Button(btns, text='Add', command=self.add_item).pack(side='left')
        ttk.Button(btns, text='Edit', command=self.edit_item).pack(side='left', padx=6)
        ttk.Button(btns, text='Delete', command=self.del_item).pack(side='left')

        self.refresh()

    def refresh(self):
        self.tree.delete(*self.tree.get_children())
        for sku, item in ITEMS.items():
            self.tree.insert('', 'end', iid=sku, values=(f"{item['name']}", f"{item['price']:.2f}"))

    def add_item(self):
        sku = simpledialog.askstring('New SKU', 'Enter new SKU (string/number):')
        if not sku:
            return
        if sku in ITEMS:
            messagebox.showerror('Exists', 'SKU already exists')
            return
        name = simpledialog.askstring('Name', 'Item name:')
        if not name:
            return
        try:
            price = float(simpledialog.askstring('Price', 'Price in dollars:'))
        except Exception:
            messagebox.showerror('Invalid', 'Price must be a number')
            return
        ITEMS[sku] = {'name': name, 'price': price}
        save_json(ITEMS_FILE, ITEMS)
        self.refresh()

    def edit_item(self):
        sel = self.tree.selection()
        if not sel:
            return
        sku = sel[0]
        item = ITEMS[sku]
        name = simpledialog.askstring('Name', 'Item name:', initialvalue=item['name'])
        if not name:
            return
        try:
            price = float(simpledialog.askstring('Price', 'Price in dollars:', initialvalue=item['price']))
        except Exception:
            messagebox.showerror('Invalid', 'Price must be a number')
            return
        ITEMS[sku] = {'name': name, 'price': price}
        save_json(ITEMS_FILE, ITEMS)
        self.refresh()

    def del_item(self):
        sel = self.tree.selection()
        if not sel:
            return
        sku = sel[0]
        if messagebox.askyesno('Delete', f'Delete SKU {sku}?'):
            ITEMS.pop(sku, None)
            save_json(ITEMS_FILE, ITEMS)
            self.refresh()

# ---- Users Admin ----
class UsersAdmin(ttk.Frame):
    def __init__(self, master):
        super().__init__(master)
        self.tree = ttk.Treeview(self, columns=('name','pin','is_admin'), show='headings')
        self.tree.heading('name', text='Name')
        self.tree.heading('pin', text='PIN')
        self.tree.heading('is_admin', text='Admin')
        self.tree.column('name', width=220)
        self.tree.column('pin', width=80)
        self.tree.column('is_admin', width=80, anchor='center')
        self.tree.pack(fill='both', expand=True, padx=8, pady=8)

        btns = ttk.Frame(self)
        btns.pack(fill='x', padx=8, pady=(0,8))
        ttk.Button(btns, text='Add', command=self.add_user).pack(side='left')
        ttk.Button(btns, text='Edit', command=self.edit_user).pack(side='left', padx=6)
        ttk.Button(btns, text='Delete', command=self.del_user).pack(side='left')

        self.refresh()

    def refresh(self):
        self.tree.delete(*self.tree.get_children())
        for uid, u in USERS.items():
            self.tree.insert('', 'end', iid=uid, values=(u['name'], u['pin'], 'Yes' if u.get('is_admin') else 'No'))

    def add_user(self):
        uid = simpledialog.askstring('User ID', '4-digit User ID:')
        if not uid:
            return
        if uid in USERS:
            messagebox.showerror('Exists', 'User ID already exists')
            return
        name = simpledialog.askstring('Name', 'Full name:')
        pin  = simpledialog.askstring('PIN',  '4-digit PIN:')
        is_admin = messagebox.askyesno('Admin', 'Grant admin access?')
        USERS[uid] = {'name': name or uid, 'pin': pin or '0000', 'is_admin': bool(is_admin)}
        save_json(USERS_FILE, USERS)
        self.refresh()

    def edit_user(self):
        sel = self.tree.selection()
        if not sel:
            return
        uid = sel[0]
        u = USERS[uid]
        name = simpledialog.askstring('Name', 'Full name:', initialvalue=u['name'])
        pin  = simpledialog.askstring('PIN',  '4-digit PIN:', initialvalue=u['pin'])
        is_admin = messagebox.askyesno('Admin', 'Is admin? (Yes=admin, No=standard)')
        USERS[uid] = {'name': name or uid, 'pin': pin or '0000', 'is_admin': bool(is_admin)}
        save_json(USERS_FILE, USERS)
        self.refresh()

    def del_user(self):
        sel = self.tree.selection()
        if not sel:
            return
        uid = sel[0]
        if messagebox.askyesno('Delete', f'Delete user {uid}?'):
            USERS.pop(uid, None)
            save_json(USERS_FILE, USERS)
            self.refresh()

# ---- Audit / Export ----
class AuditAdmin(ttk.Frame):
    def __init__(self, master):
        super().__init__(master)

        ttk.Button(self, text='Export Audit Text…', command=self.export_text).pack(anchor='w', padx=8, pady=(6,0))
        ttk.Button(self, text='Open CSV Log…', command=self.open_csv_copy).pack(anchor='w', padx=8, pady=(6,0))

    def export_text(self):
        if not os.path.exists(TX_LOG):
            messagebox.showinfo('No data', 'No transactions yet.')
            return

        path = filedialog.asksaveasfilename(
            title='Save audit report',
            defaultextension='.txt',
            filetypes=[('Text', '*.txt')]
        )
        if not path:
            return

        out_lines = []
        with open(TX_LOG, 'r', encoding='utf-8') as f:
            reader = csv.DictReader(f)
            for row in reader:
                lines = json.loads(row['lines_json'])
                out_lines.append('=' * 50)
                out_lines.append("Time: {}".format(row['timestamp']))
                out_lines.append("Cashier: {} ({})".format(row['cashier_name'], row['cashier_id']))
                out_lines.append("Payment: {}  CardTxn: {}".format(row['payment_type'].upper(), row['card_txn']))
                out_lines.append('Items:')
                for l in lines:
                    out_lines.append("  - {} x {} @ ${:.2f}".format(l['qty'], l['name'], l['price']))
                out_lines.append("Subtotal: ${}  Tax: ${}  Total: ${}".format(row['subtotal'], row['tax'], row['total']))
                out_lines.append('')

        with open(path, 'w', encoding='utf-8') as out:
            out.write("\n".join(out_lines))

        messagebox.showinfo('Saved', 'Audit text saved to:\n{}'.format(path))

    def open_csv_copy(self):
        if not os.path.exists(TX_LOG):
            messagebox.showinfo('No data', 'No transactions yet.')
            return

        path = filedialog.asksaveasfilename(
            title='Save CSV copy',
            defaultextension='.csv',
            filetypes=[('CSV', '*.csv')]
        )
        if not path:
            return

        with open(TX_LOG, 'r', encoding='utf-8') as src, open(path, 'w', encoding='utf-8') as dst:
            dst.write(src.read())

        messagebox.showinfo('Saved', 'CSV copy saved to:\n{}'.format(path))



def open_csv_copy(self):
        # Let user save a copy of the CSV next to where they want
        if not os.path.exists(TX_LOG):
            messagebox.showinfo('No data', 'No transactions yet.')
            return
        path = filedialog.asksaveasfilename(title='Save CSV log copy', defaultextension='.csv', filetypes=[('CSV','*.csv')])
        if not path:
            return
        with open(TX_LOG, 'r', encoding='utf-8') as src, open(path, 'w', encoding='utf-8') as dst:
            dst.write(src.read())
        messagebox.showinfo('Saved', 'CSV copy saved to:{}'.format(path))

# --------------------- run ---------------------
if __name__ == '__main__':
    app = POSApp()
    app.mainloop()
