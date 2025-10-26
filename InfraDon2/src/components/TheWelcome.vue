<script setup lang="ts">

import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

declare interface Post {
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref();
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:Plkjhuio0825.@localhost:5984/db_infradon2')
  if (db) {
    console.log("Connecté à la collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données
// https://pouchdb.com/api.html#batch_fetch
const fetchData = () => {
  storage.value.allDocs({
    include_docs: true
  }).then(function (result: any) {
    console.log('=>Données récuperées', result.rows);
    postsData.value = result.rows.map((row: any) => row.doc);
  }).catch(function (err: any) {
    console.error('=> Erreur lors de la récupération des données :', err)
  });
}


const addDocument = function (doc: any) {
  storage.value
    .post(doc)
    .then(function (result: any) {
      fetchData();
    })
    .catch(function (err: any) {
      console.log(err);
    })
}

const doc = {
  post_name: "Nom de post",
  post_content: "Contenu du post",
  comments: [
    {
      title: "Hello",
      author: "ES",
    },
    {
      title: "Hi",
      author: "ES",
    }
  ]
}


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  fetchData();
  addDocument(doc);
});


</script>

<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
  <button @click="addDocument(doc)">Nouveau Document</button>
</template>
