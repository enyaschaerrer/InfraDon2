<script setup lang="ts">
// lien Couch DB: http://127.0.0.1:5984/_utils/#database/db_infradon2/_all_docs
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind);

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
const sync = ref();

// Données stockées
const postsData = ref<Post[]>([])

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  //fetchData();
});

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


const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('Posts');
  if (db) {
    console.log('Connecté à la collection : ' + db?.name);
    storage.value = db;
    storage.value.createIndex({
      index: {
        fields: ['post_content']
      }
    }).then(console.log("the index has been created!"));

    storage.value.replicate.from("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2")
    fetchData()
    if (!offline.value) {
      startSync();
    }

  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}


const startSync = () => {
  if (sync.value) {
    console.log("sync already running");
    return;
  }
  if (!storage.value) {
    console.warn('No local DB to sync');
    return;
  }

  sync.value = storage.value.sync("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2",
    {
      live: true,
      retry: true
    })
    .on('change', (info: any) => {
      console.log("sync change", info);
      fetchData();
    })

  console.log('Sync started');
}


const stopSync = () => {
  try {
    sync.value.cancel()
    sync.value = null
    console.log('Sync cancelled')
  } catch (err) {
    console.error('Error cancelling sync', err)
  }
}


const handleChange = () => {

  if (!offline.value) {
    console.log("Mode ONLINE => lancement du sync");
    startSync();
  } else {
    console.log("Mode OFFLINE → arrêt du sync");
    stopSync();
  }
}


// Récupération des données + affichage
// https://pouchdb.com/api.html#batch_fetch
const fetchData = () => {
  storage.value.allDocs({
    include_docs: true

  }).then((result: any) => {
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
      console.log(result);
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
      console.log(result);
    }).catch(function (err: any) {
      console.log(err);
    })
}


const generate200Docs = (count = 200) => {
  const words = ["léa", "inoé", "camilo", "yannis", "sarah", "tanguy", "dylan"];

  for (let i = 0; i < count; i++) {
    storage.value.post({
      title: "Doc " + i,
      post_content: words[Math.floor(Math.random() * words.length)]
    });
  }

  fetchData();
};


// const deleteAllDocs = () => {
//   if (storage.value) {
//     const result = await storage.value.bulkDocs(storage.value.allDocs());
//     console.log("Tous les documents ont été supprimés :", result);

//     // Rafraîchir les données affichées
//     fetchData();
//   } else {
//     console.error("Erreur lors de la suppression de tous les documents :", err);
//   }
// }


const search = (event: Event) => {
  const value = event.target.value.trim();

  if (!value) {
    return fetchData();
  }

  storage.value.find({
    selector: { post_content: event.target.value }
  })

    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      postsData.value = result.docs;

    })
    .catch((error: any) => {
      console.error(error)
    })
}



</script>

<template>
  <h1>Fetch Data</h1>
  <p>Mode offline {{ offline }}</p>
  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
  <br>
  <button @click="createDocument(doc)">Nouveau Document</button>
  <button @click="generate200Docs()">Générer 200 documents</button>
  <!-- <button @click="deleteAllDocs()">Supprimer tous les documents</button> -->
  <input type="text" placeholder="Search" @keyup.enter="search" class="search">
  <br>
  <!-- <button v-if="!offline" id="validate" @click="replicateDB()">Valider les changements</button> -->
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