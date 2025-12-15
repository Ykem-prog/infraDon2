<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)


interface Post {
  _id: string
  _rev: string
  type: 'post'
  title: string
  content: string
  likes: number
  isModified?: boolean
  attributes?: { creation_date: any }
}

interface Comment {
  _id: string
  _rev?: string
  type: 'comment'
  plat_id: string
  text: string
  author: string
  date: string
}

interface Plat {
  _id: string
  _rev: string
  type: 'plat'
  nom: string
  prix: number
  ingredients: string
  client: string
  likes: number
  isModified?: boolean
  showComments?: boolean
  loadedComments?: Comment[]
  newCommentText?: string
}


const batchSize = ref(10)
const storage = ref<PouchDB.Database | null>(null)
const sync = ref<PouchDB.Replication.Replication<any> | null>(null)
const isOnline = ref(false)

const postsData = ref<Post[]>([])
const platsData = ref<Plat[]>([])


const words = ['Urgent', 'Brouillon', 'Validé', 'A revoir', 'Note', 'Facture', 'Devis']
const clients = ['Léa', 'Inoé', 'Camilo', 'Teicir', 'Yannis', 'Sarah', 'Pierre']
const platsNames = ['Pizza', 'Burger', 'Sushi', 'Pâtes', 'Tacos', 'Curry']

onMounted(() => {
  initDatabase()
})


