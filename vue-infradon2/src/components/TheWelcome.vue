<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)



interface Post {
  _id: string
  _rev: string
  _conflicts?: any
  type?: 'post'
  title: string
  content: string
  likes: number
  attributes?: {
    creation_date: any
  }
}

interface Plat {
  _id: string
  _rev: string
  _conflicts?: any
  type: 'plat'
  nom: string
  prix: number
  ingredients: string
  client: string
  likes: number
}



const batchSize = ref(10)
const storage = ref<PouchDB.Database | null>(null)
const sync = ref<PouchDB.Replication.Replication<Post> | null>(null)
const isOnline = ref(false)

const postsData = ref<Post[]>([])
const platsData = ref<Plat[]>([])



const words = ['Léa', 'Inoé', 'Camilo', 'Teicir', 'Yannis', 'Elia', 'Loann', 'Sasita', 'Sarah']

const clientsPotentiels = [
  'Léa', 'Inoé', 'Camilo', 'Teicir', 'Yannis', 'Elia', 'Loann', 'Sasita', 
  'Sarah', 'Enya', 'Gabriel', 'Nuno', 'Tanguy', 'Loic', 'Dylan', 'Liliana', 'Thierry'
]

const nomsPlats = [
  'Pizza Margherita', 'Burger Royal', 'Sushi Mix', 'Salade César', 
  'Pâtes Carbo', 'Tacos XL', 'Curry Vert', 'Risotto Champignons', 'Poke Bowl'
]

const ingredientsList = [
  'Tomate', 'Mozzarella', 'Basilic', 'Poulet', 'Curry', 
  'Riz', 'Saumon', 'Avocat', 'Piment', 'Fromage', 'Bacon'
]



onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = async () => {
  console.log('=> Connexion à la base de données')
  const localdb = new PouchDB('collection_infradon2')

  if (localdb) {
    storage.value = localdb
    console.log('Connected to collection : ' + localdb.name)

    try {
      await storage.value.createIndex({
        index: { fields: ['type'] },
      })
    } catch (error) { console.error("Erreur index", error) }

    try {
      await localdb.replicate.from('http://ykem:Salutpoilu99$@localhost:5984/infradon2')
      console.log('Réplication initiale terminée.')
    } catch (error) {
      console.warn('Erreur réplication initiale (CouchDB offline ?)', (error as Error).message)
    } finally {
      syncData()
      fetchData()
    }
  }
}



const syncData = () => {
  if (storage.value) {
    isOnline.value = true
    if (sync.value) sync.value.cancel()

    sync.value = storage.value
      .sync('http://ykem:Salutpoilu99$@localhost:5984/infradon2', { live: true, retry: true })
      .on('change', fetchData)
      .on('error', () => (isOnline.value = false))
      .on('paused', () => (isOnline.value = true))
      .on('active', () => (isOnline.value = true))
  }
}

const toggle = () => {
  if (sync.value) {
    sync.value.cancel()
    sync.value = null
    isOnline.value = false
  } else {
    syncData()
  }
}



const fetchData = async () => {
  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    const allDocs = result.rows
      .map((row: any) => row.doc)
      .filter((doc: any) => !doc._id.startsWith('_'))

   
    
    platsData.value = (allDocs.filter((doc: any) => doc.type === 'plat') as Plat[])
      .sort((a, b) => (b.likes || 0) - (a.likes || 0))

    postsData.value = (allDocs.filter((doc: any) => doc.type === 'post' || !doc.type) as Post[])
      .sort((a, b) => (b.likes || 0) - (a.likes || 0))

  } catch (error) {
    console.error('Erreur fetch :', error)
  }
}



