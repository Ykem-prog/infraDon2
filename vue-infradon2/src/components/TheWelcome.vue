<script setup lang="ts">

import { ref, onMounted } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';

PouchDB.plugin(PouchDBFind);


interface Post {
  _conflicts: any | null; 
  _id: string;
  _rev: string;
  title: string;
  content: string;
  attributes: {
    creation_date: any;
  };
}

const storage = ref<PouchDB.Database | null>(null);
const postsData = ref<Post[]>([]);
const sync = ref<PouchDB.Replication.Replication<Post> | null>(null);
const isOnline = ref(false);

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
});

let counter = 0; 
const words = [
    'léa', 'inoé', 'camilo', 'teicir', 'yannis', 'elia', 'loann', 'sasita',
    'sarah', 'enya', 'gabriel', 'nuno', 'tanguy', 'loic', 'dylan', 'liliana',
    'thierry', 'valentin', 'benoît', 'chloé',
];

const initDatabase = async () => {
    console.log('=> Connexion à la base de données');
    const localdb = new PouchDB('collection_infradon2');
    
    if (localdb) {
        console.log('Connected to collection : ' + localdb.name);
        storage.value = localdb;
        
        try {
            await storage.value.createIndex({
                index: {
                    fields: ['content'],
                },
            });
            console.log('Index créé');
        } catch (error) {
            console.error('Erreur lors de la création de l\'index', error);
        }

        
        try {
           
            await localdb.replicate.from('http://ykem:Salutpoilu99$@localhost:5984/infradon2');
            console.log('Réplication initiale terminée.');
        } catch (error) {
            console.warn('Erreur lors de la réplication initiale (CouchDB est-il démarré ?)', (error as Error).message);
        } finally {
      
            syncData();
            fetchData();
        }
    } else {
        console.warn('Something went wrong');
    }
};

const syncData = () => {
    if (storage.value) {
        console.log('Lancement de la synchronisation live...');
        isOnline.value = true;
        
        
        if (sync.value) sync.value.cancel();

        sync.value = storage.value
            .sync('http://ykem:Salutpoilu99$@localhost:5984/infradon2', { live: true, retry: true })
            .on('change', fetchData)
            .on('error', (err) => {
                console.error("Erreur de synchronisation", err);
                isOnline.value = false;
            })
            .on('denied', (err) => {
                console.error("Accès à CouchDB refusé", err);
                isOnline.value = false;
            })
           
            .on('paused', () => { 
               
                isOnline.value = true;
            })
            .on('active', () => {
                isOnline.value = true;
            })
            .on('complete', (info) => {
                console.log('Synchronisation interrompue par la fin de la source.', info);
                isOnline.value = false;
            });
    }
};

const toggle = () => {
    if (sync.value) {
        console.log('Synchronisation arrêtée');
        sync.value.cancel();
        sync.value = null;
        isOnline.value = false;
    } else {
        syncData();
    }
};

const search = async (event: Event) => {
   
    const query = (event.target as HTMLInputElement).value;
    (event.target as HTMLInputElement).blur();

    if (query === '') {
        fetchData();
    } else if (storage.value) {
        try {
            const result = await storage.value.find({
                selector: {
                    
                    content: { '$regex': `.*${query}.*` }
                },
                fields: ['_id', '_rev', 'title', 'content', '_conflicts']
            });
            
            console.log('=> Données filtrées :', result.docs);
            postsData.value = result.docs as Post[];
        } catch (error) {
            console.error('Erreur lors de la récupération des données :', error);
        }
    }
};

const searchReset = () => {
    
    const searchInput = document.querySelector('.search') as HTMLInputElement;
    if (searchInput) {
        searchInput.value = '';
    }
    fetchData();
};

