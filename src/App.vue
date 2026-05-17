<template>
  <main class="container">
    <header v-if="user && !needsDisplayName" class="app-header-fixed">
      <div class="app-header-text">
        <h1>家族用チャット</h1>
        <p class="login-status">{{ displayName }} でログイン中</p>
      </div>
      <div class="header-actions">
        <button class="logout-btn" @click="handleLogout">ログアウト</button>
      </div>
    </header>

    <h1 v-else>家族用チャット</h1>

    <section v-if="!user" class="card">
      <h2>ログイン</h2>
      <form @submit.prevent="handleLogin" class="form">
        <label>
          メールアドレス
          <input v-model="email" type="email" required autocomplete="email" />
        </label>
        <label>
          パスワード
          <input v-model="password" type="password" required autocomplete="current-password" />
        </label>
        <button type="submit">ログイン</button>
      </form>
      <p v-if="error" class="error">{{ error }}</p>
    </section>

    <section v-else class="card">
      <div v-if="needsDisplayName">
        <h2>表示名の設定</h2>
        <form @submit.prevent="saveDisplayName" class="form">
          <label>
            表示名
            <input v-model="nameInput" type="text" required maxlength="30" />
          </label>
          <button type="submit">保存</button>
        </form>
      </div>

      <template v-else>
      <div v-if="renaming" class="rename-box">
        <h2>表示名の変更</h2>
        <form @submit.prevent="saveDisplayName" class="form">
          <label>
            表示名
            <input v-model="nameInput" type="text" required maxlength="30" />
          </label>
          <div class="form-actions">
            <button type="button" @click="cancelRename">キャンセル</button>
            <button type="submit">保存</button>
          </div>
        </form>
      </div>
      <div class="content-scroll">
        <div v-if="screen === 'list'">
          <div class="list-head">
            <button class="plus" @click="openAdd" aria-label="追加">+</button>
            <h2>コメント一覧（最新20件）</h2>
          </div>
          <ul class="list">
            <li v-for="item in messages" :key="item.id" class="item">
              <div class="meta">
                <strong>{{ item.displayName }}</strong>
                <span class="meta-time">
                  <span v-if="item.readByMom" class="read-check" aria-label="読了">✓</span>
                  <span>{{ formatJst(item.createdAt) }}</span>
                </span>
              </div>
              <div class="body">{{ item.text }}</div>
              <div class="item-actions">
                <button class="edit" @click="openEdit(item)" aria-label="編集">✎</button>
              </div>
            </li>
          </ul>
        </div>

        <div v-else-if="screen === 'add'">
          <h2>コメント追加</h2>
          <form @submit.prevent="submitAdd" class="form">
            <label>
              コメント
              <textarea v-model="draft" required maxlength="500"></textarea>
            </label>
            <div class="form-actions">
              <button type="button" @click="backToList">戻る</button>
              <button type="button" @click="toggleReadByMom">
                {{ readByMomDraft ? '読了済み' : '読んだ' }}
              </button>
              <button type="submit">登録</button>
            </div>
          </form>
        </div>

        <div v-else>
          <h2>コメント編集</h2>
          <form @submit.prevent="submitEdit" class="form">
            <label>
              コメント
              <textarea v-model="draft" required maxlength="500"></textarea>
            </label>
            <div class="form-actions">
              <button type="button" @click="backToList">戻る</button>
              <button type="submit">登録</button>
            </div>
          </form>
        </div>

        <p v-if="error" class="error">{{ error }}</p>
      </div>
      </template>
    </section>
  </main>
</template>

<script setup>
import { computed, onMounted, ref } from 'vue'
import {
  onAuthStateChanged,
  signInWithEmailAndPassword,
  signOut,
  updateProfile
} from 'firebase/auth'
import {
  limitToLast,
  onValue,
  orderByChild,
  push,
  query,
  ref as dbRef,
  set,
  update
} from 'firebase/database'
import { auth, db } from './firebase'

