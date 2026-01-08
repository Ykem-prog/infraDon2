<script setup lang="ts">
/**
 * PROJET INFRADON 2 - VueJS & PouchDB/CouchDB
 * * JUSTIFICATION DES CHOIX TECHNIQUES :
 * 1. Architecture : "Offline First" grâce à PouchDB et la réplication Live.
 * 2. Performance : Abandon de `allDocs` au profit de `find` (Mango Queries).
 * Cela permet de déléguer le tri et le filtrage au moteur de base de données (Serveur)
 * plutôt que de surcharger le thread JavaScript du client.
 * 3. Optimisation : Pagination (limit/skip) pour ne pas charger toute la base.
 * Chargement paresseux (Lazy Loading) des commentaires complets.
 * 4. Relations : Utilisation d'une "Clé Étrangère" (plat_id) dans les commentaires
 * plutôt que d'imbriquer des tableaux, pour éviter les conflits de mise à jour.
 */

import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

// Activation du plugin Mango pour les requêtes riches (find, selector, sort)
PouchDB.plugin(PouchDBFind)

// --- INTERFACES ---
interface Post {
  _id: string
  _rev: string
  type: 'post'
  title: string
  content: string
  likes: number
  isModified?: boolean
  _attachments?: Record<string, { content_type: string, length: number }>
  imageURL?: string // URL temporaire pour l'affichage (non persisté)
  attributes?: { creation_date: any }
}

interface Comment {
  _id: string
  _rev?: string
  type: 'comment'
  plat_id: string // Lien relationnel vers le Plat parent
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
  _attachments?: Record<string, { content_type: string, length: number }>
  imageURL?: string
  // Champs UI (non persistés) pour l'UX
  showComments?: boolean
  loadedComments?: Comment[]
  lastComment?: Comment
  newCommentText?: string
}

// --- VARIABLES D'ÉTAT ---
const batchSize = ref(10)
const storage = ref<PouchDB.Database | null>(null)
const sync = ref<PouchDB.Replication.Replication<any> | null>(null)
const isOnline = ref(false)

// Gestion de la Pagination (Optimisation mémoire)
const pageSize = 10
const currentPagePlats = ref(0)
const currentPageDocs = ref(0)
const hasMorePlats = ref(true)
const hasMoreDocs = ref(true)

// Upload
const fileInput = ref<HTMLInputElement | null>(null)
const itemToUpload = ref<Post | Plat | null>(null)

const postsData = ref<Post[]>([])
const platsData = ref<Plat[]>([])

// Données "Mock" pour la génération
const words = ['Urgent', 'Brouillon', 'Validé', 'A revoir', 'Note', 'Facture', 'Devis']
const clients = ['Léa', 'Inoé', 'Camilo', 'Teicir', 'Yannis', 'Sarah', 'Pierre']
const platsNames = ['Pizza', 'Burger', 'Sushi', 'Pâtes', 'Tacos', 'Curry']

onMounted(() => {
  initDatabase()
})

// --- INITIALISATION DATABASE ---
const initDatabase = async () => {
  // Création de la base locale (Navigateur - IndexedDB)
  const localdb = new PouchDB('collection_infradon2')

  if (localdb) {
    storage.value = localdb

    try {
      /**
       * CRÉATION DES INDEX (Indispensable pour Mango Queries)
       * CouchDB a besoin de ces index pour exécuter les tris côté serveur.
       * Sans cela, PouchDB rejetterait les requêtes avec 'sort'.
       */
      await storage.value.createIndex({
        index: { fields: ['type', 'likes'], name: 'idx_type_likes_v2', ddoc: 'idx_type_likes_v2_ddoc' }
      })
      await storage.value.createIndex({
        index: { fields: ['type', 'plat_id'], name: 'idx_comments', ddoc: 'idx_comments_ddoc' }
      })
      console.log('Index créés avec succès')
    } catch (error) { console.error("Erreur index", error) }

    try {
      // Configuration de la réplication
      // Utilisation de l'authentification dans l'URL (attention en prod, à éviter)
      const remoteDB = 'http://ykem:Salutpoilu99%24@localhost:5984/infradon2'
      await localdb.replicate.from(remoteDB)
      console.log('Réplication initiale terminée.')
    } catch (error) { console.warn('Erreur réplication', error) }
    finally {
      syncData()
      // Chargement initial des premières pages
      loadMorePlats(true)
      loadMoreDocs(true)
    }
  }
}