const initDatabase = async () => {
  const localdb = new PouchDB('collection_infradon2')

  if (localdb) {
    storage.value = localdb

    try {

      await storage.value.createIndex({
        index: {
            fields: ['type', 'likes'],
            name: 'idx_type_likes_v2',
            ddoc: 'idx_type_likes_v2_ddoc'
        }
      })


      await storage.value.createIndex({
        index: {
            fields: ['type', 'plat_id'],
            name: 'idx_comments',
            ddoc: 'idx_comments_ddoc'
        }
      })
      console.log('Index créés avec succès')
    } catch (error) { console.error("Erreur index", error) }

    try {
      const remoteDB = 'http://ykem:Salutpoilu99%24@localhost:5984/infradon2'
      await localdb.replicate.from(remoteDB)
      console.log('Réplication terminée.')
    } catch (error) { console.warn('Erreur réplication', error) }
    finally {
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
      .sync('http://ykem:Salutpoilu99%24@localhost:5984/infradon2', { live: true, retry: true })
      .on('change', () => fetchData())
      .on('error', () => (isOnline.value = false))
      .on('paused', () => (isOnline.value = true))
      .on('active', () => (isOnline.value = true))
  }
}

const toggleSync = () => {
  if (sync.value) {
    sync.value.cancel(); sync.value = null; isOnline.value = false
  } else { syncData() }
}


const fetchData = async () => {
  if (!storage.value) return
  try {

    const resPlats = await storage.value.find({
      selector: { type: 'plat', likes: { $gte: null } },
      sort: [{ 'type': 'desc' }, { 'likes': 'desc' }]
    })

    const oldPlats = platsData.value
    platsData.value = (resPlats.docs as Plat[]).map(newP => {
        const old = oldPlats.find(p => p._id === newP._id)
        return {
            ...newP,
            showComments: old ? old.showComments : false,
            loadedComments: old ? old.loadedComments : [],
            newCommentText: ''
        }
    })


    const resPosts = await storage.value.find({
      selector: { type: 'post', likes: { $gte: null } },
      sort: [{ 'type': 'desc' }, { 'likes': 'desc' }]
    })
    postsData.value = resPosts.docs as Post[]

  } catch (e) { console.error("Erreur Fetch:", e) }
}



const updateDoc = async (post: Post) => {
    if (!storage.value) return
    const newContent = prompt("Modifier le contenu du document :", post.content)

    if (newContent !== null && newContent !== post.content) {
        try {
            await storage.value.put({
                ...post,
                content: newContent,
                isModified: true
            })
            fetchData()
        } catch (err) { if ((err as any).name === 'conflict') fetchData() }
    }
}

const updatePlat = async (plat: Plat) => {
    if (!storage.value) return
    const newName = prompt("Modifier le nom du plat :", plat.nom)
    if (newName === null) return
    const newPriceStr = prompt("Modifier le prix (€) :", plat.prix.toString())
    if (newPriceStr === null) return
    const newPrice = parseFloat(newPriceStr)

    try {
        await storage.value.put({
            ...plat,
            nom: newName,
            prix: isNaN(newPrice) ? plat.prix : newPrice,
            isModified: true
        })
        fetchData()
    } catch (err) { if ((err as any).name === 'conflict') fetchData() }
}



const loadComments = async (plat: Plat) => {
    if (!storage.value) return
    plat.showComments = !plat.showComments

    if (plat.showComments) {
        try {
            const result = await storage.value.find({
                selector: { type: 'comment', plat_id: plat._id }
            })
            plat.loadedComments = result.docs as Comment[]
        } catch (err) { console.error(err) }
    }
}

const addComment = async (plat: Plat) => {
    if (!storage.value || !plat.newCommentText) return
    const comment: Comment = {
        _id: new Date().toISOString() + Math.random(),
        type: 'comment',
        plat_id: plat._id,
        text: plat.newCommentText,
        author: clients[Math.floor(Math.random() * clients.length)],
        date: new Date().toISOString()
    }
    try {
        await storage.value.put(comment)
        plat.newCommentText = ''
        plat.showComments = false
        loadComments(plat)
    } catch (err) { console.error(err) }
}


const search = async (event: Event) => {
  const input = event.target as HTMLInputElement
  const query = input.value.toLowerCase().trim()
  if (query === '') { fetchData(); return }
  if (!storage.value) return


  const resPosts = await storage.value.find({
      selector: {
        type: 'post',
        $or: [
            { title: { $regex: RegExp(query, "i") } },
            { content: { $regex: RegExp(query, "i") } }
        ]
      }
  })
  postsData.value = resPosts.docs as Post[]


  const resPlats = await storage.value.find({
      selector: {
        type: 'plat',
        nom: { $regex: RegExp(query, "i") }
      }
  })
  platsData.value = resPlats.docs as Plat[]
}

const searchReset = () => {
    const el = document.querySelector('.search') as HTMLInputElement; if(el) el.value = ''; fetchData()
}


const createPlat = async () => {
  if (!storage.value) return
  await storage.value.post({
    type: 'plat',
    nom: platsNames[Math.floor(Math.random() * platsNames.length)],
    client: clients[Math.floor(Math.random() * clients.length)],
    ingredients: 'Tomate, Fromage',
    prix: 15,
    likes: 0
  })
  fetchData()
}

const createMultiplePlats = async () => {
  if (!storage.value) return
  const num = parseInt(batchSize.value.toString()) || 0
  const plats = []
  for (let i = 0; i < num; i++) {
    plats.push({
        type: 'plat',
        nom: platsNames[Math.floor(Math.random() * platsNames.length)],
        client: clients[Math.floor(Math.random() * clients.length)],
        ingredients: 'Surprise',
        prix: Math.floor(Math.random() * 20) + 10,
        likes: 0
    })
  }
  await storage.value.bulkDocs(plats)
  fetchData()
}


const createDoc = async () => {
  if (!storage.value) return
  let counter = Math.floor(Math.random() * 1000)
  await storage.value.post({
    type: 'post',
    title: 'Doc ' + counter,
    content: words[Math.floor(Math.random() * words.length)],
    likes: 0,
    attributes: { creation_date: new Date().toISOString() }
  })
  fetchData()
}

const createMultipleDocs = async () => {
    if (!storage.value) return
    const num = parseInt(batchSize.value.toString()) || 0
    const docs = []
    for (let i = 0; i < num; i++) {
        docs.push({
            type: 'post',
            title: 'Doc Batch ' + i,
            content: words[Math.floor(Math.random() * words.length)],
            likes: 0,
            attributes: { creation_date: new Date().toISOString() }
        })
    }
    await storage.value.bulkDocs(docs)
    fetchData()
}


const deleteEntity = async (doc: any) => {
  if (!storage.value) return


  if (doc.type === 'plat') {
      try {

          const result = await storage.value.find({
              selector: { type: 'comment', plat_id: doc._id }
          })


          if (result.docs.length > 0) {
              const toDelete = result.docs.map(c => ({ ...c, _deleted: true }))
              await storage.value.bulkDocs(toDelete)
              console.log(`Cascade : ${result.docs.length} commentaires supprimés.`)
          }
      } catch (e) { console.error("Erreur nettoyage commentaires", e) }
  }


  try {
      await storage.value.remove(doc)
      fetchData()
  } catch (err) { console.error("Erreur suppression doc", err) }
}
const addLike = async (doc: any) => {
    if (!storage.value) return;
    try { await storage.value.put({...doc, likes: (doc.likes||0)+1}); fetchData() } catch(e){}
}
</script>

<template>
  <div id="app-container">
    <div class="top-bar">
        <h1>Infradon : Gestion & Commentaires</h1>

    </div>

    <div class="header-actions">
      <div class="status-toggle">
        <label class="switch">
          <input type="checkbox" :checked="isOnline" @click="toggleSync" />
          <span class="slider round"></span>
        </label>
        <span :class="isOnline ? 'online' : 'offline'">{{ isOnline ? 'Online' : 'Offline' }}</span>
      </div>

      <div class="batch-actions">
        <input type="number" v-model="batchSize" min="1" class="input-batch" />
        <button @click="createMultipleDocs" class="btn-secondary">Batch Docs</button>
        <button @click="createMultiplePlats" class="btn-batch-plat">Batch Plats</button>
      </div>

      <div class="single-actions">
        <button @click="createDoc" class="btn-primary">+ Doc</button>
        <button @click="createPlat" class="btn-plat">+ Plat</button>
      </div>
    </div>

    <div class="search-container">
      <input type="text" placeholder="Rechercher..." @keyup.enter="search" class="search" />
      <button @click="searchReset" class="btn-reset">X</button>
    </div>

    <div class="lists-container">
        <div class="column">
            <h3> Documents (Top Likes)</h3>
            <div class="document-list">
              <article v-for="post in postsData" :key="post._id" class="card-doc">
                <div class="article-content">
                    <h4>
                        {{ post.title }}
                        <span v-if="post.isModified" class="modified-tag">(modifié)</span>
                    </h4>
                    <p>{{ post.content }}</p>
                </div>
                <div class="article-actions">
                    <button @click="addLike(post)" class="btn-like">❤️ {{ post.likes }}</button>
                    <button @click="updateDoc(post)" class="btn-update">📝</button>
                    <button @click="deleteEntity(post)" class="btn-delete">X</button>
                </div>
              </article>
              <p v-if="postsData.length === 0" class="empty">Aucun document.</p>
            </div>
        </div>

        <div class="column">
            <h3> Commandes (Top Likes)</h3>
            <div class="document-list">
              <article v-for="plat in platsData" :key="plat._id" class="card-plat">
                <div class="plat-header">
                    <div>
                        <h4>
                            {{ plat.nom }} <small>({{ plat.client }})</small>
                            <span v-if="plat.isModified" class="modified-tag">(modifié)</span>
                        </h4>
                        <p class="price">{{ plat.prix }} €</p>
                    </div>
                    <div class="article-actions">
                        <button @click="addLike(plat)" class="btn-like">❤️ {{ plat.likes }}</button>
                        <button @click="updatePlat(plat)" class="btn-update">📝</button>
                        <button @click="deleteEntity(plat)" class="btn-delete">X</button>
                    </div>
                </div>

                <button @click="loadComments(plat)" class="btn-comments">
                    💬 {{ plat.showComments ? 'Masquer' : 'Avis' }}
                </button>

                <div v-if="plat.showComments" class="comments-section">
                    <div class="comments-list">
                        <div v-if="!plat.loadedComments || plat.loadedComments.length === 0" class="no-com">Pas encore d'avis.</div>
                        <div v-for="com in plat.loadedComments" :key="com._id" class="comment-bubble">
                            <strong>{{ com.author }}:</strong> {{ com.text }}
                        </div>
                    </div>

                    <div class="add-comment">
                        <input type="text" v-model="plat.newCommentText" placeholder="Votre avis..." @keyup.enter="addComment(plat)">
                        <button @click="addComment(plat)">Envoyer</button>
                    </div>
                </div>

              </article>
              <p v-if="platsData.length === 0" class="empty">Aucun plat.</p>
            </div>
        </div>

    </div>
  </div>
</template>

<style scoped>
#app-container { max-width: 1000px; margin: 0 auto; background-color: #fff; padding: 2rem; border-radius: 1rem; font-family: 'Inter', sans-serif; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
.top-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; }
h1 { margin: 0; color: #1f2937; }
.btn-danger-outline { border: 1px solid #e74c3c; color: #e74c3c; background: transparent; padding: 5px 10px; border-radius: 4px; font-size: 0.8rem; cursor: pointer; }
.btn-danger-outline:hover { background: #e74c3c; color: white; }

.lists-container { display: grid; grid-template-columns: 1fr 1fr; gap: 2rem; }
@media (max-width: 768px) { .lists-container { grid-template-columns: 1fr; } }

.header-actions { display: flex; gap: 1rem; flex-wrap: wrap; margin-bottom: 1rem; }
.input-batch { width: 50px; padding: 0.5rem; border: 1px solid #ccc; border-radius: 5px; }
button { cursor: pointer; border: none; border-radius: 5px; font-weight: 600; padding: 0.5rem 1rem; transition: 0.2s; }
.btn-primary { background: #2563eb; color: white; }
.btn-secondary { background: #6366f1; color: white; }
.btn-plat { background: #ea580c; color: white; }
.btn-batch-plat { background: #d97706; color: white; }
.btn-delete { background: #ef4444; color: white; padding: 0.3rem 0.6rem; }
.btn-like { background: #ec4899; color: white; padding: 0.3rem 0.6rem;}
.btn-update { background: #f59e0b; color: white; padding: 0.3rem 0.6rem;}
.btn-comments { background: #0ea5e9; color: white; width: 100%; margin-top: 10px; padding: 5px;}

.modified-tag { font-size: 0.7em; color: #888; font-style: italic; margin-left: 5px; font-weight: normal; }

.card-doc { background: #f3f4f6; border-left: 5px solid #2563eb; padding: 1rem; margin-bottom: 1rem; border-radius: 0.5rem; display: flex; justify-content: space-between; align-items: center; }
.card-plat { background: #fff7ed; border-left: 5px solid #ea580c; padding: 1rem; margin-bottom: 1rem; border-radius: 0.5rem; border: 1px solid #fed7aa; }
.plat-header { display: flex; justify-content: space-between; align-items: start; }

.comments-section { margin-top: 10px; background: white; padding: 10px; border-radius: 5px; border: 1px solid #ddd; }
.comment-bubble { font-size: 0.9em; border-bottom: 1px solid #eee; padding: 4px 0; }
.add-comment { display: flex; margin-top: 10px; gap: 5px; }
.add-comment input { flex: 1; padding: 5px; border: 1px solid #ccc; border-radius: 4px; }
.add-comment button { background: #10b981; color: white; padding: 5px 10px; }
.no-com { font-style: italic; color: #999; font-size: 0.8em; }
.empty { font-style: italic; color: #aaa; text-align: center; }

.price { font-weight: bold; color: #15803d; }
.switch { position: relative; display: inline-block; width: 44px; height: 24px; }
.switch input { opacity: 0; width: 0; height: 0; }
.slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #ccc; transition: .4s; border-radius: 34px; }
input:checked + .slider { background-color: #10b981; }
input:checked + .slider:before { transform: translateX(20px); height: 16px; width: 16px; left: 4px; bottom: 4px; content: ""; background-color: white; border-radius: 50%; position: absolute; }
.status-online { color: #059669; font-weight: bold; margin-left: 0.5rem; }
.status-offline { color: #dc2626; font-weight: bold; margin-left: 0.5rem; }
.search-container { display: flex; gap: 0.5rem; margin-bottom: 1rem; }
.search { flex-grow: 1; padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 0.5rem; }
</style>
