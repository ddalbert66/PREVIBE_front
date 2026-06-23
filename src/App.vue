<script setup lang="ts">
import { computed, onMounted, reactive, ref } from "vue";

type AuthMode = "login" | "register";
type Member = {
  name: string;
  email: string;
  password: string;
  provider?: string;
};
type PersonalProduct = {
  id: string;
  ownerEmail: string;
  name: string;
  description: string;
  price: number;
  launchDate: string;
  imageDataUrl: string;
};

const apiBase = import.meta.env.VITE_API_BASE_URL ?? "http://localhost:8080/api";
const memberStorageKey = "previbe-members";
const productStorageKey = "previbe-personal-products";
const platformBank = {
  bankName: "PREVIBE 合作銀行",
  bankCode: "000",
  account: "000-0000-0000-0000",
  accountName: "PREVIBE 平台代收專戶",
};

const health = ref("Checking...");
const log = ref<string[]>([]);
const activeView = ref<"preorder" | "products" | "login">("preorder");
const authMode = ref<AuthMode>("login");
const authMessage = ref("");
const authError = ref("");
const currentMember = ref<Member | null>(null);

const seller = reactive({ name: "PREVIBE Studio", email: "seller@example.com", threadsUsername: "previbe_shop" });
const buyer = reactive({ name: "PREVIBE Buyer", email: "buyer@example.com" });
const product = reactive({
  name: "早鳥限定香氛禮盒",
  description: "從 Threads 貼文開放預購，支援訂單建立、付款模擬與到貨確認。",
  price: 1280,
});
const login = reactive({ email: "", password: "", remember: true });
const register = reactive({ name: "", email: "", password: "", confirmPassword: "" });
const personalProducts = ref<PersonalProduct[]>([]);
const productForm = reactive({
  name: "",
  description: "",
  price: "",
  launchDate: "",
  imageDataUrl: "",
});
const productFormError = ref("");
const productFormMessage = ref("");
const ids = reactive<{ buyerId?: number; sellerId?: number; productId?: number; postId?: number; orderId?: number }>({});
const metrics = reactive({ viewsCount: 1200, likesCount: 168, repliesCount: 12, repostsCount: 8, quotesCount: 3 });

const canCreateProduct = computed(() => ids.sellerId);
const canBindPost = computed(() => ids.sellerId && ids.productId);
const canOrder = computed(() => ids.buyerId && ids.productId && ids.postId);
const canPay = computed(() => ids.orderId);
const authTitle = computed(() => (authMode.value === "login" ? "登入" : "加入會員"));
const authSubtitle = computed(() =>
  authMode.value === "login" ? "登入會員以加快結帳速度與管理訂單" : "建立 PREVIBE 會員，保存預購與付款紀錄",
);
const authSubmitLabel = computed(() => (authMode.value === "login" ? "登入會員" : "立即加入"));
const myProducts = computed(() =>
  currentMember.value
    ? personalProducts.value.filter((item) => normalizeEmail(item.ownerEmail) === normalizeEmail(currentMember.value!.email))
    : [],
);

const galleryItems = [
  { title: "Mellow 香氛組", tag: "Lifestyle", tone: "mint" },
  { title: "City Walk 球鞋", tag: "Fashion", tone: "blue" },
  { title: "Roast Lab 咖啡豆", tag: "Food", tone: "amber" },
  { title: "Desk Set 桌面套件", tag: "Design", tone: "charcoal" },
  { title: "Crew Week 外套", tag: "Apparel", tone: "silver" },
  { title: "Light Bag 隨身包", tag: "Accessory", tone: "rose" },
];

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const response = await fetch(`${apiBase}${path}`, {
    headers: { "Content-Type": "application/json", ...(init?.headers ?? {}) },
    ...init,
  });
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`${response.status} ${body}`);
  }
  return response.json() as Promise<T>;
}

function record(message: string) {
  log.value = [`${new Date().toLocaleTimeString()} ${message}`, ...log.value].slice(0, 8);
}

function normalizeEmail(email: string) {
  return email.trim().toLowerCase();
}

function readMembers(): Member[] {
  try {
    return JSON.parse(localStorage.getItem(memberStorageKey) || "[]") as Member[];
  } catch {
    return [];
  }
}

