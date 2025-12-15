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
  likes: number;
}

declare interface Comment {
  _id: string;
  _rev: string;
  id_post: string;
  comment_content: string;
  attributes: {
    creation_date: any;
  };
}

// Référence à la base de données
const storage = ref();
const storageComments = ref();
let offline = ref(false);
let syncRunning = false;
const sync = ref();
const syncComments = ref();
const needle = ref([]);
const needleComment = ref([]);


// Données stockées
const postsData = ref<Post[]>([])
const commentsData = ref<Comment[]>([])


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  //fetchData();
});


// Doc hardcodé
const doc = {
  post_name: "Nom de post",
  post_content: "Contenu du post",
  likes: 0
}

const comment = {
  id_post: "",
  comment_content: "ceci est un super commentaire",
}



const initDatabase = () => {
  console.log('=> Connexion à la base de données');

  const db = new PouchDB('Posts');
  const dbComments = new PouchDB('Comments');

  storage.value = db;
  storageComments.value = dbComments;

  if (!db || !dbComments) {
    console.warn("Échec lors de la connexion aux bases");
    return;
  }

  console.log('Connecté aux collections : ' + db?.name, dbComments?.name);

  // index nom du post
  storage.value.createIndex({
    index: {
      fields: ['post_name']
    }
  }).then(console.log("the index has been created!"));


  // index likes
  storage.value.createIndex({
    index: { fields: ['likes'] }
  }).then(() => console.log("Index likes created"));


  // RÉPLICATION POSTS
  storage.value
    .replicate
    .from("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2")
    .on("complete", () => {
      console.log("Replication POSTS terminée");
      fetchData();
      if (!offline.value) startSync();
    });

  // RÉPLICATION COMMENTS
  storageComments.value
    .replicate
    .from("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2_comments")
    .on("complete", () => {
      console.log("Replication COMMENTS terminée");
      fetchData();                 // <-- 100% safe
      if (!offline.value) startSync();
    });
}



const startSync = () => {
  if (syncRunning) {
    console.log("sync already running");
    return;
  }

  if (!storage.value) {
    console.warn('No local DB to sync');
    return;
  }

  if (!storageComments.value) {
    console.warn('No local DB to sync');
    return;
  }

  syncRunning = true;

  sync.value = storage.value.sync("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2",
    {
      live: true,
      retry: true
    })
    .on('change', fetchData)
    .on("paused", (err: any) => {
      if (offline.value) {
        console.log("Sync paused because offline");
      }
    })
    .on("error", (err: any) => {
      console.log("Sync error :", err);
      syncRunning = false;
    });

  syncComments.value = storageComments.value.sync("http://admin:Plkjhuio0825.@localhost:5984/db_infradon2_comments",
    {
      live: true,
      retry: true
    })
    .on('change', fetchData)
    .on("error", () => {
      syncRunning = false;
    });

  console.log('Sync started');
}


const stopSync = () => {
  if (!syncRunning) return;

  try {
    if (sync.value) {
      sync.value.cancel()
      sync.value = null
    }

    if (syncComments.value) {
      syncComments.value.cancel()
      syncComments.value = null
    }

    syncRunning = false;
    console.log('Sync cancelled')

  } catch (err) {
    console.error('Error cancelling sync', err)
  }
}


const handleChange = () => {
  if (!offline.value) {
    console.log("Mode ONLINE → lancement du sync");
    startSync();
  } else {
    console.log("Mode OFFLINE → arrêt du sync");
    stopSync();
  }
}


// Récupération des données + affichage
// https://pouchdb.com/api.html#batch_fetch
const fetchData = () => {

  if (!storage.value) return;

  storage.value.find({
    selector: {
      likes: { $gte: 0 }  
    },
    sort: [{ likes: "desc" }]
  })
    .then((result: any) => {
      console.log('=> Posts récupérés triés par likes :', result.docs);
      postsData.value = result.docs.filter(d => !(d._id.startsWith("_design")));
    })
    .catch((err: any) => {
      console.error('=> Erreur lors de la récupération des posts triés :', err);
    });

  // COMMENTS
  if (!storageComments.value) return;

  storageComments.value.allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Commentaires récupérés :', result.rows);
      commentsData.value = result.rows.map((row: any) => row.doc).filter(c => !(c._id.startsWith("_design")));;;
    })
    .catch((err: any) => {
      console.error('=> Erreur lors de la récupération des commentaires :', err);
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
const updateDocument = function (doc: any, i) {
  doc.post_name = needle.value[i];
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
      post_name: "Doc " + i,
      post_content: words[Math.floor(Math.random() * words.length)],
      likes: 0
    });
  }

  fetchData();
};


const search = (event: any) => {
  const value = event.target.value.trim();

  if (!value) {
    return fetchData();
  }

  storage.value.find({
    selector: { post_name: event.target.value }
  })

    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      postsData.value = result.docs;

    })
    .catch((error: any) => {
      console.error(error)
    })
}


const handleLike = (doc: any) => {
  doc.likes++;
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


// COMMENTAIRES
const addComment = function (c: any, post_id: any) {
  if (!storageComments.value) {
    console.warn("La DB Comments n'est pas prête !");
    return;
  }

  c.id_post = post_id;
  storageComments.value
    .post(c)
    .then(function (result: any) {
      fetchData();
    })
    .catch(function (err: any) {
      console.log(err);
    })
}

// Modification d'un document
const updateComment = function (c: any, i: any) {
  c.comment_content = needleComment.value[i];
  storageComments.value
    .put(c)
    .then(function (result: any) {
      fetchData();
      console.log(result);
    })
    .catch(function (err: any) {
      console.log(err);
    })
}

// Suppression d'un document
const deleteComment = function (id: string, rev: string) {
  storageComments.value
    .remove(id, rev)
    .then(function (result: any) {
      fetchData();
      console.log(result);
    }).catch(function (err: any) {
      console.log(err);
    })
}


const getComments = (postId: any) => {
  const commentsFiltered = commentsData.value.filter(a => a.id_post === postId)
  return commentsFiltered;
}


</script>

<template>
  <h1>Fetch Data</h1>
  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
  <br>
  <button @click="createDocument(doc)">Nouveau Document</button>
  <button @click="generate200Docs()">Générer 200 documents</button>
  <!-- <button @click="deleteAllDocs()">Supprimer tous les documents</button> -->
  <input type="text" placeholder="Search" @keyup.enter="search" class="search">
  <br>
  Nombre de post : {{ postsData.length }}
  <article v-for="(post, i) in postsData" v-bind:key="(post as any).id">
    <ul>
      <li>
        <hr>
        <h2>{{ post.post_name }}</h2>
        <p>{{ post.post_content }}</p>
        <p>Likes: {{ post.likes }}</p>

        <label for="needle">Editer le post</label>
        <input type="text" name="needle" v-model="needle[i]" @keyup.enter="updateDocument(post, i)">

        <button @click="deleteDocument(post._id, post._rev)">Supprimer le document</button>
        <button @click="handleLike(post)">Liker le document</button>
        <button @click="addComment(comment, post._id)">Nouveau commentaire</button>


        <article v-for="(comment, i) in getComments(post._id)" v-bind:key="(comment as any)._id">

          <ul>
            <li>
              <h2>{{ comment.comment_content }}</h2>

              <label for="needleComments">Editer le commentaire</label>
              <input type="text" name="needleComments" v-model="needleComment[i]"
                @keyup.enter="updateComment(comment, i)">
              <button @click="deleteComment(comment._id, comment._rev)">Supprimer le commentaire</button>
            </li>
          </ul>

        </article>

      </li>
    </ul>
  </article>
</template>