# Frontend
Project: Website bán đồ nội thất (Demo full‑stack)

> Tài liệu này chứa mã mẫu cho một website bán đồ nội thất: Frontend (Next.js / React) + Backend (Express) + Database nhẹ (lowdb - file JSON) + Cart + Mock thanh toán. Bạn có thể copy toàn bộ project này vào máy để chạy thử.




---

Cấu trúc project (gợi ý)

furniture-store-demo/
├─ frontend/           # Next.js app (React)
│  ├─ package.json
│  ├─ pages/
│  │  ├─ _app.js
│  │  ├─ index.js
│  │  └─ api/          # optional serverless API (not used if bạn chạy backend riêng)
│  ├─ components/
│  │  └─ CartSidebar.jsx
│  └─ styles/
├─ backend/            # Express API
│  ├─ package.json
│  ├─ index.js         # server chính
│  ├─ routes/
│  │  ├─ products.js
│  │  └─ orders.js
│  └─ db.json          # lowdb database file (products, orders)
└─ README.md


---

1) Frontend (Next.js) — ý chính

pages/index.js hiển thị danh mục, sản phẩm nổi bật.

Gọi API backend để lấy danh sách sản phẩm: GET /api/products.

Khi khách bấm "Thêm vào giỏ" sẽ lưu vào state và mở CartSidebar.

Khi bấm "Thanh toán" trong Cart gửi POST tới POST /api/orders với payload đơn hàng (items, customer info).


Chạy frontend (từ thư mục frontend):

npm install
npm run dev


---

2) Backend (Express) — ý chính

GET /api/products — trả về danh sách sản phẩm từ db.json.

POST /api/orders — nhận đơn hàng, lưu vào db.json dưới orders, trả về paymentUrl giả lập.

GET /api/orders/:id — lấy trạng thái đơn hàng.


Chạy backend (từ thư mục backend):

npm install
node index.js
# (hoặc nodemon index.js nếu cài nodemon)

Gợi ý nội dung db.json (mẫu):

{
  "products": [
    { "id": 1, "name": "Sofa Modern", "price": 2500000, "image": "/images/sofa.jpg", "category": "Sofa" },
    { "id": 2, "name": "Giường Gỗ", "price": 3500000, "image": "/images/bed.jpg", "category": "Giường" }
  ],
  "orders": []
}


---

3) Code backend (file: backend/index.js) — MẪU

const express = require('express')
const bodyParser = require('body-parser')
const { Low, JSONFile } = require('lowdb')
const cors = require('cors')
const { nanoid } = require('nanoid')

const app = express()
app.use(cors())
app.use(bodyParser.json())

const adapter = new JSONFile('./db.json')
const db = new Low(adapter)

async function initDb() {
  await db.read()
  db.data ||= { products: [], orders: [] }
  await db.write()
}

initDb()

// Lấy danh sách sản phẩm
app.get('/api/products', async (req, res) => {
  await db.read()
  res.json(db.data.products)
})

// Tạo đơn hàng
app.post('/api/orders', async (req, res) => {
  const { items, customer } = req.body
  if (!items || items.length === 0) return res.status(400).json({ error: 'Cart empty' })

  await db.read()
  const id = nanoid()
  const total = items.reduce((s, it) => s + (it.price * (it.quantity || 1)), 0)
  const order = { id, items, customer, total, status: 'pending', createdAt: new Date().toISOString() }

  db.data.orders.push(order)
  await db.write()

  // Mock payment URL: trong thực tế bạn sẽ gọi API VNPAY/Momo và trả về url
  const paymentUrl = `http://localhost:4000/mock-pay/${id}`

  res.json({ orderId: id, paymentUrl })
})

// Mock trang thanh toán (mô phỏng cổng trả về)
app.get('/mock-pay/:orderId', async (req, res) => {
  const { orderId } = req.params
  // 1. Hiển thị nút thanh toán
  // 2. Khi user bấm, redirect về /api/mock-pay-success/:orderId
  res.send(`
    <h1>Thanh toán giả lập cho đơn ${orderId}</h1>
    <form method="GET" action="/api/mock-pay-success/${orderId}">
      <button type="submit">Thanh toán (mô phỏng)</button>
    </form>
  `)
})

// Khi thanh toán thành công (mô phỏng)
app.get('/api/mock-pay-success/:orderId', async (req, res) => {
  const { orderId } = req.params
  await db.read()
  const order = db.data.orders.find(o => o.id === orderId)
  if (!order) return res.status(404).send('Order not found')
  order.status = 'paid'
  order.paidAt = new Date().toISOString()
  await db.write()
  res.send(`<h1>Thanh toán thành công cho đơn ${orderId}</h1>`) 
})

// Lấy đơn hàng
app.get('/api/orders/:id', async (req, res) => {
  const { id } = req.params
  await db.read()
  const order = db.data.orders.find(o => o.id === id)
  if (!order) return res.status(404).json({ error: 'Not found' })
  res.json(order)
})

const PORT = process.env.PORT || 4000
app.listen(PORT, () => console.log('Backend running on', PORT))


---

4) Thanh toán thực tế (hướng dẫn nhanh)

Để tích hợp VNPay / Momo / ZaloPay bạn thường phải:

1. Đăng ký tài khoản merchant với nhà cung cấp.


2. Lấy partnerCode/merchantId, secretKey, returnUrl, notifyUrl.


3. Tạo endpoint backend POST /create-payment để tạo signature và redirect khách sang cổng.


4. Cổng sẽ redirect về returnUrl sau khi thanh toán — bạn phải verify signature và cập nhật trạng thái order.



Mình đã để một endpoint mock (/mock-pay/:orderId) để bạn thử luồng thanh toán trong demo trước khi đưa cổng thật.


---

5) Gợi ý UI: CartSidebar component

CartSidebar.jsx sẽ hiển thị items, tổng tiền, form thông tin khách (tên, sđt, địa chỉ) và nút Thanh toán.

Khi bấm Thanh toán gọi API POST /api/orders và nhận paymentUrl → window.location = paymentUrl.



---

6) Hướng dẫn nhanh để mình deploy giúp bạn (tùy chọn)

Nếu bạn muốn, mình có thể chuẩn bị repository sẵn (Git) và hướng dẫn deploy lên: Vercel (frontend) + Render/Heroku/ Railway (backend). Bạn muốn mình làm tiếp phần này không?



---

Ghi chú

Tài liệu này là demo mẫu, chưa có xác thực, chưa xử lý lỗi nâng cao, bảo mật hoặc xác thực CSRF. Khi đưa lên production cần bổ sung bảo mật, HTTPS và validate input.



---

Xong — code demo full‑stack đã được ghi ở trong tài liệu này. Nếu bạn muốn mình tiếp tục:

Tạo repo Git hoàn chỉnh và upload code, hoặc

Thêm tích hợp Momo/VNPay thực tế (mình sẽ cần thông tin merchant để test), hoặc

Triển khai lên hosting (Vercel + Render).