const search = async (event: Event) => {
  const input = event.target as HTMLInputElement
  const query = input.value.toLowerCase().trim()
  input.blur()

  if (query === '') {
    fetchData()
    return
  }

  if (!storage.value) return

  try {
    const result = await storage.value.allDocs({
      include_docs: true,
      conflicts: true,
    })

    const allDocs = result.rows
      .map((row: any) => row.doc)
      .filter((doc: any) => !doc._id.startsWith('_'))

    
    postsData.value = (allDocs.filter((doc: any) => doc.type === 'post' || !doc.type) as Post[])
      .filter(post => 
        (post.title && post.title.toLowerCase().includes(query)) || 
        (post.content && post.content.toLowerCase().includes(query))
      )
      .sort((a, b) => (b.likes || 0) - (a.likes || 0))

    
    platsData.value = (allDocs.filter((doc: any) => doc.type === 'plat') as Plat[])
      .filter(plat => 
        (plat.nom && plat.nom.toLowerCase().includes(query)) || 
        (plat.client && plat.client.toLowerCase().includes(query)) ||
        (plat.ingredients && plat.ingredients.toLowerCase().includes(query))
      )
      .sort((a, b) => (b.likes || 0) - (a.likes || 0))

  } catch (error) {
    console.error('Erreur recherche', error)
  }
}

const searchReset = () => {
  const el = document.querySelector('.search') as HTMLInputElement
  if(el) el.value = ''
  fetchData()
}



const createPlat = async () => {
  if (!storage.value) return
  
  const nomPlat = nomsPlats[Math.floor(Math.random() * nomsPlats.length)]
  const clientRandom = clientsPotentiels[Math.floor(Math.random() * clientsPotentiels.length)]
  const ingr = ingredientsList[Math.floor(Math.random() * ingredientsList.length)]

  try {
    await storage.value.post({
      type: 'plat',
      nom: nomPlat,
      client: clientRandom,
      ingredients: ingr,
      prix: Math.floor(Math.random() * 20) + 10,
      likes: 0
    })
    fetchData()
  } catch (err) { console.log(err) }
}

const createMultiplePlats = async () => {
  if (!storage.value) return
  const numToCreate = parseInt(batchSize.value.toString()) || 0
  
  if (numToCreate > 0) {
    const plats = []
    for (let i = 0; i < numToCreate; i++) {
      plats.push({
        type: 'plat',
        nom: nomsPlats[Math.floor(Math.random() * nomsPlats.length)],
        client: clientsPotentiels[Math.floor(Math.random() * clientsPotentiels.length)],
        ingredients: ingredientsList[Math.floor(Math.random() * ingredientsList.length)],
        prix: Math.floor(Math.random() * 20) + 10,
        likes: 0
      })
    }
    try {
      await storage.value.bulkDocs(plats)
      fetchData()
    } catch (err) { console.log(err) }
  }
}

const updatePlat = async (plat: Plat) => {
  if (!storage.value) return

  const nouveauNom = nomsPlats[Math.floor(Math.random() * nomsPlats.length)]
  const nouveauClient = clientsPotentiels[Math.floor(Math.random() * clientsPotentiels.length)]
  const nouvelIngr = ingredientsList[Math.floor(Math.random() * ingredientsList.length)]
  const nouveauPrix = Math.floor(Math.random() * 30) + 5

  try {
    await storage.value.put({
      ...plat,
      nom: nouveauNom,
      client: nouveauClient,
      ingredients: nouvelIngr,
      prix: nouveauPrix,
      _rev: plat._rev,
    })
    fetchData()
  } catch (err) { fetchData() }
}

const addLikePlat = async (plat: Plat) => {
  if (!storage.value) return
  try {
    await storage.value.put({
      ...plat,
      likes: (plat.likes || 0) + 1,
      _rev: plat._rev,
    })
    fetchData()
  } catch (err) { fetchData() }
}



let counter = 0

const createDoc = async () => {
  if (!storage.value) return
  counter++
  try {
    await storage.value.post({
      type: 'post',
      title: 'Document ' + counter,
      content: words[Math.floor(Math.random() * words.length)],
      likes: 0,
      attributes: { creation_date: new Date().toISOString() },
    })
    fetchData()
  } catch (err) { console.log(err) }
}