// --- RESET DB ---
// Utile en développement si les Design Documents (Index) sont corrompus ou obsolètes
const resetDatabase = async () => {
    if (!storage.value) return;
    if(!confirm("Attention : Suppression complète de la base locale pour régénérer les index. Continuer ?")) return;
    if (sync.value) sync.value.cancel();
    await storage.value.destroy();
    location.reload();
};

/**
 * SYNCHRONISATION
 * Utilisation du mode { live: true, retry: true } pour garantir la résilience
 * face aux coupures réseau (Offline First).
 */
const syncData = () => {
  if (storage.value) {
    isOnline.value = true
    if (sync.value) sync.value.cancel()

    sync.value = storage.value
      .sync('http://ykem:Salutpoilu99%24@localhost:5984/infradon2', { live: true, retry: true })
      .on('change', () => {
          // Recharger les données si changement distant détecté
          loadMorePlats(true)
          loadMoreDocs(true)
      })
      .on('error', () => (isOnline.value = false))
      .on('paused', () => (isOnline.value = true))
      .on('active', () => (isOnline.value = true))
  }
}
const toggleSync = () => {
  if (sync.value) { sync.value.cancel(); sync.value = null; isOnline.value = false }
  else { syncData() }
}

// --- GESTION DES ASSETS (Page 7) ---
// Utilisation des '_attachments' de CouchDB pour stocker les binaires directement dans le document.

const triggerUpload = (item: Post | Plat) => {
    itemToUpload.value = item
    if (fileInput.value) fileInput.value.click()
}

const processUpload = async (event: Event) => {
    const input = event.target as HTMLInputElement
    if (!input.files || input.files.length === 0 || !storage.value || !itemToUpload.value) return

    const file = input.files[0]
    const item = itemToUpload.value

    try {
        await storage.value.putAttachment(
            item._id,
            file.name,
            item._rev,
            file,
            file.type
        )
        // Reset UI et rechargement pour mettre à jour le _rev
        input.value = ''
        itemToUpload.value = null
        if (item.type === 'plat') loadMorePlats(true)
        else loadMoreDocs(true)

    } catch (err) { console.error("Erreur upload", err) }
}

// Conversion Blob -> ObjectURL pour affichage dans le DOM
const loadImage = async (item: Post | Plat) => {
    if (!storage.value || !item._attachments) return
    const attachmentName = Object.keys(item._attachments)[0]
    if (!attachmentName) return

    try {
        const blob = await storage.value.getAttachment(item._id, attachmentName) as Blob
        item.imageURL = URL.createObjectURL(blob)
    } catch (err) { console.error("Erreur chargement image", err) }
}

const deleteImage = async (item: Post | Plat) => {
    if (!storage.value || !item._attachments) return
    if (!confirm("Supprimer l'image ?")) return

    const attachmentName = Object.keys(item._attachments)[0]
    try {
        await storage.value.removeAttachment(item._id, attachmentName, item._rev)
        item.imageURL = undefined
        if (item.type === 'plat') loadMorePlats(true)
        else loadMoreDocs(true)
    } catch (err) { console.error(err) }
}


/**
 * LOGIQUE DE PAGINATION & TRI (Page 8 & 9)
 * * Choix technique : Utilisation de .find() avec 'limit' et 'skip'.
 * Contrairement à un filtre JavaScript post-chargement, cette approche
 * économise la bande passante et la mémoire client en ne récupérant que
 * ce qui est nécessaire.
 */

const loadMorePlats = async (reset = false) => {
    if (!storage.value) return
    if (reset) { currentPagePlats.value = 0; platsData.value = []; hasMorePlats.value = true }

    try {
        const result = await storage.value.find({
            selector: { type: 'plat', likes: { $gte: null } }, // Filtre sur le type
            sort: [{ 'type': 'desc' }, { 'likes': 'desc' }], // Tri Serveur par Likes
            limit: pageSize,
            skip: currentPagePlats.value * pageSize
        })

        if (result.docs.length < pageSize) hasMorePlats.value = false
        const newPlats = result.docs as Plat[]

        // Hydratation des données complémentaires (Images + Dernier avis)
        for (let plat of newPlats) {
            const lastCom = await fetchLastComment(plat._id)
            plat.lastComment = lastCom || undefined

            if (plat._attachments) loadImage(plat)

            plat.showComments = false
            plat.loadedComments = []
        }
        if (reset) platsData.value = newPlats
        else platsData.value = [...platsData.value, ...newPlats]
        currentPagePlats.value++
    } catch (e) { }
}