const email = ref('')
const password = ref('')
const user = ref(null)
const error = ref('')
const messages = ref([])
const screen = ref('list')
const draft = ref('')
const editingId = ref('')
const nameInput = ref('')
const renaming = ref(false)
const readByMomDraft = ref(false)

const displayName = computed(() => {
  if (!user.value) return ''
  return user.value.displayName || user.value.email?.split('@')[0] || '未設定'
})
const needsDisplayName = computed(() => Boolean(user.value && !user.value.displayName))

function formatJst(epochMs) {
  const dt = new Date(epochMs)
  const formatter = new Intl.DateTimeFormat('ja-JP', {
    timeZone: 'Asia/Tokyo',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    hour12: false
  })
  const parts = formatter.formatToParts(dt)
  const map = Object.fromEntries(parts.map((part) => [part.type, part.value]))
  return `${map.year}/${map.month}/${map.day} ${map.hour}:${map.minute}`
}

async function handleLogin() {
  error.value = ''
  try {
    await signInWithEmailAndPassword(auth, email.value, password.value)
    password.value = ''
    if (!needsDisplayName.value) {
      nameInput.value = auth.currentUser?.displayName || auth.currentUser?.email?.split('@')[0] || ''
      renaming.value = true
    }
  } catch (e) {
    error.value = 'ログインに失敗しました。入力値を確認してください。'
  }
}

async function handleLogout() {
  error.value = ''
  try {
    await signOut(auth)
    screen.value = 'list'
    draft.value = ''
    renaming.value = false
  } catch (e) {
    error.value = 'ログアウトに失敗しました。'
  }
}

async function saveDisplayName() {
  error.value = ''
  try {
    const trimmed = nameInput.value.trim()
    if (!user.value || !trimmed) return
    await updateProfile(user.value, { displayName: trimmed })
    user.value = auth.currentUser
    nameInput.value = ''
    renaming.value = false
  } catch (e) {
    error.value = '表示名の保存に失敗しました。'
  }
}

function cancelRename() {
  renaming.value = false
}

function openAdd() {
  draft.value = ''
  editingId.value = ''
  readByMomDraft.value = false
  screen.value = 'add'
}

function openEdit(item) {
  editingId.value = item.id
  draft.value = item.text
  readByMomDraft.value = Boolean(item.readByMom)
  screen.value = 'edit'
}

function backToList() {
  screen.value = 'list'
  draft.value = ''
  editingId.value = ''
  readByMomDraft.value = false
}

function toggleReadByMom() {
  readByMomDraft.value = !readByMomDraft.value
}

async function submitAdd() {
  error.value = ''
  try {
    const now = Date.now()
    const node = push(dbRef(db, 'messages'))
    await set(node, {
      uid: user.value.uid,
      displayName: displayName.value,
      text: draft.value.trim(),
      createdAt: now,
      updatedAt: now,
      readByMom: false
    })
    backToList()
  } catch (e) {
    error.value = '登録に失敗しました。'
  }
}

async function submitEdit() {
  error.value = ''
  try {
    if (!editingId.value) return
    await update(dbRef(db, `messages/${editingId.value}`), {
      text: draft.value.trim(),
      updatedAt: Date.now(),
      readByMom: readByMomDraft.value
    })
    backToList()
  } catch (e) {
    error.value = '更新に失敗しました。'
  }
}

onMounted(() => {
  onAuthStateChanged(auth, (current) => {
    user.value = current
    if (current && !current.displayName) {
      nameInput.value = current.email?.split('@')[0] || ''
    }
    if (!current) {
      messages.value = []
      nameInput.value = ''
      renaming.value = false
      return
    }
    const q = query(dbRef(db, 'messages'), orderByChild('createdAt'), limitToLast(20))
    onValue(q, (snapshot) => {
      const val = snapshot.val() || {}
      const list = Object.entries(val).map(([id, v]) => ({ id, ...v }))
      list.sort((a, b) => b.createdAt - a.createdAt)
      messages.value = list
    })
  })
})
</script>