const createMultipleDocs = async () => {
    if (!storage.value) return
    const numToCreate = parseInt(batchSize.value.toString()) || 0
    const docs = []
    for (let i = 0; i < numToCreate; i++) {
        counter++
        docs.push({
            type: 'post',
            title: 'Doc Batch ' + counter,
            content: 'Contenu batch',
            likes: 0
        })
    }
    await storage.value.bulkDocs(docs)
    fetchData()
}

const updateDoc = async (post: Post) => {
  if (!storage.value) return
  try {
    await storage.value.put({
      ...post,
      content: words[Math.floor(Math.random() * words.length)] + ' (modifié)',
      _rev: post._rev,
    })
    fetchData()
  } catch (err) { fetchData() }
}

const addLike = async (post: Post) => {
  if (!storage.value) return
  try {
    await storage.value.put({
      ...post,
      likes: (post.likes || 0) + 1,
      _rev: post._rev,
    })
    fetchData()
  } catch (err) { fetchData() }
}

const deleteEntity = async (doc: Post | Plat) => {
  if (!storage.value) return
  try {
    await storage.value.remove(doc as any)
    fetchData()
  } catch (err) { console.log(err) }
}
</script>

<template>
  <div id="app-container">
    <h1>Gestion PouchDB</h1>

    <div class="header-actions">
      <div class="status-toggle">
        <label class="switch">
          <input type="checkbox" :checked="isOnline" @click="toggle" />
          <span class="slider round"></span>
        </label>
        <label :class="isOnline ? 'status-online' : 'status-offline'">
          {{ isOnline ? 'Online' : 'Offline' }}
        </label>
      </div>

      <div class="batch-actions">
        <input type="number" v-model="batchSize" min="1" class="input-batch" />
        <button @click="createMultipleDocs" class="btn-secondary">Batch Docs</button>
        <button @click="createMultiplePlats" class="btn-batch-plat">Batch Plats 🍲</button>
      </div>
      
      <div class="single-actions">
        <button @click="createDoc" class="btn-primary">Ajouter Document</button>
        <button @click="createPlat" class="btn-plat">Ajouter Plat 🍲</button>
      </div>
    </div>

    <div class="search-container">
      <input type="text" placeholder="Chercher (Plat, Client, Ingrédient, Document...)" @keyup.enter="search" class="search" />
      <button @click="searchReset" class="btn-reset">X</button>
    </div>

    <div class="lists-container">
        
        <div class="column">
            <h3>📑 Documents ({{ postsData.length }})</h3>
            <div class="document-list">
              <article v-for="post in postsData" :key="post._id" class="card-doc">
                <div class="article-content">
                    <h4>{{ post.title }} <span class="conflicts" v-if="post._conflicts">!</span></h4>
                    <p>{{ post.content }}</p>
                </div>
                <div class="article-actions">
                    <button @click="addLike(post)" class="btn-like">❤️ {{ post.likes || 0 }}</button>
                    <button @click="updateDoc(post)" class="btn-update">📝</button>
                    <button @click="deleteEntity(post)" class="btn-delete">X</button>
                </div>
              </article>
              <p v-if="postsData.length === 0" class="empty">Aucun document.</p>
            </div>
        </div>

        <div class="column">
            <h3>🍲 Commandes ({{ platsData.length }})</h3>
            <div class="document-list">
              <article v-for="plat in platsData" :key="plat._id" class="card-plat">
                <div class="article-content">
                    <h4>{{ plat.nom }} <small>pour {{ plat.client || '?' }}</small></h4>
                    <p>Ingrédient: <strong>{{ plat.ingredients }}</strong></p>
                    <p class="price">{{ plat.prix }} €</p>
                </div>
                <div class="article-actions">
                    <button @click="addLikePlat(plat)" class="btn-like">❤️ {{ plat.likes || 0 }}</button>
                    <button @click="updatePlat(plat)" class="btn-update-plat">🎲 Change</button>
                    <button @click="deleteEntity(plat)" class="btn-delete">X</button>
                </div>
              </article>
              <p v-if="platsData.length === 0" class="empty">Aucun plat.</p>
            </div>
        </div>

    </div>
  </div>
