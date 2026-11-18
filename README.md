import React, { useState, useEffect } from 'react';
import { createClient } from '@supabase/supabase-js';

// 🔴 ใส่ค่า Supabase Project URL และ Anon Key ของคุณที่นี่
const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co';
const SUPABASE_ANON_KEY = 'YOUR_SUPABASE_ANON_KEY';
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

const ORDER_STATUSES = [
  'รอตรวจสอบการชำระเงิน', 'ชำระเงินแล้ว / สั่งสินค้าแล้ว', 'รอผู้ขายจัดส่ง', 
  'ผู้ขายจัดส่งแล้ว', 'สินค้าเข้าโกดังต่างประเทศ', 'กำลังส่งกลับไทย', 
  'เข้าโกดังไทยแล้ว', 'สินค้าถึงร้าน', 'กำลังจัดส่ง', 'จัดส่งสำเร็จ', 'ยกเลิก'
];

export default function App() {
  const [page, setPage] = useState('store'); 
  const [settings, setSettings] = useState({});
  const [products, setProducts] = useState([]);
  const [cart, setCart] = useState([]);
  const [user, setUser] = useState(null);

  const [form, setForm] = useState({ account_x: '', customer_name: '', phone: '', address: '', shipping: 'ems' });
  const [slipFile, setSlipFile] = useState(null);
  const [loading, setLoading] = useState(false);
  const [orderSuccess, setOrderSuccess] = useState(null);

  const [trackQuery, setTrackQuery] = useState('');
  const [trackedOrders, setTrackedOrders] = useState([]);

  const [adminTab, setAdminTab] = useState('orders');
  const [orders, setOrders] = useState([]);
  const [loginForm, setLoginForm] = useState({ email: '', password: '' });
  const [newProduct, setNewProduct] = useState({ name: '', price: '', stock: '', description: '' });
  const [productImage, setProductImage] = useState(null);

  useEffect(() => {
    fetchInitialData();
    supabase.auth.getSession().then(({ data: { session } }) => setUser(session?.user ?? null));
    supabase.auth.onAuthStateChange((_event, session) => setUser(session?.user ?? null));
  }, []);

  async function fetchInitialData() {
    const { data: setData } = await supabase.from('store_settings').select('*').eq('id', 1).single();
    if (setData) setSettings(setData);
    const { data: prodData } = await supabase.from('products').select('*').order('created_at', { ascending: false });
    if (prodData) setProducts(prodData);
  }

  async function fetchOrders() {
    const { data } = await supabase.from('orders').select('*').order('created_at', { ascending: false });
    if (data) setOrders(data);
  }

  useEffect(() => {
    if (page === 'admin' && user) fetchOrders();
  }, [page, user]);

  async function uploadImage(file) {
    if (!file) return '';
    const fileName = `${Date.now()}_${file.name}`;
    const { error } = await supabase.storage.from('shop-images').upload(fileName, file);
    if (error) return '';
    const { data } = supabase.storage.from('shop-images').getPublicUrl(fileName);
    return data.publicUrl;
  }

  const addToCart = (product) => {
    const exist = cart.find((item) => item.id === product.id);
    if (exist) {
      if (exist.qty < product.stock) {
        setCart(cart.map((item) => (item.id === product.id ? { ...item, qty: item.qty + 1 } : item)));
      }
    } else {
      setCart([...cart, { ...product, qty: 1 }]);
    }
  };

  const subtotal = cart.reduce((sum, item) => sum + item.price * item.qty, 0);
  const shippingCost = form.shipping === 'reg' ? (settings.shipping_reg || 35) : form.shipping === 'ems' ? (settings.shipping_ems || 50) : 0;
  const shippingLabel = form.shipping === 'reg' ? 'ส่งลงทะเบียน (ลทบ.)' : form.shipping === 'ems' ? 'ส่งด่วน EMS / เอกชน' : 'นัดรับ / ฝากส่งรวมรอบ';
  const totalPrice = subtotal + shippingCost;

  async function handleCheckout(e) {
    e.preventDefault();
    if (cart.length === 0 || !slipFile) return alert('กรุณาเลือกสินค้าและแนบสลิปโอนเงิน');
    setLoading(true);

    const orderId = 'ORD-' + Math.floor(100000 + Math.random() * 900000);
    const slipUrl = await uploadImage(slipFile);

    const orderData = {
      id: orderId,
      account_x: form.account_x,
      customer_name: form.customer_name,
      phone: form.phone,
      address: form.address,
      items: cart,
      shipping_method: shippingLabel,
      shipping_cost: shippingCost,
      total_price: totalPrice,
      slip_url: slipUrl,
    };

    const { error } = await supabase.from('orders').insert([orderData]);
    if (!error) {
      for (const item of cart) {
        await supabase.from('products').update({ stock: item.stock - item.qty }).eq('id', item.id);
      }
      if (settings.discord_webhook) {
        const itemsText = cart.map((i) => `• ${i.name} x${i.qty} (${i.price * i.qty} บ.)`).join('\n');
        await fetch(settings.discord_webhook, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            embeds: [{
              title: `🌸 มีออเดอร์ใหม่! (${orderId})`,
              color: 16753920,
              fields: [
                { name: 'Account X', value: form.account_x, inline: true },
                { name: 'ชื่อลูกค้า', value: form.customer_name, inline: true },
                { name: 'เบอร์โทร', value: form.phone, inline: true },
                { name: 'รายการสินค้า', value: itemsText },
                { name: 'การจัดส่ง', value: `${shippingLabel} (${shippingCost} บาท)`, inline: true },
                { name: 'ยอดโอนสุทธิ', value: `${totalPrice} บาท`, inline: true },
              ],
              image: { url: slipUrl }
            }]
          })
        });
      }
      setOrderSuccess(orderId);
      setCart([]);
      setSlipFile(null);
      fetchInitialData();
    }
    setLoading(false);
  }

  async function handleTrack(e) {
    e.preventDefault();
    const { data } = await supabase
      .from('orders')
      .select('*')
      .or(`id.eq.${trackQuery},account_x.ilike.%${trackQuery}%`)
      .order('created_at', { ascending: false });
    setTrackedOrders(data || []);
  }

  async function handleAddProduct(e) {
    e.preventDefault();
    setLoading(true);
    const imageUrl = await uploadImage(productImage);
    await supabase.from('products').insert([{
      name: newProduct.name,
      price: Number(newProduct.price),
      stock: Number(newProduct.stock),
      description: newProduct.description,
      image_url: imageUrl
    }]);
    setNewProduct({ name: '', price: '', stock: '', description: '' });
    setProductImage(null);
    fetchInitialData();
    setLoading(false);
  }

  async function handleSaveSettings(e) {
    e.preventDefault();
    setLoading(true);
    await supabase.from('store_settings').update(settings).eq('id', 1);
    alert('บันทึกการตั้งค่าร้านเรียบร้อย!');
    setLoading(false);
  }

  return (
    <div className="min-h-screen bg-pink-50/60 text-gray-700 font-sans">
      <header className="bg-white/90 backdrop-blur sticky top-0 z-10 border-b border-pink-100 shadow-sm">
        <div className="max-w-5xl mx-auto px-4 py-3 flex flex-wrap items-center justify-between gap-2">
          <div className="flex items-center gap-2 cursor-pointer" onClick={() => setPage('store')}>
            {settings.logo_url && <img src={settings.logo_url} alt="logo" className="w-9 h-9 rounded-full object-cover border border-pink-200" />}
            <span className="font-bold text-lg text-pink-500">{settings.store_name || 'Pastel Shop'}</span>
          </div>
          <nav className="flex items-center gap-3 text-sm font-medium">
            <button onClick={() => setPage('store')} className="hover:text-pink-500">สินค้าทั้งหมด</button>
            <button onClick={() => setPage('track')} className="hover:text-pink-500">เช็คสถานะ</button>
            <button onClick={() => { setPage('cart'); setOrderSuccess(null); }} className="bg-pink-400 text-white px-3 py-1.5 rounded-full shadow-sm hover:bg-pink-500">
              ตะกร้า ({cart.reduce((s, i) => s + i.qty, 0)})
            </button>
            <button onClick={() => setPage('admin')} className="text-xs text-purple-400 hover:underline ml-2">Admin</button>
          </nav>
        </div>
      </header>

      <main className="max-w-5xl mx-auto p-4">
        {page === 'store' && (
          <div>
            {settings.banner_url && (
              <div className="relative rounded-2xl overflow-hidden mb-6 shadow-sm border border-pink-100">
                <img src={settings.banner_url} alt="banner" className="w-full h-48 md:h-64 object-cover" />
                <div className="bg-white/90 p-4 text-center">
                  <h1 className="text-xl font-bold text-pink-600">{settings.store_name}</h1>
                  <p className="text-sm text-gray-500 mt-1">{settings.store_desc}</p>
                </div>
              </div>
            )}
            <h2 className="text-lg font-bold text-purple-600 mb-4">✨ รายการการ์ดและอัลบั้ม</h2>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
              {products.map((p) => (
                <div key={p.id} className="bg-white rounded-2xl p-3 shadow-sm border border-pink-100 flex flex-col justify-between">
                  <div>
                    <img src={p.image_url || 'https://placehold.co/300x300/fce7f3/db2777?text=No+Image'} alt={p.name} className="w-full h-40 object-cover rounded-xl mb-2" />
                    <h3 className="font-bold text-gray-800 text-sm line-clamp-2">{p.name}</h3>
                    <p className="text-xs text-gray-400 mt-1 line-clamp-2">{p.description}</p>
                  </div>
                  <div className="mt-3">
                    <div className="flex justify-between items-center mb-2">
                      <span className="font-bold text-pink-500">{p.price} ฿</span>
                      <span className="text-xs text-gray-400">เหลือ {p.stock}</span>
                    </div>
                    <button
                      disabled={p.stock <= 0}
                      onClick={() => addToCart(p)}
                      className={`w-full py-1.5 rounded-xl text-sm font-medium ${p.stock > 0 ? 'bg-purple-400 text-white hover:bg-purple-500' : 'bg-gray-200 text-gray-400 cursor-not-allowed'}`}
                    >
                      {p.stock > 0 ? '+ หยิบใส่ตะกร้า' : 'สินค้าหมด'}
                    </button>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

        {page === 'cart' && (
          <div className="max-w-xl mx-auto bg-white p-6 rounded-2xl shadow-sm border border-pink-100">
            {orderSuccess ? (
              <div className="text-center py-6">
                <div className="text-4xl mb-2">🎉</div>
                <h2 className="text-xl font-bold text-pink-500">สั่งซื้อสำเร็จ!</h2>
                <p className="text-sm text-gray-500 mt-2">เลขที่คำสั่งซื้อของคุณคือ:</p>
                <div className="text-2xl font-mono font-bold bg-pink-50 text-purple-600 py-2 px-4 rounded-xl my-3 inline-block">{orderSuccess}</div>
                <p className="text-xs text-gray-400">สามารถนำเลขนี้หรือ Account X ไปเช็คสถานะสินค้าได้เลยค่ะ</p>
              </div>
            ) : (
              <form onSubmit={handleCheckout} className="space-y-4">
                <h2 className="text-lg font-bold text-pink-500">🛒 ตะกร้าสินค้าและการชำระเงิน</h2>
                {cart.length === 0 ? <p className="text-sm text-gray-400">ยังไม่มีสินค้าในตะกร้า</p> : (
                  <>
                    <div className="space-y-2 border-b border-pink-100 pb-3">
                      {cart.map((item) => (
                        <div key={item.id} className="flex justify-between items-center text-sm">
                          <span>{item.name} x{item.qty}</span>
                          <div className="flex items-center gap-2">
                            <span>{item.price * item.qty} ฿</span>
                            <button type="button" onClick={() => setCart(cart.filter((c) => c.id !== item.id))} className="text-red-400 text-xs">ลบ</button>
                          </div>
                        </div>
                      ))}
                    </div>
                    <div>
                      <label className="block text-xs font-bold mb-1">เลือกวิธีจัดส่ง</label>
                      <select value={form.shipping} onChange={(e) => setForm({ ...form, shipping: e.target.value })} className="w-full border border-pink-200 rounded-xl p-2 text-sm">
                        <option value="reg">ส่งลงทะเบียน (ลทบ.) - {settings.shipping_reg || 35} บาท</option>
                        <option value="ems">ส่งด่วน EMS / เอกชน - {settings.shipping_ems || 50} บาท</option>
                        <option value="pickup">นัดรับ / ฝากส่งรวมรอบ - 0 บาท</option>
                      </select>
                    </div>
                    <div className="bg-purple-50 p-3 rounded-xl text-sm flex justify-between font-bold text-purple-700">
                      <span>ยอดโอนสุทธิ (รวมส่ง):</span>
                      <span>{totalPrice} บาท</span>
                    </div>
                    {settings.qr_url && (
                      <div className="text-center bg-pink-50/50 p-3 rounded-xl">
                        <p className="text-xs text-gray-500 mb-2">สแกน QR Code เพื่อชำระเงิน</p>
                        <img src={settings.qr_url} alt="QR Payment" className="w-44 mx-auto rounded-lg shadow-sm" />
                      </div>
                    )}
                    <input required placeholder="ชื่อ Account X (เช่น @username)" value={form.account_x} onChange={(e) => setForm({ ...form, account_x: e.target.value })} className="w-full border border-pink-200 rounded-xl p-2 text-sm" />
                    <input required placeholder="ชื่อ-นามสกุล ผู้รับ" value={form.customer_name} onChange={(e) => setForm({ ...form, customer_name: e.target.value })} className="w-full border border-pink-200 rounded-xl p-2 text-sm" />
                    <input required placeholder="เบอร์โทรศัพท์" value={form.phone} onChange={(e) => setForm({ ...form, phone: e.target.value })} className="w-full border border-pink-200 rounded-xl p-2 text-sm" />
                    <textarea required placeholder="ที่อยู่จัดส่งครบถ้วน" value={form.address} onChange={(e) => setForm({ ...form, address: e.target.value })} className="w-full border border-pink-200 rounded-xl p-2 text-sm" rows="2" />
                    <div>
                      <label className="block text-xs font-bold mb-1">อัปโหลดสลิปโอนเงิน *</label>
                      <input required type="file" accept="image/*" onChange={(e) => setSlipFile(e.target.files[0])} className="text-xs" />
                    </div>
                    <button disabled={loading} type="submit" className="w-full bg-pink-500 text-white py-2.5 rounded-xl font-bold hover:bg-pink-600">
                      {loading ? 'กำลังบันทึกออเดอร์...' : 'ยืนยันการสั่งซื้อและแจ้งโอน'}
                    </button>
                  </>
                )}
              </form>
            )}
          </div>
        )}

        {page === 'track' && (
          <div className="max-w-xl mx-auto bg-white p-6 rounded-2xl shadow-sm border border-pink-100">
            <h2 className="text-lg font-bold text-purple-600 mb-3">📦 ตรวจสอบสถานะสินค้า</h2>
            <form onSubmit={handleTrack} className="flex gap-2 mb-4">
              <input required placeholder="กรอกเลขที่ออเดอร์ (ORD-...) หรือชื่อ Account X" value={trackQuery} onChange={(e) => setTrackQuery(e.target.value)} className="flex-1 border border-pink-200 rounded-xl p-2 text-sm" />
              <button type="submit" className="bg-purple-500 text-white px-4 rounded-xl text-sm font-bold">ค้นหา</button>
            </form>
            <div className="space-y-3">
              {trackedOrders.map((o) => (
                <div key={o.id} className="border border-pink-100 rounded-xl p-4 bg-pink-50/30 text-sm">
                  <div className="flex justify-between font-bold text-pink-600">
                    <span>{o.id} ({o.account_x})</span>
                    <span>{o.total_price} ฿</span>
                  </div>
                  <div className="mt-2">สถานะ: <span className="bg-purple-100 text-purple-700 px-2 py-0.5 rounded-full font-bold text-xs">{o.status}</span></div>
                  {o.tracking_number && (
                    <div className="mt-2 text-xs bg-white p-2 rounded-lg border border-pink-100">
                      🚚 ขนส่ง: <b>{o.courier}</b> | เลขพัสดุ: <b className="text-pink-600">{o.tracking_number}</b>
                    </div>
                  )}
                </div>
              ))}
            </div>
          </div>
        )}

        {page === 'admin' && (
          <div className="bg-white p-6 rounded-2xl shadow-sm border border-pink-100">
            {!user ? (
              <form onSubmit={async (e) => {
                e.preventDefault();
                const { error } = await supabase.auth.signInWithPassword(loginForm);
                if (error) alert('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
              }} className="max-w-sm mx-auto space-y-3 py-6">
                <h2 className="text-lg font-bold text-center text-purple-600">🔐 เข้าสู่ระบบหลังบ้าน</h2>
                <input required type="email" placeholder="อีเมลแอดมิน" onChange={(e) => setLoginForm({ ...loginForm, email: e.target.value })} className="w-full border rounded-xl p-2 text-sm" />
                <input required type="password" placeholder="รหัสผ่าน" onChange={(e) => setLoginForm({ ...loginForm, password: e.target.value })} className="w-full border rounded-xl p-2 text-sm" />
                <button type="submit" className="w-full bg-purple-500 text-white py-2 rounded-xl font-bold text-sm">เข้าสู่ระบบ</button>
              </form>
            ) : (
              <div>
                <div className="flex justify-between items-center mb-4 border-b pb-3">
                  <div className="flex gap-2">
                    <button onClick={() => setAdminTab('orders')} className={`px-3 py-1.5 rounded-xl text-xs font-bold ${adminTab === 'orders' ? 'bg-pink-500 text-white' : 'bg-pink-50'}`}>1. ออเดอร์</button>
                    <button onClick={() => setAdminTab('products')} className={`px-3 py-1.5 rounded-xl text-xs font-bold ${adminTab === 'products' ? 'bg-pink-500 text-white' : 'bg-pink-50'}`}>2. สินค้า</button>
                    <button onClick={() => setAdminTab('settings')} className={`px-3 py-1.5 rounded-xl text-xs font-bold ${adminTab === 'settings' ? 'bg-pink-500 text-white' : 'bg-pink-50'}`}>3. ตั้งค่า</button>
                  </div>
                  <button onClick={() => supabase.auth.signOut()} className="text-xs text-red-400">ออกจากระบบ</button>
                </div>

                {adminTab === 'orders' && (
                  <div className="space-y-4">
                    {orders.map((o) => (
                      <div key={o.id} className="border border-pink-100 rounded-xl p-4 text-sm space-y-2">
                        <div className="flex flex-wrap justify-between font-bold">
                          <span className="text-pink-600">{o.id} | X: {o.account_x} ({o.customer_name})</span>
                          <span>ยอดโอน: {o.total_price} ฿</span>
                        </div>
                        <p className="text-xs text-gray-500">ที่อยู่: {o.address} | โทร: {o.phone}</p>
                        <div className="text-xs bg-gray-50 p-2 rounded">
                          สินค้า: {o.items.map((i) => `${i.name} (x${i.qty})`).join(', ')}
                        </div>
                        {o.slip_url && <a href={o.slip_url} target="_blank" rel="noreferrer" className="inline-block text-xs text-blue-500 underline">🖼️ เปิดดูสลิป</a>}
                        <div className="grid grid-cols-1 md:grid-cols-3 gap-2 pt-2">
                          <select
                            value={o.status}
                            onChange={async (e) => {
                              await supabase.from('orders').update({ status: e.target.value }).eq('id', o.id);
                              fetchOrders();
                            }}
                            className="border rounded-lg p-1.5 text-xs bg-purple-50 font-bold"
                          >
                            {ORDER_STATUSES.map((st) => <option key={st} value={st}>{st}</option>)}
                          </select>
                          <input placeholder="ขนส่ง" defaultValue={o.courier} onBlur={async (e) => { await supabase.from('orders').update({ courier: e.target.value }).eq('id', o.id); }} className="border rounded-lg p-1.5 text-xs" />
                          <input placeholder="เลขพัสดุ" defaultValue={o.tracking_number} onBlur={async (e) => { await supabase.from('orders').update({ tracking_number: e.target.value }).eq('id', o.id); }} className="border rounded-lg p-1.5 text-xs" />
                        </div>
                      </div>
                    ))}
                  </div>
                )}

                {adminTab === 'products' && (
                  <div className="space-y-6">
                    <form onSubmit={handleAddProduct} className="grid grid-cols-1 md:grid-cols-2 gap-3 bg-pink-50/50 p-4 rounded-xl">
                      <input required placeholder="ชื่อสินค้า" value={newProduct.name} onChange={(e) => setNewProduct({ ...newProduct, name: e.target.value })} className="border rounded-lg p-2 text-xs" />
                      <input required type="number" placeholder="ราคา" value={newProduct.price} onChange={(e) => setNewProduct({ ...newProduct, price: e.target.value })} className="border rounded-lg p-2 text-xs" />
                      <input required type="number" placeholder="สต็อก" value={newProduct.stock} onChange={(e) => setNewProduct({ ...newProduct, stock: e.target.value })} className="border rounded-lg p-2 text-xs" />
                      <input type="file" accept="image/*" onChange={(e) => setProductImage(e.target.files[0])} className="text-xs" />
                      <input placeholder="รายละเอียด" value={newProduct.description} onChange={(e) => setNewProduct({ ...newProduct, description: e.target.value })} className="border rounded-lg p-2 text-xs md:col-span-2" />
                      <button disabled={loading} type="submit" className="bg-pink-500 text-white py-2 rounded-lg text-xs font-bold md:col-span-2">+ เพิ่มสินค้า</button>
                    </form>
                    <div className="space-y-2">
                      {products.map((p) => (
                        <div key={p.id} className="flex items-center justify-between border p-2 rounded-xl text-xs">
                          <div className="flex items-center gap-2">
                            <img src={p.image_url} alt="" className="w-10 h-10 rounded object-cover" />
                            <div>
                              <p className="font-bold">{p.name}</p>
                              <p className="text-gray-400">ราคา: {p.price} ฿ | สต็อก: {p.stock}</p>
                            </div>
                          </div>
                          <button onClick={async () => { await supabase.from('products').delete().eq('id', p.id); fetchInitialData(); }} className="text-red-500 px-2">ลบ</button>
                        </div>
                      ))}
                    </div>
                  </div>
                )}

                {adminTab === 'settings' && (
                  <form onSubmit={handleSaveSettings} className="space-y-3 text-xs max-w-lg">
                    <div><label className="font-bold">ชื่อร้าน</label><input value={settings.store_name || ''} onChange={(e) => setSettings({ ...settings, store_name: e.target.value })} className="w-full border rounded-lg p-2 mt-1" /></div>
                    <div><label className="font-bold">คำอธิบาย</label><input value={settings.store_desc || ''} onChange={(e) => setSettings({ ...settings, store_desc: e.target.value })} className="w-full border rounded-lg p-2 mt-1" /></div>
                    <div className="grid grid-cols-2 gap-2">
                      <div><label className="font-bold">ค่าส่ง ลทบ.</label><input type="number" value={settings.shipping_reg || 35} onChange={(e) => setSettings({ ...settings, shipping_reg: Number(e.target.value) })} className="w-full border rounded-lg p-2 mt-1" /></div>
                      <div><label className="font-bold">ค่าส่ง EMS</label><input type="number" value={settings.shipping_ems || 50} onChange={(e) => setSettings({ ...settings, shipping_ems: Number(e.target.value) })} className="w-full border rounded-lg p-2 mt-1" /></div>
                    </div>
                    <div><label className="font-bold">Discord Webhook URL</label><input value={settings.discord_webhook || ''} onChange={(e) => setSettings({ ...settings, discord_webhook: e.target.value })} className="w-full border rounded-lg p-2 mt-1" /></div>
                    <div><label className="font-bold block">โลโก้ร้าน</label><input type="file" accept="image/*" onChange={async (e) => { const url = await uploadImage(e.target.files[0]); if (url) setSettings({ ...settings, logo_url: url }); }} className="mt-1" /></div>
                    <div><label className="font-bold block">แบนเนอร์</label><input type="file" accept="image/*" onChange={async (e) => { const url = await uploadImage(e.target.files[0]); if (url) setSettings({ ...settings, banner_url: url }); }} className="mt-1" /></div>
                    <div><label className="font-bold block">QR Code รับเงิน</label><input type="file" accept="image/*" onChange={async (e) => { const url = await uploadImage(e.target.files[0]); if (url) setSettings({ ...settings, qr_url: url }); }} className="mt-1" /></div>
                    <button disabled={loading} type="submit" className="bg-purple-500 text-white px-4 py-2 rounded-xl font-bold">บันทึกตั้งค่า</button>
                  </form>
                )}
              </div>
            )}
          </div>
        )}
      </main>
    </div>
  );
}
