<script setup lang="ts">

// lien Couch DB: http://127.0.0.1:5984/_utils/#database/db_infradon2/_all_docs

import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

declare interface Post {
  _id: string;
  _rev: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref();
let offline = ref(false);

// Données stockées
const postsData = ref<Post[]>([])

// Doc hardcodé
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

// Initialisation de la base de données
// const initDatabase = () => {
//   console.log('=> Connexion à la base de données');
//   const db = new PouchDB('http://admin:Plkjhuio0825.@localhost:5984/db_infradon2')
//   if (db) {
//     console.log("Connecté à la collection : " + db?.name)
//     storage.value = db
//   } else {
//     console.warn('Echec lors de la connexion à la base de données')
//   }
// }


const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('Posts');
  if (db) {
    console.log('Connecté à la collection : ' + db?.name);
    storage.value = db;
    db.replicate.from("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2").then();
    fetchData();
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}


const replicateDB = () => {
  storage.value.replicate.to("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2")
}


// Récupération des données + affichage
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

// Création d'un document
const createDocument = function (doc: any) {
  storage.value
    .post(doc)
    .then(function (result: any) {
      fetchData();
    })
    .catch(function (err: any) {
      console.log(err);
    })
}

// Modification d'un document
const updateDocument = function (doc: any) {
  doc.post_name = "modifié";
  storage.value
    .put(doc)
    .then(function (result: any) {
      fetchData();
    })
    .catch(function (err: any) {
      console.log(err);
    })
}

// Suppression d'un document
const deleteDocument = function (id: string, rev: string) {
  storage.value
    .remove(id, rev)
    .then(function (result: any) {
      fetchData();
    }).catch(function (err: any) {
      console.log(err);
    })
}


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  fetchData();
  //createDocument(doc);
});

let sync;

const startSync = () => {
  sync = PouchDB.sync('db_infradon2', 'http://admin:Plkjhuio0825.@localhost:5984/db_infradon2').on('paused', (err) => {
    console.log(err);
  })
}

const stopSync = () => {
  sync = PouchDB.sync('db_infradon2', 'http://admin:Plkjhuio0825.@localhost:5984/db_infradon2').cancel();
  console.log("cancelled")
}

const handleChange = () => {

  if (!offline) {
    startSync();
  } else if (offline) {
    stopSync();
  }

}


</script>

<template>
  <h1>Fetch Data</h1>
  <p>Mode offline {{ offline }}</p>
  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
  <br>
  <button @click="createDocument(doc)">Nouveau Document</button>
  <br>
  <button v-if="!offline" id="validate" @click="replicateDB()">Valider les changements</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <ul>
      <li>
        <h2>{{ post.post_name }}</h2>
        <p>{{ post.post_content }}</p>
        <button @click="updateDocument(post)">Editer le document</button>
        <button @click="deleteDocument(post._id, post._rev)">Supprimer le document</button>
      </li>
    </ul>
  </article>
</template>