const fetchData = async () => {
    if (!storage.value) return; 

    try {
        const result = await storage.value.allDocs({
            include_docs: true,
            conflicts: true,
        });
        
        console.log('=> Données récupérées (allDocs) :', result.rows);
        postsData.value = result.rows.map((row: PouchDB.Core.GetResult<Post>) => row.doc) as Post[];
    } catch (error) {
        console.error('Erreur lors de la récupération des données :', error);
    }
};

const createDoc = async () => {
    if (!storage.value) return;
    counter++;
    try {
        const response = await storage.value.post({
            title: 'Document ' + counter,
            content: 'Contenu du document : ' + words[Math.floor(Math.random() * words.length)],
            attributes: {
                creation_date: new Date().toISOString()
            }
        });
        fetchData();
        console.log('Document créé', response);
    } catch (err) {
        console.log('Erreur création document', err);
    }
};

const deleteDoc = async (post: Post) => {
    if (!storage.value) return;
    try {
        const response = await storage.value.remove(post);
        fetchData();
        console.log('Document supprimé', response);
    } catch (err) {
        console.log('Erreur suppression document', err);
    }
};

const updateDoc = async (post: Post) => {
    if (!storage.value) return;
    const newContent = 'Contenu du document : ' + words[Math.floor(Math.random() * words.length)] + ' (modifié)';
    const updatedPost = {
        ...post,
        content: newContent,
        _rev: post._rev 
    };
    
    try {
        const response = await storage.value.put(updatedPost);
      
        fetchData(); 
        console.log('Document mis à jour', response);
    } catch (err) {
        if ((err as PouchDB.Core.Error).name === 'conflict') {
             console.error('Erreur de conflit lors de la mise à jour. Récupération des dernières données...', err);
             fetchData(); 
        } else {
            console.log('Erreur de mise à jour', err);
        }
    }
};

</script>

<template>
    <div id="app-container">
        <h1>Gestion des Documents (PouchDB Sync)</h1>
        
        <!-- Toggle et statut -->
        <div class="header-actions">
            <div class="status-toggle">
                <label class="switch">
                    <input type="checkbox" :checked="isOnline" @click="toggle" /><span class="slider round"></span>
                </label>
                <label :class="isOnline ? 'status-online' : 'status-offline'">
                    {{ isOnline ? 'Online (Synchro Active)' : 'Offline (Synchro Arrêtée)' }}
                </label>
            </div>
            
            <button @click="createDoc" class="btn-primary">Ajouter un document</button>
        </div>
        
        <!-- Recherche -->
        <div class="search-container">
            <input type="text" placeholder="Rechercher par contenu (Entrée)" @keyup.enter="search" class="search" />
            <button @click="searchReset" class="btn-reset">X</button>
        </div>
        
        <!-- Liste des documents -->
        <div class="document-list">
            <article v-for="post in postsData" :key="post._id">
                <div class="article-content">
                    <h2>
                        {{ post.title }}
                        <span class="conflicts" v-if="post._conflicts">Attention, conflits !</span>
                    </h2>
                    <p>{{ post.content }}</p>
                </div>
                
                <div class="article-actions">
                    <button @click="updateDoc(post)" class="btn-update">Modifier</button>
                    <button @click="deleteDoc(post)" class="btn-delete">Effacer</button>
                </div>
            </article>
            <p v-if="postsData.length === 0" class="empty-message">Aucun document trouvé.</p>
        </div>
    </div>
</template>

<style scoped>
/* Conteneur principal */
#app-container {
    max-width: 900px;
    margin: 0 auto;
    background-color: #ffffff;
    padding: 2.5rem;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
    border-radius: 1rem;
    font-family: 'Inter', sans-serif;
}

/* En-tête */
h1 {
    font-size: 2.25rem;
    font-weight: 800;
    color: #1f2937;
    border-bottom: 2px solid #e5e7eb;
    padding-bottom: 0.5rem;
    margin-bottom: 1.5rem;
}

/* Actions du header (Toggle et Ajouter) */
.header-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1.5rem;
    gap: 1rem;
    flex-wrap: wrap;
}

