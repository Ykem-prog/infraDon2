<script setup lang="ts">
import { onMounted, ref, type Ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  title: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')

  const db = new PouchDB('http://ykem:F4bd3df077e@localhost:5984/infradon2_db')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données

// TODO Récupération des données
// https://pouchdb.com/api.html#batch_fetch
// Regarder l'exemple avec function allDocs
// Remplir le tableau postsData avec les données récupérées
const titre = ref('')
const actor = ref('')
const content = ref('')
const id: Ref<string> = ref('')
const addData = () =>
  storage.value
    .post({
      title: titre.value,
      post_content: content.value,
      actor: actor.value,
    })
    .then(function (response) {
      console.log(response)
      fetchData()
      titre.value = ''
      actor.value = ''
      content.value = ''
      // handle response
    })
    .catch(function (err) {
      console.log(err)
    })
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
      console.log(postsData.value)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
const getData = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      titre.value = doc.title
      content.value = doc.post_content
      actor.value = doc.actor
      id.value = doc._id
      return storage.value.put(doc)
    })
    .then((response: any) => {
      console.log('Document modifié', response)
      fetchData()
    })
}
const deleteData = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      return storage.value.remove(doc)
    })
    .then((response: any) => {
      console.log('Document supprimé', response)
      fetchData()
    })
}
const modifiedData = (id: string) => {
  storage.value
    .get(id)
    .then((doc: any) => {
      doc.title = titre.value
      doc.post_content = content.value
      doc.actor = actor.value
      return storage.value.put(doc)
    })
    .then((response: any) => {
      console.log('Document modifié', response)
      fetchData()
    })
}
//v-model permet de lier les champs du formulaire aux variables du script
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addData">Ajouter Document</button>
  <button @click="modifiedData(id)">Modifier</button>

  <form action="" @submit="addData">
    <label for="titre">Titre:</label><br />
    <input type="text" id="titre" name="titre" v-model="titre" /><br />
    <label for="actorName">Acteur:</label><br />
    <input type="text" id="actorName" name="lname" v-model="actor" />
    <br />
    <label for="content">Contenu:</label><br />
    <input type="text" id="content" name="content" v-model="content" /><br />
  </form>

  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <h2>{{ post.post_content }}</h2>
    <h2>{{ post.actor }}</h2>

    <button @click="getData(post._id)">Get Data</button>
    <button @click="deleteData(post._id)">Delete Data</button>
  </article>
</template>