function saveMembers(members: Member[]) {
  localStorage.setItem(memberStorageKey, JSON.stringify(members));
}

function readPersonalProducts(): PersonalProduct[] {
  try {
    return JSON.parse(localStorage.getItem(productStorageKey) || "[]") as PersonalProduct[];
  } catch {
    return [];
  }
}

function savePersonalProducts(products: PersonalProduct[]) {
  personalProducts.value = products;
  localStorage.setItem(productStorageKey, JSON.stringify(products));
}

function clearAuthFeedback() {
  authError.value = "";
  authMessage.value = "";
}

function switchAuthMode(mode: AuthMode) {
  authMode.value = mode;
  clearAuthFeedback();
  if (mode === "register") {
    register.email = login.email;
  } else {
    login.email = register.email || login.email;
  }
}

function setSignedInMember(member: Member, message: string) {
  currentMember.value = member;
  activeView.value = "preorder";
  authError.value = "";
  authMessage.value = message;
  buyer.name = member.name;
  buyer.email = member.email;
  record(message);
}

function validatePassword(password: string) {
  return password.length >= 6 && password.length <= 12;
}

async function handleRegister() {
  clearAuthFeedback();
  const name = register.name.trim();
  const email = normalizeEmail(register.email);
  const password = register.password.trim();

  if (!name) {
    authError.value = "請輸入會員姓名。";
    return;
  }
  if (!email) {
    authError.value = "請輸入電子信箱。";
    return;
  }
  if (!validatePassword(password)) {
    authError.value = "密碼需為 6 至 12 位數。";
    return;
  }
  if (password !== register.confirmPassword.trim()) {
    authError.value = "兩次輸入的密碼不一致。";
    return;
  }

  const members = readMembers();
  if (members.some((member) => normalizeEmail(member.email) === email)) {
    authError.value = "此電子信箱已加入會員，請直接登入。";
    authMode.value = "login";
    login.email = email;
    return;
  }

  try {
    const createdUser = await request<{ id: number; name: string; email: string; role: string }>("/users", {
      method: "POST",
      body: JSON.stringify({ name, email, role: "BUYER" }),
    });

    const member = { name: createdUser.name ?? name, email: createdUser.email ?? email, password };
    saveMembers([...members, member]);
    login.email = email;
    login.password = password;
    register.name = "";
    register.email = "";
    register.password = "";
    register.confirmPassword = "";
    authMode.value = "login";
    setSignedInMember(member, `歡迎加入 PREVIBE，${member.name}！`);
  } catch (error) {
    authError.value = `註冊失敗：${(error as Error).message}`;
  }
}

function handleLogin() {
  clearAuthFeedback();
  const email = normalizeEmail(login.email);
  const password = login.password.trim();

  if (!email || !password) {
    authError.value = "請輸入帳號與密碼。";
    return;
  }

  const member = readMembers().find((item) => normalizeEmail(item.email) === email);
  if (!member) {
    authError.value = "查無此會員，請先點選立即加入。";
    return;
  }
  if (member.password !== password) {
    authError.value = "密碼不正確，請重新輸入。";
    return;
  }

  setSignedInMember(member, `登入成功，歡迎回來 ${member.name}。`);
}

function signOut() {
  const name = currentMember.value?.name ?? "會員";
  currentMember.value = null;
  activeView.value = "preorder";
  login.password = "";
  authMessage.value = `${name} 已登出。`;
  authError.value = "";
  record(`${name} 已登出`);
}

function handleProductImage(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file) return;
  if (!file.type.startsWith("image/")) {
    productFormError.value = "請選擇圖片檔案。";
    return;
  }
  if (file.size > 2 * 1024 * 1024) {
    productFormError.value = "圖片大小請勿超過 2 MB。";
    return;
  }

  const reader = new FileReader();
  reader.onload = () => {
    productForm.imageDataUrl = String(reader.result);
    productFormError.value = "";
  };
  reader.readAsDataURL(file);
}

