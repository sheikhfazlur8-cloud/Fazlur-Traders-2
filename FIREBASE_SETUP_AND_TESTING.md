# StockDesk — Firebase Setup, Security Rules & Testing Guide

## 1. Required Firebase Configuration Placeholders

In `index.html`, locate `DEFAULT_FIREBASE_CONFIG` inside `<script type="module">`. It is pre-populated with your provisioned project (`gen-lang-client-0523990342`), or you can replace it with any Firebase Web App config:

```js
const DEFAULT_FIREBASE_CONFIG = {
  apiKey: "YOUR_FIREBASE_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT_ID-default-rtdb.firebaseio.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.firebasestorage.app",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_FIREBASE_APP_ID"
};
```

## 2. External Setup Requirements (Firebase Console)

Because `ProvisionFirebase` creates the Firebase project and Google Authentication, you must complete these steps in the [Firebase Console](https://console.firebase.google.com/) for Realtime Database to go live:

1. **Create Realtime Database**:
   - Open your Firebase project → **Build** → **Realtime Database** → **Create Database**.
   - Select your database region and start in **Locked mode**.
   - Copy the generated database URL (e.g. `https://gen-lang-client-0523990342-default-rtdb.firebaseio.com` or regional `.firebasedatabase.app` URL) into `DEFAULT_FIREBASE_CONFIG.databaseURL` in `index.html` (or click **Database URL** in the app's connection banner/settings).
2. **Deploy Realtime Database Security Rules**:
   - Open **Realtime Database** → **Rules** tab.
   - Paste the complete contents of `database.rules.json` and click **Publish**.
3. **Authentication Providers & Authorized Domains**:
   - **Google Sign-In** is ready to use (`signInWithPopup`).
   - *(Optional)* To use **Email & Password** login/registration, enable **Email/Password** under **Authentication** → **Sign-in method**.
   - Under **Authentication** → **Settings** → **Authorized domains**, ensure your hosted app domain (and `localhost` for local testing) is listed.

## 3. Security Architecture & Client-Only Limitations

- **Enforced by `database.rules.json`**:
  - Unauthenticated reads and writes are completely blocked (`".read": false`, `".write": false` at root).
  - First registered user (or verified bootstrap admin email) becomes `admin`; subsequent registrations can only self-register as `cashier` with `active: true` and cannot self-escalate roles.
  - Only `admin` users can create/edit/delete products, purchase prices, categories, suppliers, purchases, expenses, store settings, and user roles, or cancel completed sales.
  - Cashiers can read inventory/customers/settings, create customers, create `completed` sales under their own `auth.uid`, increment `invoiceCounter` by `+1`, and decrement `products/{id}/stock` (`>= 0`).
- **Trusted Server-Side Limitation**:
  - A pure client-only HTML application cannot independently verify pricing math, tax/discount calculations, or invoice subtotal integrity on a trusted server. Restrictive RTDB rules validate data types, non-negative bounds, ownership (`cashierUid === auth.uid`), and role permissions, while full financial enforcement against malicious REST clients would require Firebase Cloud Functions or a backend API.

## 4. Testing Checklist

- [ ] **Auth & Roles**: Sign in with Google or Email/Password; verify first user receives `admin` role in `stockdesk/users/{uid}`; sign in with a second account and verify `cashier` role hides Admin-only tabs (Purchases, Expenses, Settings, Users).
- [ ] **Realtime Connection & Offline Handling**: Check the sidebar Firebase RTDB indicator shows `Connected`; toggle browser offline mode and confirm status switches to `Offline`, banner appears, and billing is safely blocked until reconnected.
- [ ] **Inventory CRUD & Stock Alerts**: Add a product with SKU, barcode, purchase/selling price, stock, min stock, and unit; edit product; verify Low-Stock and Out-of-Stock badges and topbar counter update in real time.
- [ ] **Quick Billing (POS) & Atomic Deduction**: Search by name/SKU or scan barcode + Enter; add items; test +/- quantity limits against available stock; apply discount, GST, delivery, and Cash/UPI/Card payment; complete sale and confirm invoice number increments and stock deducts across two open browser tabs simultaneously.
- [ ] **Duplicate Prevention & Insufficient Stock**: Attempt to sell more than available stock or double-click Complete Sale; confirm transaction aborts cleanly without negative stock or duplicate invoices.
- [ ] **Receipt Printing & Reprint**: Toggle between `A4 Invoice` and `80mm Thermal` in the receipt modal, print/preview, and reprint from **Sales History**.
- [ ] **Bill Cancellation & Stock Restoration**: As Admin, cancel a completed bill in **Sales History** and verify sold quantities are restored to `products/{id}/stock` and logged in `stock_movements`.
- [ ] **Purchases & Suppliers**: Add a supplier, record a purchase invoice with multiple items, and confirm product stock increases and `stock_movements` logs the `purchase` entry.
- [ ] **Customers & Reports**: Add/edit a customer, complete a sale linked to that customer, inspect Customer Purchase History, switch Daily/Weekly/Monthly reports, and export CSV files.
