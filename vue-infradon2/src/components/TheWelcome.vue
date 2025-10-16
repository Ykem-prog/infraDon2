<script setup lang="ts">
import { onMounted, ref } from 'vue'
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

const addData = () =>
  storage.value
    .post({
      title: 'Heroes',
    })
    .then(function (response) {
      console.log(response)
      fetchData()
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
</script>

<template>
  <h1>Fetch Data</h1>
  <button @click="addData">Add Data</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
  </article>
</template>
