<script lang="ts">
import { ref, onMounted } from "vue";
import PouchDB from "pouchdb";
import PouchDBFind from "pouchdb-find";
PouchDB.plugin(PouchDBFind);

// Types
interface Post {
  _id?: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
    media?: string[];
  };
}

export default {
  data() {
    return {
      localDB: null as PouchDB.Database | null,
      remoteDB: null as PouchDB.Database | null,
      syncHandler: null as any,
      posts: [] as Post[],
      searchTerm: "",
      document: {
        post_name: "",
        post_content: "",
        attributes: {
          creation_date: new Date().toISOString(),
          media: [],
        },
      } as Post,
      editingPost: null as Post | null,
      selectedFiles: [] as File[],
      error: null as string | null,
      isLoading: false,
      isOnline: true,
    };
  },

  async mounted() {
    try {
      await this.initDatabases();
      await this.createIndex();
      await this.setupSync();
      await this.fetchPosts();
      
      window.addEventListener('online', this.handleConnectionChange);
      window.addEventListener('offline', this.handleConnectionChange);
    } catch (err) {
      this.error = "Erreur lors de l'initialisation: " + (err as Error).message;
    }
  },

  beforeUnmount() {
    window.removeEventListener('online', this.handleConnectionChange);
    window.removeEventListener('offline', this.handleConnectionChange);
    if (this.syncHandler) {
      this.syncHandler.cancel();
    }
  },

  methods: {
    handleConnectionChange() {
      this.isOnline = navigator.onLine;
      if (this.isOnline) {
        this.setupSync();
      }
    },

    async initDatabases() {
      try {
        // Base de données locale
        this.localDB = new PouchDB("posts_local");
        
        // Base de données distante
        const remoteUrl = "http://localhost:5984/database";
        this.remoteDB = new PouchDB(remoteUrl, {
          auth: {
            username: "admin",
            password: "admin"
          },
          skip_setup: false,
          ajax: {
            timeout: 30000,
            withCredentials: true
          }
        });

        console.log("Bases de données initialisées avec succès");
      } catch (err) {
        console.error("Erreur d'initialisation des bases de données:", err);
        throw err;
      }
    },

    async setupSync() {
      if (!this.localDB || !this.remoteDB || !this.isOnline) return;

      try {
        if (this.syncHandler) {
          this.syncHandler.cancel();
        }

        const syncOptions = {
          live: true,
          retry: true,
          batch_size: 50,
          batches_limit: 5,
          back_off_function: (delay: number) => {
            if (delay === 0) return 1000;
            return Math.min(delay * 3, 60000);
          }
        };

        console.log("Démarrage de la synchronisation...");

        this.syncHandler = this.localDB.sync(this.remoteDB, syncOptions)
          .on('change', (info) => {
            console.log('Changement détecté:', info);
            if (info.direction === 'pull') {
              this.fetchPosts();
            }
          })
          .on('paused', () => {
            console.log('Synchronisation en pause - Base de données à jour');
          })
          .on('active', () => {
            console.log('Synchronisation active - Transfert en cours');
          })
          .on('denied', (err) => {
            console.error('Synchronisation refusée:', err);
            this.error = "Accès refusé lors de la synchronisation";
          })
          .on('complete', (info) => {
            console.log('Synchronisation terminée:', info);
          })
          .on('error', (err) => {
            console.error('Erreur de synchronisation:', err);
            this.error = "Erreur de synchronisation: " + err.message;
          });
      } catch (err) {
        console.error("Erreur lors de la configuration de la synchronisation:", err);
        throw err;
      }
    },

    async createIndex() {
      if (!this.localDB) return;
      
      try {
        await this.localDB.createIndex({
          index: {
            fields: ['post_name', 'attributes.creation_date']
          }
        });
        console.log("Index créé avec succès");
      } catch (err) {
        console.error("Erreur lors de la création de l'index:", err);
        throw err;
      }
    },

    async fetchPosts() {
      if (!this.localDB) return;
      this.isLoading = true;

      try {
        let query: any = {
          selector: {},
          sort: [{ 'attributes.creation_date': 'desc' }]
        };

        if (this.searchTerm) {
          query.selector.post_name = { 
            $regex: RegExp(this.searchTerm, 'i') 
          };
        }

        const result = await this.localDB.find(query);
        this.posts = result.docs;
      } catch (err) {
        console.error("Erreur lors de la récupération des posts:", err);
        this.error = "Erreur lors de la récupération des posts: " + (err as Error).message;
      } finally {
        this.isLoading = false;
      }
    },

    generateFakePost() {
      const id = Date.now().toString();
      return {
        post_name: `Post Test ${id}`,
        post_content: `Contenu du post test ${id}`,
        attributes: {
          creation_date: new Date().toISOString(),
          media: [],
        },
      };
    },

    async addFakePost() {
      try {
        const fakePost = this.generateFakePost();
        await this.createDocument(fakePost);
        console.log("Post test créé avec succès");
      } catch (err) {
        console.error("Erreur lors de la création du post test:", err);
        this.error = "Erreur lors de l'ajout du post test: " + (err as Error).message;
      }
    },

    async createDocument(doc = null) {
      if (!this.localDB) return;

      try {
        const postToCreate = doc || {
          post_name: this.document.post_name,
          post_content: this.document.post_content,
          attributes: {
            creation_date: new Date().toISOString(),
            media: await this.handleFileUploads(),
          },
        };

        const result = await this.localDB.post(postToCreate);
        console.log("Document créé avec succès:", result);
        this.resetForm();
        await this.fetchPosts();
        this.error = null;
      } catch (err) {
        console.error("Erreur lors de la création du document:", err);
        throw err;
      }
    },

    async handleFileUploads() {
      if (!this.selectedFiles.length) return [];

      const mediaUrls = [];
      for (const file of this.selectedFiles) {
        try {
          const base64 = await new Promise((resolve) => {
            const reader = new FileReader();
            reader.onloadend = () => resolve(reader.result);
            reader.readAsDataURL(file);
          });
          mediaUrls.push(base64);
        } catch (err) {
          console.error("Erreur lors de l'upload du fichier:", err);
          throw err;
        }
      }
      return mediaUrls;
    },

    async updateDocument(post: Post) {
      if (!this.localDB || !post._id) return;

      try {
        const updatedPost = {
          ...post,
          attributes: {
            ...post.attributes,
            media: [...(post.attributes.media || []), ...(await this.handleFileUploads())],
          },
        };

        await this.localDB.put(updatedPost);
        console.log("Document mis à jour avec succès");
        this.editingPost = null;
        await this.fetchPosts();
        this.error = null;
      } catch (err) {
        console.error("Erreur lors de la mise à jour du document:", err);
        throw err;
      }
    },

    async deleteDocument(post: Post) {
      if (!this.localDB || !post._id) return;

      try {
        await this.localDB.remove(post);
        console.log("Document supprimé avec succès");
        await this.fetchPosts();
        this.error = null;
      } catch (err) {
        console.error("Erreur lors de la suppression du document:", err);
        throw err;
      }
    },

    async deleteMedia(post: Post, mediaIndex: number) {
      if (!post.attributes.media) return;
      
      try {
        const updatedMedia = [...post.attributes.media];
        updatedMedia.splice(mediaIndex, 1);
        
        const updatedPost = {
          ...post,
          attributes: {
            ...post.attributes,
            media: updatedMedia,
          },
        };
        
        await this.updateDocument(updatedPost);
        console.log("Média supprimé avec succès");
      } catch (err) {
        console.error("Erreur lors de la suppression du média:", err);
        throw err;
      }
    },

    resetForm() {
      this.document = {
        post_name: "",
        post_content: "",
        attributes: {
          creation_date: new Date().toISOString(),
          media: [],
        },
      };
      this.selectedFiles = [];
      this.error = null;
    },

    startEditing(post: Post) {
      this.editingPost = { ...post };
    },

    cancelEdit() {
      this.editingPost = null;
    },
  },

  watch: {
    searchTerm() {
      this.fetchPosts();
    },
  },
};
</script>