const loadMoreDocs = async (reset = false) => {
    if (!storage.value) return
    if (reset) { currentPageDocs.value = 0; postsData.value = []; hasMoreDocs.value = true }

    try {
        const result = await storage.value.find({
            selector: { type: 'post', likes: { $gte: null } },
            sort: [{ 'type': 'desc' }, { 'likes': 'desc' }],
            limit: pageSize,
            skip: currentPageDocs.value * pageSize
        })

        if (result.docs.length < pageSize) hasMoreDocs.value = false

        for (let doc of (result.docs as Post[])) {
            if (doc._attachments) loadImage(doc)
        }

        if (reset) postsData.value = result.docs as Post[]
        else postsData.value = [...postsData.value, ...result.docs as Post[]]
        currentPageDocs.value++
    } catch (e) { }
}

// --- OPTIMISATION COMMENTAIRES (Page 9) ---

// Récupère UNIQUEMENT le dernier commentaire pour l'aperçu (Limit: 1)
// Évite de charger des milliers de commentaires inutilement.
const fetchLastComment = async (platId: string): Promise<Comment | null> => {
    if (!storage.value) return null
    try {
        const res = await storage.value.find({
            selector: { type: 'comment', plat_id: platId },
            limit: 1
        })
        return res.docs.length > 0 ? (res.docs[0] as Comment) : null
    } catch (e) { return null }
}

// Charge tous les commentaires seulement à la demande (Lazy Loading)
const loadAllComments = async (plat: Plat) => {
    if (!storage.value) return
    plat.showComments = !plat.showComments
    if (plat.showComments) {
        const result = await storage.value.find({ selector: { type: 'comment', plat_id: plat._id } })
        plat.loadedComments = result.docs as Comment[]
    }
}
const addComment = async (plat: Plat) => {
    if (!storage.value || !plat.newCommentText) return
    const comment: Comment = {
        _id: new Date().toISOString() + Math.random(), type: 'comment', plat_id: plat._id,
        text: plat.newCommentText, author: clients[Math.floor(Math.random() * clients.length)], date: new Date().toISOString()
    }
    await storage.value.put(comment)
    plat.newCommentText = ''; plat.lastComment = comment; if(plat.showComments) loadAllComments(plat)
}
const updateComment = async (plat: Plat, comment: Comment) => {
    if (!storage.value) return
    const newText = prompt("Modifier :", comment.text); if (newText && newText !== comment.text) {
        await storage.value.put({ ...comment, text: newText }); loadAllComments(plat)
    }
}
const deleteComment = async (plat: Plat, comment: Comment) => {
    if (!storage.value || !confirm("Effacer ?")) return
    await storage.value.remove(comment); loadAllComments(plat)
}

// --- UPDATE & DELETE (CRUD) ---
const updateDoc = async (post: Post) => {
    if (!storage.value) return
    const newContent = prompt("Modifier :", post.content); if (newContent && newContent !== post.content) {
        await storage.value.put({ ...post, content: newContent, isModified: true }); loadMoreDocs(true)
    }
}
const updatePlat = async (plat: Plat) => {
    if (!storage.value) return
    const newName = prompt("Nom :", plat.nom); if(!newName) return;
    const newPrice = parseFloat(prompt("Prix :", plat.prix.toString()) || '0');
    await storage.value.put({ ...plat, nom: newName, prix: newPrice, isModified: true }); loadMorePlats(true)
}

/**
 * SUPPRESSION EN CASCADE
 * Si on supprime un Plat, on doit nettoyer la base en supprimant
 * tous les commentaires orphelins associés via 'bulkDocs'.
 */
