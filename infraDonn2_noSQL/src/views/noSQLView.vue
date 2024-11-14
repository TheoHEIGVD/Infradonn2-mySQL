<template>
  <div class="container mx-4 my-4">
        <!-- Titre principal de la page -->
    <h1 class="text-2xl mb-4">Posts Database</h1>
    
    <!-- Bouton pour ajouter un post de test -->
    <button 
      class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mb-4"
      @click="addFakePost" 
    >
      Ajouter un post de test
    </button>


 <!-- Liste des posts -->
      <div class="space-y-4">
              <!-- Boucle sur chaque post dans `postsData` pour les afficher -->
      <div v-for="post in postsData" :key="post._id" class="border p-4 rounded">
        
        
        <!-- Affichage normal du post si l'utilisateur n'est pas en train de l'éditer -->        <div v-if="editingPost?._id !== post._id">
          <h3 class="font-bold">{{ post.doc.post_name }}</h3>
          <p class="my-2">{{ post.doc.post_content }}</p>
          <p class="text-sm text-gray-500">
            Créé le: {{ new Date(post.doc.attributes.creation_date).toLocaleString() }}
          </p>


         <!-- Boutons pour éditer ou supprimer le post -->
          <div class="mt-2 space-x-2">
            <button 
              @click="editPost(post)"
              class="bg-yellow-500 hover:bg-yellow-700 text-white font-bold py-1 px-3 rounded"
            >
              Modifier
            </button>
            <button 
              @click="deleteDocument(post._id)"
              class="bg-red-500 hover:bg-red-700 text-white font-bold py-1 px-3 rounded"
            >
              Supprimer
            </button>
          </div>
        </div>


        <!-- Formulaire d'édition du post si l'utilisateur est en mode édition -->
        <div v-else class="space-y-2">
          <input 
            v-model="editingPost.doc.post_name"
            class="w-full p-2 border rounded"
            type="text"
          >
          <textarea 
            v-model="editingPost.doc.post_content"
            class="w-full p-2 border rounded"
            rows="3"
          ></textarea>
          <div class="space-x-2">
            <button 
              @click="saveEdit"
              class="bg-green-500 hover:bg-green-700 text-white font-bold py-1 px-3 rounded"
            >
              Sauvegarder
            </button>
            <button 
              @click="cancelEdit"
              class="bg-gray-500 hover:bg-gray-700 text-white font-bold py-1 px-3 rounded"
            >
              Annuler
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Message d'état vide si aucun post n'est présent -->
    <div v-if="postsData.length === 0" class="text-center py-8 text-gray-500">
      Aucun post trouvé. Cliquez sur "Ajouter un post de test" pour commencer.
    </div>
  </div>
</template>

<script lang="ts">
import { ref } from 'vue';       // Importation de `ref` pour réactiver les données de l'application
import PouchDB from 'pouchdb';   // Importation de la bibliothèque PouchDB pour gérer les bases de données

