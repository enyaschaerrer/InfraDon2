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
  _attachments?: {};
}

declare interface Comment {
  _id: string;
  _rev: string;
  id_post: string;
  comment_content: string;
  attributes: {
    creation_date: string;
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
let $displayPosts = ref(10);

// si l'id du post est dans le tableau, on affiche tous ses commentaires
const showAllComments = ref<{ [key: string]: boolean }>({});


// Données stockées
const postsData = ref<Post[]>([])
const commentsData = ref<Comment[]>([])


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
});


// Doc hardcodé
const doc = {
  post_name: "Nom de post",
  post_content: "Contenu du post",
  likes: 0
}


// Initialisation des 2 databases
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
  }).then(console.log("the index on the post name has been created!"));


  // index likes
  storage.value.createIndex({
    index: { fields: ['likes'] }
  }).then(() => console.log("Index likes created"));

  storageComments.value.createIndex({
    index: { fields: ['id_post'] }
  }).then(() => console.log("Index on post linked created"));


  // RÉPLICATION POSTS
  storage.value
    .replicate
    .from("http://admin:Plkjhuio0825.@localhost:5984/posts_db_enyaschaerrer")
    .on("complete", () => {
      console.log("Replication POSTS terminée");
      fetchData();
      if (!offline.value) startSync();
    });


  // RÉPLICATION COMMENTS
  storageComments.value
    .replicate
    .from("http://admin:Plkjhuio0825.@localhost:5984/comments_db_enyaschaerrer")
    .on("complete", () => {
      console.log("Replication COMMENTS terminée");
      fetchData();
      if (!offline.value) startSync();
    });
}