const deleteEntity = async (doc: any) => {
  if (!storage.value) return
  if (doc.type === 'plat') {
      const res = await storage.value.find({ selector: { type: 'comment', plat_id: doc._id } })
      if (res.docs.length) await storage.value.bulkDocs(res.docs.map(c => ({ ...c, _deleted: true })))
  }
  await storage.value.remove(doc);
  if(doc.type === 'plat') loadMorePlats(true); else loadMoreDocs(true)
}
const addLike = async (doc: any) => {
    if (!storage.value) return;
    await storage.value.put({...doc, likes: (doc.likes||0)+1});
    if(doc.type === 'plat') loadMorePlats(true); else loadMoreDocs(true)
}

// --- SEARCH & CREATE ---
// La recherche utilise les Regex de CouchDB ($regex)
const search = async (event: Event) => {
  const q = (event.target as HTMLInputElement).value.toLowerCase().trim()
  if (!q) { loadMorePlats(true); loadMoreDocs(true); return }
  if (!storage.value) return
  const rPosts = await storage.value.find({ selector: { type: 'post', title: { $regex: RegExp(q, "i") } } })
  postsData.value = rPosts.docs as Post[]; hasMoreDocs.value = false
  const rPlats = await storage.value.find({ selector: { type: 'plat', nom: { $regex: RegExp(q, "i") } } })
  platsData.value = rPlats.docs as Plat[]; hasMorePlats.value = false
}
const searchReset = () => { (document.querySelector('.search') as HTMLInputElement).value = ''; loadMorePlats(true); loadMoreDocs(true); }

const createPlat = async () => {
  if (!storage.value) return
  await storage.value.post({ type: 'plat', nom: platsNames[Math.floor(Math.random()*platsNames.length)], client: clients[Math.floor(Math.random()*clients.length)], ingredients: 'Tomate', prix: 15, likes: 0 })
  loadMorePlats(true)
}
const createDoc = async () => {
  if (!storage.value) return
  await storage.value.post({ type: 'post', title: 'Doc ' + Math.floor(Math.random()*1000), content: words[Math.floor(Math.random()*words.length)], likes: 0, attributes: { creation_date: new Date().toISOString() } })
  loadMoreDocs(true)
}
const createMultipleDocs = async () => {
    if (!storage.value) return; const num = parseInt(batchSize.value.toString()) || 0; const docs = []
    for (let i=0; i<num; i++) docs.push({ type: 'post', title: 'Doc Batch ' + i, content: '...', likes: 0, attributes: { creation_date: new Date().toISOString() } })
    await storage.value.bulkDocs(docs); loadMoreDocs(true)
}
const createMultiplePlats = async () => {
  if (!storage.value) return; const num = parseInt(batchSize.value.toString()) || 0; const plats = []
  for (let i=0; i<num; i++) plats.push({ type: 'plat', nom: 'Batch', client: 'Batch', ingredients: '...', prix: 10, likes: 0 })
  await storage.value.bulkDocs(plats); loadMorePlats(true)
}
</script>