</template>

<style scoped>
#app-container {
  max-width: 1000px;
  margin: 0 auto;
  background-color: #fff;
  padding: 2rem;
  border-radius: 1rem;
  font-family: 'Inter', sans-serif;
  box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1);
}

h1 { margin-bottom: 1.5rem; color: #1f2937; }
h3 { color: #4b5563; border-bottom: 2px solid #e5e7eb; padding-bottom: 0.5rem; }
h4 small { font-weight: normal; font-size: 0.8em; color: #6b7280; margin-left: 5px; font-style: italic; }

.lists-container {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
}

@media (max-width: 768px) {
    .lists-container { grid-template-columns: 1fr; }
}

.header-actions { display: flex; gap: 1rem; flex-wrap: wrap; align-items: center; margin-bottom: 1.5rem; }
.batch-actions { display: flex; gap: 0.5rem; }
.single-actions { display: flex; gap: 0.5rem; margin-left: auto; }

.input-batch { width: 60px; padding: 0.5rem; border: 1px solid #ccc; border-radius: 0.5rem; }

button { cursor: pointer; border: none; border-radius: 0.5rem; font-weight: 600; padding: 0.5rem 1rem; transition: 0.2s; }
.btn-primary { background: #2563eb; color: white; }
.btn-secondary { background: #6366f1; color: white; }
.btn-plat { background: #ea580c; color: white; }
.btn-plat:hover { background: #c2410c; }
.btn-batch-plat { background: #d97706; color: white; }
.btn-batch-plat:hover { background: #b45309; }

.btn-delete { background: #ef4444; color: white; padding: 0.5rem; }
.btn-update { background: #f59e0b; color: white; padding: 0.5rem; }
.btn-like { background: #ec4899; color: white; padding: 0.5rem;}
.btn-update-plat { background: #10b981; color: white; padding: 0.5rem; } 

.card-doc {
    background: #f3f4f6;
    border-left: 5px solid #2563eb;
    padding: 1rem;
    margin-bottom: 1rem;
    border-radius: 0.5rem;
    display: flex; justify-content: space-between; align-items: center;
}

.card-plat {
    background: #fff7ed;
    border-left: 5px solid #ea580c;
    padding: 1rem;
    margin-bottom: 1rem;
    border-radius: 0.5rem;
    display: flex; justify-content: space-between; align-items: center;
    border: 1px solid #fed7aa;
}

.price { font-weight: bold; color: #15803d; font-size: 1.1rem; }
.article-actions { display: flex; gap: 0.5rem; }
.empty { font-style: italic; color: #9ca3af; text-align: center; margin-top: 1rem; }

.switch { position: relative; display: inline-block; width: 44px; height: 24px; }
.switch input { opacity: 0; width: 0; height: 0; }
.slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #ccc; transition: .4s; border-radius: 34px; }
.slider:before { position: absolute; content: ""; height: 16px; width: 16px; left: 4px; bottom: 4px; background-color: white; transition: .4s; border-radius: 50%; }
input:checked + .slider { background-color: #10b981; }
input:checked + .slider:before { transform: translateX(20px); }
.status-online { color: #059669; font-weight: bold; margin-left: 0.5rem;}
.status-offline { color: #dc2626; font-weight: bold; margin-left: 0.5rem;}

.search-container { display: flex; gap: 0.5rem; margin-bottom: 1rem; }
.search { flex-grow: 1; padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 0.5rem; }
</style>