/* Toggle et statut */
.status-toggle {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 1rem;
    border-radius: 0.75rem;
    background-color: #f3f4f6;
    box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
}
.status-online {
    color: #059669; /* Green */
    /* background-color: #d1fae5; */
    font-weight: 600;
}
.status-offline {
    color: #dc2626; /* Red */
    /* background-color: #fee2e2; */
    font-weight: 600;
}

/* Recherche */
.search-container {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    margin-bottom: 1.5rem;
}
.search {
    flex-grow: 1;
    padding: 0.75rem 1rem;
    border: 2px solid #d1d5db;
    border-radius: 0.5rem;
    transition: all 0.2s;
}
.search:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.2);
}

/* Boutons génériques */
button {
    cursor: pointer;
    border: none;
    border-radius: 0.5rem;
    transition: all 0.2s ease-in-out;
    font-weight: 600;
    font-size: 0.95rem;
}
.btn-primary {
    background-color: #2563eb;
    color: white;
    padding: 0.75rem 1.5rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}
.btn-primary:hover {
    background-color: #1d4ed8;
    box-shadow: 0 6px 10px -1px rgba(0, 0, 0, 0.15);
}
.btn-reset {
    background-color: #e5e7eb;
    color: #4b5563;
    padding: 0.75rem;
    width: 40px;
    height: 40px;
    line-height: 1;
    font-size: 1rem; /* Rendre le X plus visible */
}
.btn-reset:hover {
    background-color: #d1d5db;
}


/* Liste des documents */
.document-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-top: 1rem;
}
article {
    background-color: #f9fafb;
    padding: 1.5rem;
    border-radius: 0.75rem;
    box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.05);
    border: 1px solid #e5e7eb;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
}

/* Rendre la carte responsive sur mobile */
/* @media (max-width: 600px) {
    article {
        flex-direction: column;
        align-items: flex-start;
    }
    .article-actions {
        margin-top: 1rem;
        width: 100%;
        justify-content: flex-start;
    }
} */

.article-content {
    flex-grow: 5;
}
article h2 {
    font-size: 1.125rem;
    font-weight: 700;
    color: #374151;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.25rem;
}
article p {
    color: #6b7280;
    font-style: italic;
    margin: 0;
}
.article-actions {
    display: flex;
    flex-shrink: 0;
    gap: 0.5rem;
}
.btn-update {
    background-color: #f59e0b;
    color: white;
    padding: 0.5rem 1rem;
}
.btn-update:hover {
    background-color: #d97706;
}
.btn-delete {
    background-color: #ef4444;
    color: white;
    padding: 0.5rem 1rem;
}
.btn-delete:hover {
    background-color: #dc2626;
}

/* Conflicts */
.conflicts {
    color: #b91c1c; 
    background-color: #fee2e2;
    padding: 0.25rem 0.5rem;
    border-radius: 9999px;
    font-weight: 500;
    font-style: normal;
    font-size: 0.75rem;
    animation: pulse 2s infinite;
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
}

/* Switch (molette) */
.switch {
    position: relative;
    display: inline-block;
    width: 44px;
    height: 24px;
    vertical-align: middle;
}
.switch input {
    opacity: 0;
    width: 0;
    height: 0;
}
.slider {
    position: absolute;
    cursor: pointer;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: #ccc;
    transition: 0.4s;
    border-radius: 24px;
}
.slider:before {
    position: absolute;
    content: '';
    height: 16px;
    width: 16px;
    left: 4px;
    bottom: 4px;
    background-color: white;
    transition: 0.4s;
    border-radius: 50%;
}
input:checked + .slider {
    background-color: #10b981;
}
input:focus + .slider {
    box-shadow: 0 0 0 3px rgba(16, 185, 129, 0.5);
}
input:checked + .slider:before {
    transform: translateX(20px);
}
.empty-message {
    text-align: center;
    padding: 2rem 0;
    color: #9ca3af;
    font-style: italic;
}
</style>