<template>
  <div id="app-container">
    <div class="top-bar">
        <h1>Infradon : TP Final (Complet & Commenté)</h1>
        <button @click="resetDatabase" class="btn-danger-outline">🔥 Reset DB</button>
    </div>

    <input type="file" ref="fileInput" style="display: none" @change="processUpload" accept="image/*">

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
            <h3>📑 Documents (Top Likes)</h3>
            <div class="document-list">
              <article v-for="post in postsData" :key="post._id" class="card-doc">
                <div class="article-content">
                    <h4>{{ post.title }} <span v-if="post.isModified" class="modified-tag">(modifié)</span></h4>
                    <p>{{ post.content }}</p>

                    <div class="image-area">
                        <img v-if="post.imageURL" :src="post.imageURL" class="preview-img" />
                        <div class="img-actions">
                            <button v-if="!post.imageURL" @click="triggerUpload(post)" class="btn-tiny">📎 Ajouter Image</button>
                            <button v-else @click="deleteImage(post)" class="btn-tiny delete">🗑️ Suppr Image</button>
                        </div>
                    </div>
                </div>
                <div class="article-actions">
                    <button @click="addLike(post)" class="btn-like">❤️ {{ post.likes }}</button>
                    <button @click="updateDoc(post)" class="btn-update">📝</button>
                    <button @click="deleteEntity(post)" class="btn-delete">X</button>
                </div>
              </article>
              <button v-if="hasMoreDocs" @click="loadMoreDocs(false)" class="btn-load-more">Charger plus...</button>
              <p v-if="postsData.length === 0" class="empty">Aucun document.</p>
            </div>
        </div>

        <div class="column">
            <h3>🍲 Commandes (Top Likes)</h3>
            <div class="document-list">
              <article v-for="plat in platsData" :key="plat._id" class="card-plat">
                <div class="plat-header">
                    <div>
                        <h4>{{ plat.nom }} <small>({{ plat.client }})</small> <span v-if="plat.isModified" class="modified-tag">(modifié)</span></h4>
                        <p class="price">{{ plat.prix }} €</p>
                        <div v-if="plat.lastComment" class="last-comment-preview">Dernier avis : <em>"{{ plat.lastComment.text }}"</em></div>

                        <div class="image-area">
                            <img v-if="plat.imageURL" :src="plat.imageURL" class="preview-img" />
                            <div class="img-actions">
                                <button v-if="!plat.imageURL" @click="triggerUpload(plat)" class="btn-tiny">📎 Image</button>
                                <button v-else @click="deleteImage(plat)" class="btn-tiny delete">🗑️</button>
                            </div>
                        </div>
                    </div>
                    <div class="article-actions">
                        <button @click="addLike(plat)" class="btn-like">❤️ {{ plat.likes }}</button>
                        <button @click="updatePlat(plat)" class="btn-update">📝</button>
                        <button @click="deleteEntity(plat)" class="btn-delete">X</button>
                    </div>
                </div>

                <button @click="loadAllComments(plat)" class="btn-comments">💬 {{ plat.showComments ? 'Masquer' : 'Voir avis' }}</button>

                <div v-if="plat.showComments" class="comments-section">
                    <div class="comments-list">
                        <div v-if="!plat.loadedComments || plat.loadedComments.length === 0" class="no-com">Pas encore d'avis.</div>
                        <div v-for="com in plat.loadedComments" :key="com._id" class="comment-item">
                            <div class="comment-content"><strong>{{ com.author }}:</strong> {{ com.text }}</div>
                            <div class="comment-actions">
                                <button @click="updateComment(plat, com)" class="btn-tiny update">✏️</button>
                                <button @click="deleteComment(plat, com)" class="btn-tiny delete">❌</button>
                            </div>
                        </div>
                    </div>
                    <div class="add-comment">
                        <input type="text" v-model="plat.newCommentText" placeholder="Avis..." @keyup.enter="addComment(plat)">
                        <button @click="addComment(plat)">Envoyer</button>
                    </div>
                </div>
              </article>
              <button v-if="hasMorePlats" @click="loadMorePlats(false)" class="btn-load-more">Charger plus...</button>
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
.btn-load-more { background: #f3f4f6; color: #333; width: 100%; margin-top: 10px; border: 1px solid #ddd; }
.btn-tiny { padding: 2px 6px; font-size: 0.7rem; border-radius: 4px; cursor: pointer; border: 1px solid #ccc; background: #fff; margin-top: 5px;}
.btn-tiny.delete { background: #fee2e2; color: #b91c1c; border: none; }
.btn-tiny.update { background: #e0f2fe; color: #0284c7; border: none; }

.image-area { margin-top: 10px; }
.preview-img { max-width: 100%; max-height: 150px; border-radius: 5px; border: 1px solid #eee; display: block;}
.img-actions { display: flex; gap: 5px; }

.modified-tag { font-size: 0.7em; color: #888; font-style: italic; margin-left: 5px; font-weight: normal; }
.last-comment-preview { font-size: 0.8em; color: #666; margin-top: 5px; border-left: 2px solid #ddd; padding-left: 5px; }

.card-doc { background: #f3f4f6; border-left: 5px solid #2563eb; padding: 1rem; margin-bottom: 1rem; border-radius: 0.5rem; display: flex; justify-content: space-between; align-items: flex-start; }
.card-plat { background: #fff7ed; border-left: 5px solid #ea580c; padding: 1rem; margin-bottom: 1rem; border-radius: 0.5rem; border: 1px solid #fed7aa; }
.plat-header { display: flex; justify-content: space-between; align-items: start; }

.comments-section { margin-top: 10px; background: white; padding: 10px; border-radius: 5px; border: 1px solid #ddd; }
.comment-item { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #eee; padding: 6px 0; }
.comment-content { font-size: 0.9em; flex-grow: 1; }
.comment-actions { display: flex; gap: 5px; margin-left: 10px; }

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