function createPersonalProduct() {
  productFormError.value = "";
  productFormMessage.value = "";
  if (!currentMember.value) {
    activeView.value = "login";
    return;
  }

  const price = Number(productForm.price);
  if (!productForm.name.trim() || !productForm.description.trim() || !productForm.launchDate) {
    productFormError.value = "請完整填寫產品名稱、介紹與預計上線日期。";
    return;
  }
  if (!Number.isFinite(price) || price <= 0) {
    productFormError.value = "請輸入正確的預購價格。";
    return;
  }
  if (!productForm.imageDataUrl) {
    productFormError.value = "請上傳產品照片。";
    return;
  }
  const newProduct: PersonalProduct = {
    id: crypto.randomUUID(),
    ownerEmail: currentMember.value.email,
    name: productForm.name.trim(),
    description: productForm.description.trim(),
    price,
    launchDate: productForm.launchDate,
    imageDataUrl: productForm.imageDataUrl,
  };
  savePersonalProducts([newProduct, ...personalProducts.value]);
  Object.assign(productForm, {
    name: "",
    description: "",
    price: "",
    launchDate: "",
    imageDataUrl: "",
  });
  productFormMessage.value = "產品已新增，顯示於你的預購產品列表。";
  record(`新增預購產品：${newProduct.name}`);
}

function removePersonalProduct(id: string) {
  savePersonalProducts(personalProducts.value.filter((item) => item.id !== id));
  productFormMessage.value = "產品已刪除。";
}

function socialLogin(provider: "Facebook" | "Google" | "LINE") {
  const email = `${provider.toLowerCase()}@previbe.demo`;
  const member = { name: `${provider} 會員`, email, password: "social-login", provider };
  const members = readMembers();
  if (!members.some((item) => normalizeEmail(item.email) === email)) {
    saveMembers([...members, member]);
  }
  setSignedInMember(member, `已使用 ${provider} 帳號登入。`);
}

async function checkHealth() {
  try {
    const data = await request<{ status: string; service: string }>("/health");
    health.value = `${data.service} ${data.status}`;
    record("後端連線成功");
  } catch (error) {
    health.value = "Backend offline";
    record(`後端連線失敗：${(error as Error).message}`);
  }
}

async function createBuyer() {
  const data = await request<{ id: number }>("/users", {
    method: "POST",
    body: JSON.stringify({ ...buyer, role: "BUYER" }),
  });
  ids.buyerId = data.id;
  record(`建立買家 #${data.id}`);
}

async function createSeller() {
  const data = await request<{ id: number }>("/sellers", {
    method: "POST",
    body: JSON.stringify(seller),
  });
  ids.sellerId = data.id;
  record(`建立賣家 #${data.id}`);
}

async function createProduct() {
  const data = await request<{ id: number }>("/products", {
    method: "POST",
    body: JSON.stringify({
      sellerId: ids.sellerId,
      name: product.name,
      description: product.description,
      price: product.price,
      preorderStartAt: "2026-05-21",
      preorderEndAt: "2026-06-20",
      estimatedShipAt: "2026-07-15",
    }),
  });
  ids.productId = data.id;
  record(`建立預購商品 #${data.id}`);
}

async function bindThreadsPost() {
  const data = await request<{ id: number }>("/threads/posts", {
    method: "POST",
    body: JSON.stringify({
      sellerId: ids.sellerId,
      productId: ids.productId,
      threadsPostId: `threads_${Date.now()}`,
      postUrl: "https://www.threads.net/@previbe_shop/post/demo",
    }),
  });
  ids.postId = data.id;
  record(`綁定 Threads 貼文 #${data.id}`);
}

async function updateMetrics() {
  await request(`/threads/posts/${ids.postId}/metrics/manual`, {
    method: "POST",
    body: JSON.stringify(metrics),
  });
  record("更新 Threads 互動數據");
}

async function createOrder() {
  const data = await request<{ id: number }>("/orders", {
    method: "POST",
    body: JSON.stringify({ buyerId: ids.buyerId, productId: ids.productId, threadsPostId: ids.postId, quantity: 1 }),
  });
  ids.orderId = data.id;
  record(`建立訂單 #${data.id}`);
}

async function payOrder() {
  await request(`/orders/${ids.orderId}/pay/credit-card/simulate`, { method: "POST" });
  record("模擬付款成功，款項進入履約保管");
}

async function releaseEscrow() {
  await request(`/orders/${ids.orderId}/confirm-received`, { method: "POST" });
  record("確認收貨，釋放款項給賣家");
}