// démarrer synchronisation
const startSync = () => {
  if (syncRunning) {
    console.log("sync already running");
    return;
  }

  if (!storage.value || !storageComments.value) {
    console.warn('No local DB to sync');
    return;
  }

  syncRunning = true;

  sync.value = storage.value.sync("http://admin:Plkjhuio0825.@localhost:5984/posts_db_enyaschaerrer",
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

  syncComments.value = storageComments.value.sync("http://admin:Plkjhuio0825.@localhost:5984/comments_db_enyaschaerrer",
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


// arrêter synchronisation
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


// gestion du offline ou online
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

  // chargement des posts les plus likés
  storage.value.find({
    selector: {
      likes: { $gte: 0 }
    },
    sort: [{ likes: "desc" }],
    limit: $displayPosts.value
  })
    .then((result: any) => {
      console.log('=> Posts récupérés triés par likes :', result.docs);
      postsData.value = result.docs.filter(d => !(d._id.startsWith("_design")));
    })
    .catch((err: any) => {
      console.error('=> Erreur lors de la récupération des posts triés :', err);
    });


  // chargement de tous les commentaires car on veut pouvoir facilement tous les afficher sous un post,
  // pas comme les posts
  if (!storageComments.value) return;

  storageComments.value.allDocs({
    include_docs: true
  })
    .then((result: any) => {
      commentsData.value = result.rows
        .map(row => row.doc)
        .filter(c => c && !c._id.startsWith("_design"));
      console.log('=> Commentaires récupérés :', commentsData.value.length);
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
const updateDocument = function (doc: any, i: number) {
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


// Factory de docs
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


// fonction de recherche sur l'index qui est sur le nom du post (=doc)
const search = (event: any) => {
  const value = event.target.value.trim();

  if (!value) {
    return fetchData();
  }

  storage.value.find({
    selector: {
      post_name: {
        $regex: new RegExp(value, 'i') // insensible à la casse
      }
    }
  })

    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      postsData.value = result.docs;
    })
    .catch((error: any) => {
      console.error(error)
    })
}


// Quand on like un post
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


// Ajout d'un commentaire
const addComment = function (post_id: any) {
  if (!storageComments.value) {
    console.warn("La DB Comments n'est pas prête !");
    return;
  }

  const c = {
    id_post: post_id,
    comment_content: "ceci est un super commentaire",
    attributes: {
      creation_date: new Date().toISOString()
    }
  };

  storageComments.value
    .post(c)
    .then(function (result: any) {
      fetchData();
    })
    .catch(function (err: any) {
      console.log(err);
    })
}


// Modification d'un commentaire
const updateComment = function (c: any, i: string) {
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


// Suppression d'un commentaire
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


// Obtenir tous les commentaires d'un post
const getAllComments = function (postId: string) {
  return commentsData.value
    .filter(c => c.id_post === postId)
    .sort((a, b) =>
      new Date(b.attributes.creation_date).getTime() - new Date(a.attributes.creation_date).getTime()
    )
}


// Dernier commentaire d'un post
const getLastComment = function (postId: string) {
  const comments = getAllComments(postId);
  // s'il n'y a pas de comms
  if (comments.length <= 0) return;
  return comments[0];
}


// charger 10 posts supplémentaires
const changeDisplayPosts = () => {
  $displayPosts.value += 10;
  fetchData();
}


// Retourne les commentaires à afficher (seulement dernier ou tous)
const getCommentsToDisplay = function (postId: string) {
  const allComments = getAllComments(postId);

  // Si on veut voir tous les commentaires OU s'il n'y en a qu'un seul
  if (showAllComments.value[postId] || allComments.length <= 1) {
    return allComments;
  }

  // sinon, afficher seulement le dernier
  const lastComment = getLastComment(postId);
  return [lastComment];
}


// Toggle l'affichage des commentaires (dernier ou tous)
const toggleComments = function (postId: string) {
  showAllComments.value[postId] = !showAllComments.value[postId];
}


// Ajouter une image
const selectImg = function (event: any, post: Post) {
  const file = event.target.files[0];
  if (!file) return;

  if (!file.type.startsWith('image/')) {
    alert('Veuillez sélectionner une image');
    return;
  }

  storage.value.putAttachment(post._id, file.name, post._rev, file, file.type)
    .then((result: any) => {
      console.log('Image attachée avec succès:', result);
      fetchData();
    })
    .catch((err: any) => {
      console.error('Erreur lors de l\'attachement:', err);
    });
}


// Supprimer une image
const deleteImg = function (post: Post, attachmentName: string) {
  storage.value.removeAttachment(post._id, attachmentName, post._rev)
    .then((result: any) => {
      console.log('Image supprimée avec succès:', result);
      fetchData();
    })
    .catch((err: any) => {
      console.error("Erreur suppression média :", err);
    });
};


// Afficher image(s) attachée(s)
const displayImage = (post: Post, attachName: string, imgElement: HTMLImageElement) => {
  storage.value.getAttachment(post._id, attachName)
    .then((blob: Blob) => {
      const url = URL.createObjectURL(blob);
      imgElement.src = url;
    })
    .catch((err: any) => {
      console.error("Erreur chargement image:", err);
    });
};


// HTML
</script>

<template>
  <h1>Fetch Data</h1>
  <label for="mode" name="mode">Mode offline</label>
  <input type="checkbox" id="mode" name="mode" v-model="offline" @change="handleChange">
  <br>
  <button @click="createDocument(doc)">Nouveau Document</button>
  <button @click="generate200Docs()">Générer 200 documents</button>
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

        <br>


        <!-- Ajouter une img -->
        <br>
        <label for="img">Ajouter une image</label>
        <input type="file" name="img" accept="image/*" @change="selectImg($event, post)">


        <!-- Affichage des images attachées -->
        <div v-if="post._attachments && Object.keys(post._attachments).length > 0">
          <h4>Image(s) :</h4>
          <div v-for="(attachment, name) in post._attachments" :key="name">
            <img :ref="(el) => el && displayImage(post, name, el)" />
            <br>

            <!-- supprimer img -->
            <button @click="deleteImg(post, name)">Supprimer l'image</button>
          </div>
        </div>


        <!-- Commentaires -->
        <br>
        <button @click="addComment(post._id)">Nouveau commentaire</button>
        <br>

        <p>Commentaires ({{ getAllComments(post._id).length }})</p>

        <!-- gère affichage de 1 ou tous les commentaires -->
        <article v-for="(comment, j) in getCommentsToDisplay(post._id)" v-bind:key="comment._id">
          <ul>
            <li>
              <h3>{{ comment.comment_content }}</h3>

              <label for="needleComments">Editer le commentaire</label>
              <input type="text" name="needleComments" v-model="needleComment[comment._id]"
                @keyup.enter="updateComment(comment, comment._id)">
              <button @click="deleteComment(comment._id, comment._rev)">Supprimer le commentaire</button>
            </li>
          </ul>
        </article>

        <!-- load tous les commentaires/ un seulement -->
        <button v-if="getAllComments(post._id).length > 1" @click="toggleComments(post._id)">
          {{ showAllComments[post._id] ? 'Voir moins' : 'Voir tous les commentaires' }}
        </button>

      </li>
    </ul>
  </article>

  <!-- load 10 posts de plus -->
  <button @click="changeDisplayPosts()">Charger plus de posts</button>

</template>