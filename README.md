

# Pasos para Usar Algolia API

# 1. Crea un Proyecto en Algolia

  <img width="1148" height="728" alt="image" src="https://github.com/user-attachments/assets/04477d35-01e0-4f99-8286-de8712caa70b" />

# 2. Con las claves completa este momando con las claves y ejecutalo dentro de functions: 

    firebase functions:config:set algolia.app_id="TU_ALGOLIA_APP_ID" algolia.admin_key="TU_ALGOLIA_ADMIN_KEY"

# 3. Agrega este objeto de configuracion en environmets: 

      export const environment = {
      production: false,
        algolia: {
          appId: 'TU_ALGOLIA_APP_ID',
          searchOnlyApiKey: 'TU_CLAVE_DE_BUSQUEDA_DE_ALGOLIA'
        }
    };



    Paso 2: Sincronización del Backend (Cloud Function)
Ahora, vamos a crear la función que se encargará de sincronizar los cambios de Firestore con Algolia de forma automática.

Instala las dependencias necesarias en tu carpeta de functions:

Bash

    npm install algoliasearch

Crea la Cloud Function. En tu archivo functions/src/index.ts, añade el siguiente código. Este código se activa cada vez que un documento en tu colección de Firestore es creado, actualizado o eliminado.

TypeScript

    import * as functions from 'firebase-functions';
    import * as admin from 'firebase-admin';
    import algoliasearch from 'algoliasearch';
    
    admin.initializeApp();
    
    const ALGOLIA_ID = functions.config().algolia.app_id;
    const ALGOLIA_ADMIN_KEY = functions.config().algolia.admin_key;
    const INDEX_NAME = 'products'; // Reemplaza con el nombre de tu colección
    
    const algoliaClient = algoliasearch(ALGOLIA_ID, ALGOLIA_ADMIN_KEY);
    
    export const onDocumentChange = functions.firestore
      .document(`${INDEX_NAME}/{docId}`)
      .onWrite(async (change, context) => {
        const docId = context.params.docId;
        const document = change.after.data();
    
        if (!change.after.exists) {
          await algoliaClient.initIndex(INDEX_NAME).deleteObject(docId);
          return;
        }
    
        const objectID = { ...document, objectID: docId };
        await algoliaClient.initIndex(INDEX_NAME).saveObject(objectID);
      });

      
# Despliega la función: Abre tu terminal en la carpeta functions y ejecuta:

Bash

    firebase deploy --only functions

Esto subirá tu función a Firebase y se encargará de sincronizar los datos de forma automática.

# Paso 3: Servicio de Búsqueda en el Frontend
Finalmente, crea un servicio en tu aplicación Angular para realizar las búsquedas.

Instala Algolia en tu proyecto Angular:

Bash

    npm install algoliasearch/lite

Crea el servicio AlgoliaService:

TypeScript

    import { Injectable, signal, computed, WritableSignal } from '@angular/core';
    import algoliasearch, { SearchClient, SearchIndex } from 'algoliasearch/lite';
    import { environment } from '../../environments/environment';
    
    export type SearchIndexName = string;
    export type SearchQuery = string;
    
    @Injectable({
      providedIn: 'root'
    })
    export class AlgoliaService<T> {
    
      private algoliaClient: SearchClient;
      private algoliaIndex!: SearchIndex;
    
      private _searchResults: WritableSignal<T[]> = signal([]);
      public searchResults = computed(() => this._searchResults());
    
      private _loading: WritableSignal<boolean> = signal(false);
      public loading = computed(() => this._loading());
    
      constructor() {
        this.algoliaClient = algoliasearch(
          environment.algolia.appId,
          environment.algolia.searchOnlyApiKey
        );
      }
    
      public buscar(indexName: SearchIndexName, query: SearchQuery): void {
        if (!query) {
          this._searchResults.set([]);
          return;
        }
    
        this._loading.set(true);
        this.algoliaIndex = this.algoliaClient.initIndex(indexName);
    
        this.algoliaIndex.search<T>(query)
          .then((response) => {
            this._searchResults.set(response.hits as T[]);
            this._loading.set(false);
          })
          .catch((error) => {
            console.error('Error en la búsqueda de Algolia:', error);
            this._loading.set(false);
            this._searchResults.set([]);
          });
      }
    }
    
Con estos pasos manuales, tendrás un buscador de alta calidad completamente integrado en tu proyecto Lidertech.