onMounted(() => {
  personalProducts.value = readPersonalProducts();
  checkHealth();
});
</script>

<template>
  <main class="page-shell">
    <header class="site-header">
      <button class="brand" type="button" @click="activeView = 'preorder'">
        <span class="brand-mark" aria-hidden="true"></span>
        <span>PREVIBE</span>
      </button>
      <nav class="nav-links" aria-label="主要導覽">
        <button :class="{ active: activeView === 'preorder' }" type="button" @click="activeView = 'preorder'">LINEUP</button>
        <button v-if="currentMember" :class="{ active: activeView === 'products' }" type="button" @click="activeView = 'products'">
          MY PRODUCTS
        </button>
        <button :class="{ active: activeView === 'login' }" type="button" @click="activeView = 'login'">
          {{ currentMember ? "MEMBER" : "LOGIN" }}
        </button>
        <a href="https://www.threads.net/@previbe_shop" target="_blank" rel="noreferrer">THREADS</a>
      </nav>
      <p class="header-note">把 Threads 熱度轉成可信任的預購訂單。</p>
    </header>

    <section v-if="activeView === 'preorder'" class="preorder-view">
      <section v-if="personalProducts.length" class="public-products">
        <div class="public-products-heading">
          <div>
            <p class="eyebrow">NEW PREORDERS</p>
            <h1>最新預購商品</h1>
          </div>
          <span>{{ personalProducts.length }} 項商品</span>
        </div>
        <div class="public-product-grid">
          <article v-for="item in personalProducts" :key="item.id" class="public-product-card">
            <img :src="item.imageDataUrl" :alt="item.name" />
            <div class="public-product-copy">
              <p class="eyebrow">預計上線 {{ item.launchDate }}</p>
              <h2>{{ item.name }}</h2>
              <p>{{ item.description }}</p>
              <strong>NT$ {{ item.price.toLocaleString() }}</strong>
              <p class="payment-summary">平台代收代管 · 下單後產生專屬付款識別碼</p>
            </div>
          </article>
        </div>
      </section>

      <section class="gallery-grid" aria-label="預購商品列表">
        <article v-for="item in galleryItems" :key="item.title" class="gallery-card">
          <div class="thumb" :class="item.tone">
            <span>{{ item.tag }}</span>
          </div>
          <h2>{{ item.title }}</h2>
        </article>
      </section>

      <section class="product-layout">
        <article class="product-feature">
          <div class="product-art">
            <span class="product-label">PREORDER</span>
            <strong>{{ product.name }}</strong>
          </div>
          <div class="product-copy">
            <p class="eyebrow">Threads @{{ seller.threadsUsername }}</p>
            <h1>{{ product.name }}</h1>
            <p>{{ product.description }}</p>
            <div class="price-row">
              <strong>NT$ {{ product.price.toLocaleString() }}</strong>
              <span>預購至 2026.06.20</span>
            </div>
          </div>
        </article>

        <aside class="workflow-panel" aria-label="預購流程操作">
          <div class="status-pill">{{ health }}</div>
          <h2>預購流程</h2>
          <div class="actions">
            <button @click="createBuyer">建立買家</button>
            <button @click="createSeller">建立賣家</button>
            <button :disabled="!canCreateProduct" @click="createProduct">建立商品</button>
            <button :disabled="!canBindPost" @click="bindThreadsPost">綁定 Threads</button>
            <button :disabled="!ids.postId" @click="updateMetrics">更新數據</button>
            <button :disabled="!canOrder" @click="createOrder">建立訂單</button>
            <button :disabled="!canPay" @click="payOrder">模擬付款</button>
            <button :disabled="!canPay" @click="releaseEscrow">確認收貨</button>
          </div>
        </aside>
      </section>

      <section class="dashboard-row">
        <article class="data-panel">
          <h2>流程編號</h2>
          <dl>
            <dt>Buyer</dt><dd>#{{ ids.buyerId ?? "-" }}</dd>
            <dt>Seller</dt><dd>#{{ ids.sellerId ?? "-" }}</dd>
            <dt>Product</dt><dd>#{{ ids.productId ?? "-" }}</dd>
            <dt>Threads</dt><dd>#{{ ids.postId ?? "-" }}</dd>
            <dt>Order</dt><dd>#{{ ids.orderId ?? "-" }}</dd>
          </dl>
        </article>

        <article class="data-panel metrics">
          <h2>Threads 熱度</h2>
          <div><span>瀏覽</span><strong>{{ metrics.viewsCount.toLocaleString() }}</strong></div>
          <div><span>喜歡</span><strong>{{ metrics.likesCount.toLocaleString() }}</strong></div>
          <div><span>回覆</span><strong>{{ metrics.repliesCount }}</strong></div>
          <div><span>轉發</span><strong>{{ metrics.repostsCount }}</strong></div>
        </article>

        <article class="data-panel activity">
          <h2>最新紀錄</h2>
          <ul>
            <li v-for="item in log" :key="item">{{ item }}</li>
          </ul>
        </article>
      </section>
    </section>

    <section v-else-if="activeView === 'products'" class="products-view">
      <div class="section-heading">
        <p class="eyebrow">PREVIBE SELLER STUDIO</p>
        <h1>我的預購產品</h1>
        <p>建立自己的預購頁面，整理照片、介紹、價格與上線日期。</p>
      </div>

      <div class="product-studio">
        <form class="product-editor" @submit.prevent="createPersonalProduct">
          <h2>新增預購產品</h2>
          <label>
            <span>產品照片</span>
            <input type="file" accept="image/*" @change="handleProductImage" />
          </label>
          <div v-if="productForm.imageDataUrl" class="upload-preview">
            <img :src="productForm.imageDataUrl" alt="產品照片預覽" />
          </div>
          <label>
            <span>產品名稱</span>
            <input v-model="productForm.name" type="text" placeholder="例如：夏日限定手作杯" />
          </label>
          <label>
            <span>產品介紹</span>
            <textarea v-model="productForm.description" rows="4" placeholder="輸入產品特色、材質或預購說明"></textarea>
          </label>
          <div class="editor-grid">
            <label>
              <span>預購價格</span>
              <input v-model="productForm.price" type="number" min="1" placeholder="NT$" />
            </label>
            <label>
              <span>預計上線日期</span>
              <input v-model="productForm.launchDate" type="date" />
            </label>
          </div>
          <div class="platform-payment-note">
            <strong>平台代收代管</strong>
            <p>買家款項統一匯入 PREVIBE 平台專戶。產品建立時不需要填寫私人銀行帳號。</p>
          </div>
          <p v-if="productFormError" class="auth-feedback error">{{ productFormError }}</p>
          <p v-if="productFormMessage" class="auth-feedback success">{{ productFormMessage }}</p>
          <button class="primary-action" type="submit">新增產品</button>
        </form>

        <section class="my-product-list" aria-label="我的預購產品列表">
          <div class="list-heading">
            <h2>產品列表</h2>
            <span>{{ myProducts.length }} 項產品</span>
          </div>
          <p v-if="!myProducts.length" class="empty-state">尚未建立產品，從左側表單新增第一個預購商品。</p>
          <article v-for="item in myProducts" :key="item.id" class="personal-product-card">
            <img :src="item.imageDataUrl" :alt="item.name" />
            <div>
              <p class="eyebrow">預計上線 {{ item.launchDate }}</p>
              <h3>{{ item.name }}</h3>
              <p>{{ item.description }}</p>
              <strong>NT$ {{ item.price.toLocaleString() }}</strong>
              <p class="seller-payment-state">交易方式：PREVIBE 平台代收代管</p>
            </div>
            <button class="delete-action" type="button" aria-label="刪除產品" title="刪除產品" @click="removePersonalProduct(item.id)">×</button>
          </article>
        </section>
      </div>
      <section class="escrow-panel">
        <div>
          <p class="eyebrow">PLATFORM ESCROW</p>
          <h2>平台公用銀行專戶</h2>
          <p>買家建立訂單後，系統才提供平台專戶與該筆訂單的付款識別碼。賣家不直接向買家收款。</p>
        </div>
        <dl class="escrow-bank">
          <dt>銀行</dt><dd>{{ platformBank.bankName }} ({{ platformBank.bankCode }})</dd>
          <dt>帳號</dt><dd>{{ platformBank.account }}</dd>
          <dt>戶名</dt><dd>{{ platformBank.accountName }}</dd>
        </dl>
        <div class="escrow-rules">
          <strong>入帳檢核</strong>
          <ul>
            <li>每筆訂單產生唯一付款識別碼，限該筆訂單使用。</li>
            <li>核對訂單金額、付款識別碼與買家資料後才標記已付款。</li>
            <li>無訂單、金額不符或來源異常的款項進入人工審核，不自動認列。</li>
            <li>賣家出貨、買家確認收貨後，平台才進行款項釋放。</li>
          </ul>
        </div>
      </section>
    </section>

    <section v-else class="login-view">
      <div class="breadcrumb">{{ currentMember ? "會員資料" : "登入/註冊" }}</div>
      <div v-if="currentMember" class="profile-panel">
        <p class="eyebrow">PREVIBE MEMBER</p>
        <h1>會員資料</h1>
        <dl class="profile-details">
          <dt>會員姓名</dt><dd>{{ currentMember.name }}</dd>
          <dt>電子信箱</dt><dd>{{ currentMember.email }}</dd>
          <dt>登入方式</dt><dd>{{ currentMember.provider || "PREVIBE 帳號" }}</dd>
        </dl>
        <div class="profile-actions">
          <button type="button" class="primary-action" @click="activeView = 'preorder'">回到首頁</button>
          <button type="button" class="secondary-action" @click="signOut">登出</button>
        </div>
      </div>
      <div v-else class="login-card">
        <form class="login-form" @submit.prevent="authMode === 'login' ? handleLogin() : handleRegister()">
          <p class="eyebrow">{{ authSubtitle }}</p>
          <h1>{{ authTitle }}</h1>

          <label v-if="authMode === 'register'">
            <span>姓名</span>
            <input v-model="register.name" type="text" autocomplete="name" placeholder="請輸入會員姓名" />
          </label>
          <label v-if="authMode === 'login'">
            <span>帳號</span>
            <input v-model="login.email" type="email" autocomplete="email" placeholder="請輸入您的電子信箱" />
          </label>
          <label v-else>
            <span>帳號</span>
            <input v-model="register.email" type="email" autocomplete="email" placeholder="請輸入您的電子信箱" />
          </label>
          <label v-if="authMode === 'login'">
            <span>密碼</span>
            <input v-model="login.password" type="password" autocomplete="current-password" placeholder="請輸入 6 至 12 位數密碼" />
          </label>
          <label v-else>
            <span>密碼</span>
            <input v-model="register.password" type="password" autocomplete="new-password" placeholder="請輸入 6 至 12 位數密碼" />
          </label>
          <label v-if="authMode === 'register'">
            <span>確認</span>
            <input v-model="register.confirmPassword" type="password" autocomplete="new-password" placeholder="請再次輸入密碼" />
          </label>

          <p v-if="authError" class="auth-feedback error">{{ authError }}</p>
          <p v-if="authMessage" class="auth-feedback success">{{ authMessage }}</p>

          <button class="primary-action" type="submit">{{ authSubmitLabel }}</button>
          <div v-if="authMode === 'login'" class="form-options">
            <label class="remember"><input v-model="login.remember" type="checkbox" /> 記住我</label>
            <button type="button" class="text-link">忘記密碼？</button>
          </div>
          <p class="signup-line">
            <template v-if="authMode === 'login'">
              尚未加入 PREVIBE 會員嗎？
              <button type="button" class="text-link strong" @click="switchAuthMode('register')">立即加入</button>
            </template>
            <template v-else>
              已經是 PREVIBE 會員嗎？
              <button type="button" class="text-link strong" @click="switchAuthMode('login')">回到登入</button>
            </template>
          </p>
        </form>

        <div class="divider"><span>or</span></div>

        <div class="social-login">
          <p>使用以下方式快速登入</p>
          <button class="facebook" type="button" @click="socialLogin('Facebook')">使用 Facebook 帳號登入</button>
          <button class="google" type="button" @click="socialLogin('Google')">使用 Google 帳號登入</button>
          <button class="line" type="button" @click="socialLogin('LINE')">使用 LINE 帳號登入</button>
        </div>
      </div>
    </section>
  </main>
</template>