<template>
  <div class="container">
    <div class="search-bar">
      <input
        v-model="searchTerm"
        placeholder="Rechercher des posts..."
        type="text"
      />
    </div>

    <div v-if="error" class="error-message">
      {{ error }}
    </div>

    <div v-if="isLoading" class="loading">
      Chargement...
    </div>

    <form @submit.prevent="createDocument()" class="create-form">
      <h2>Créer un nouveau post</h2>
      <div>
        <label for="title">Titre:</label>
        <input id="title" v-model="document.post_name" required />
      </div>
      <div>
        <label for="content">Contenu:</label>
        <textarea
          id="content"
          v-model="document.post_content"
          required
        ></textarea>
      </div>
      <div>
        <label for="media">Médias:</label>
        <input
          type="file"
          id="media"
          multiple
          @change="(e) => selectedFiles = e.target.files || []"
        />
      </div>
      <div class="button-group">
        <button type="submit">Créer le post</button>
        <button type="button" @click="addFakePost">Ajouter un post test</button>
      </div>
    </form>

    <div class="posts-list">
      <h2>Posts</h2>
      <div v-if="posts.length === 0 && !isLoading">Aucun post trouvé.</div>
      <div v-else>
        <div v-for="post in posts" :key="post._id" class="post-item">
          <div v-if="editingPost?._id !== post._id">
            <h3>{{ post.post_name }}</h3>
            <p>{{ post.post_content }}</p>
            <p class="date">
              Créé le: {{ new Date(post.attributes.creation_date).toLocaleString() }}
            </p>
            
            <div v-if="post.attributes.media?.length" class="media-gallery">
              <div v-for="(media, index) in post.attributes.media" :key="index" class="media-item">
                <img :src="media" :alt="'Media ' + (index + 1)" />
                <button @click="deleteMedia(post, index)" class="delete-media">
                  Supprimer
                </button>
              </div>
            </div>

            <div class="button-group">
              <button
                @click="deleteDocument(post)"
                class="delete-btn"
              >
                Supprimer
              </button>
              <button
                @click="startEditing(post)"
                class="edit-btn"
              >
                Modifier
              </button>
            </div>
          </div>

          <div v-else class="edit-form">
            <form @submit.prevent="updateDocument(editingPost)">
              <div>
                <label :for="'edit-title-' + post._id">Titre:</label>
                <input
                  :id="'edit-title-' + post._id"
                  v-model="editingPost.post_name"
                  required
                />
              </div>
              <div>
                <label :for="'edit-content-' + post._id">Contenu:</label>
                <textarea
                  :id="'edit-content-' + post._id"
                  v-model="editingPost.post_content"
                  required
                ></textarea>
              </div>
              <div>
                <label :for="'edit-media-' + post._id">Ajouter des médias:</label>
                <input
                  type="file"
                  :id="'edit-media-' + post._id"
                  multiple
                  @change="(e) => selectedFiles = e.target.files || []"
                />
              </div>
              <div class="button-group">
                <button type="submit">Enregistrer</button>
                <button type="button" @click="cancelEdit">Annuler</button>
              </div>
            </form>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.error-message {
  background-color: #ffebee;
  color: #c62828;
  padding: 10px;
  margin-bottom: 20px;
  border-radius: 4px;
  border: 1px solid #ef9a9a;
}

.loading {
  text-align: center;
  padding: 20px;
  color: #666;
}

.search-bar {
  margin-bottom: 20px;
}

.search-bar input {
  width: 100%;
  padding: 8px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.create-form, .edit-form {
  background-color: #1f1e1e;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  border: 1px solid #ddd;
}

.post-item {
  background-color: white;
  padding: 20px;
  margin-bottom: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.media-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 10px;
  margin: 10px 0;
}

.media-item {
  position: relative;
}

.media-item img {
  width: 100%;
  height: 150px;
  object-fit: cover;
  border-radius: 4px;
}

.delete-media {
  position: absolute;
  top: 5px;
  right: 5px;
  background: rgba(255,0,0,0.7);
  color: white;
  border: none;
  border-radius: 4px;
  padding: 4px 8px;
  cursor: pointer;
}

.button-group {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.delete-btn {
  background-color: #ff4444;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

.edit-btn {
  background-color: #44aaff;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

input, textarea {
  width: 100%;
  padding: 8px;
  margin-bottom: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

textarea {
  min-height: 100px;
  resize: vertical;
}

.date {
  color: #666;
  font-size: 0.9em;
}
</style>