// DECLARATION DE L'INTERFACE POUR LES POSTS 
declare interface Post {
  _id: string;
  doc: {
    post_name: string;
    post_content: string;
    attributes: {
      creation_date: string;
    };
  };
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      editingPost: null as Post | null,
      localDb: null as PouchDB.Database | null,
      remoteDb: null as PouchDB.Database | null,
    };
  },

  mounted() {
    this.initDatabase();
    this.fetchData();
  },

  methods: {
    generateFakePost(): any {
      const timestamp = new Date().toISOString();
      // Structure modifiée pour correspondre à l'interface Post
      return {
        _id: timestamp,
        doc: {
          post_name: `Post ${Math.floor(Math.random() * 1000)}`,
          post_content: `Contenu exemple ${Math.random().toString(36).substring(7)}`,
          attributes: {
            creation_date: timestamp,
          },
        }
      };
    },
    // Ajoute un faux post à la base de données

    async addFakePost() {
      try {
        const fakePost = this.generateFakePost();
        if (this.storage) {
          // Supprime la propriété "doc" avant d'enregistrer
          const docToSave = {
            _id: fakePost._id,
            ...fakePost.doc
          };
          await this.storage.put(docToSave);
          console.log('Post ajouté avec succès');
          await this.fetchData();
        }
      } catch (error) {
        console.error('Erreur lors de l\'ajout du post:', error);
      }
    },
    // Récupère tous les documents de la base de données
    async fetchData() {
      if (this.storage) {
        try {
          const result = await this.storage.allDocs({
            include_docs: true,
            attachments: true
          });
          console.log('Données récupérées:', result);
          this.postsData = result.rows.map(row => ({
            _id: row.id,
            doc: {
              post_name: row.doc.post_name,
              post_content: row.doc.post_content,
              attributes: {
                creation_date: row.doc.attributes?.creation_date || row.doc._id
              }
            }
          }));
        } catch (error) {
          console.error('Erreur lors de la récupération des données:', error);
        }
      }
    },

           // Met à jour un document dans la base de données 
           async updateDocument(post: Post) {
      try {
        if (this.storage) {
          const existingDoc = await this.storage.get(post._id);
          const updatedDoc = {
            _id: post._id,
            _rev: existingDoc._rev,
            post_name: post.doc.post_name,
            post_content: post.doc.post_content,
            attributes: {
              creation_date: post.doc.attributes.creation_date
            }
          };
          await this.storage.put(updatedDoc);
          console.log('Mise à jour réussie');
          await this.fetchData();
        }
      } catch (error) {
        console.error('Erreur lors de la mise à jour:', error);
      }
    },

// Met le post en mode édition
    editPost(post: Post) {
      this.editingPost = JSON.parse(JSON.stringify(post));
    },

    // Sauvegarde les modifications faites sur le post en édition
    async saveEdit() {
      if (this.editingPost) {
        await this.updateDocument(this.editingPost);
        this.editingPost = null;
      }
    },

    // Annule l'édition en réinitialisant `editingPost`
    cancelEdit() {
      this.editingPost = null;
    },



    // Supprime un document par son identifiant
    async deleteDocument(postId: string) {
      try {
        if (this.storage) {
          const doc = await this.storage.get(postId);
          await this.storage.remove(doc);
          console.log('Suppression réussie');
          this.fetchData();
        }
      } catch (error) {
        console.error('Erreur lors de la suppression:', error);
      }
    },

    // Ajoute un document à la base de données
    putDocument(document: Post) {
      if (this.storage) {
        this.storage.put(document).then(() => {
          console.log('Add ok');
        }).catch((error) => {
          console.log('Add ko', error);
        });
      }
    },



    // Récupère tous les documents de la base de données
    async initDatabase() {
      try {
        this.localDb = new PouchDB('local_database');
        console.log('Base de données locale créée');

        this.remoteDb = new PouchDB('http://localhost:5984/database');
        console.log('Connecté à la base de données distante');

        this.storage = this.localDb;

        await this.syncDatabases();
        this.setupLiveSync();
      } catch (error) {
        console.error('Erreur lors de l\'initialisation des bases de données:', error);
      }
    },

    async syncDatabases() {
      if (this.localDb && this.remoteDb) {
        try {
          const result = await this.localDb.sync(this.remoteDb);
          console.log('Synchronisation réussie:', result);
        } catch (error) {
          console.error('Erreur de synchronisation:', error);
        }
      }
    },

    // Synchronise les bases
    setupLiveSync() {
      if (this.localDb && this.remoteDb) {
        PouchDB.sync(this.localDb, this.remoteDb, {
          live: true,
          retry: true,
        })
        .on('change', (change) => {
          console.log('Changement détecté:', change);
          this.fetchData();
        })
        .on('error', (err) => {
          console.error('Erreur de synchronisation:', err);
        });
      }
    },
  },
};
</script>