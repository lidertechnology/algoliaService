1. Configurar tu proyecto Angular 🛠️
Primero, actualiza la configuración de tu proyecto para que TypeScript reconozca el cliente de Algolia.

Abre el archivo tsconfig.json y agrega la propiedad "allowSyntheticDefaultImports": true dentro del objeto compilerOptions.

JSON

    {
      "compileOnSave": false,
      "compilerOptions": {
        "outDir": "./dist/out-tsc",
        "strict": true,
        "noImplicitOverride": true,
        "noPropertyAccessFromIndexSignature": true,
        "noImplicitReturns": true,
        "noFallthroughCasesInSwitch": true,
        "skipLibCheck": true,
        "isolatedModules": true,
        "esModuleInterop": true,
        "experimentalDecorators": true,
        "moduleResolution": "bundler",
        "importHelpers": true,
        "target": "ES2022",
        "module": "ES2022",
        "allowSyntheticDefaultImports": true
      },
      "angularCompilerOptions": {
        "enableI18nLegacyMessageIdFormat": false,
        "strictInjectionParameters": true,
        "strictInputAccessModifiers": true,
        "strictTemplates": true
      }
    }

    
# 2. Instalar el paquete de Algolia 📦
Instala la versión 5 del cliente de Algolia, que es la recomendada para la mayoría de los proyectos modernos y viene con soporte nativo para TypeScript.

Bash

    npm install algoliasearch@5

# 3. Crear el servicio de Algolia ⚙️
Ahora, crea el directorio algolia dentro de src/app y agrega tu AlgoliaService. Este servicio encapsula toda la lógica de búsqueda.

Archivo: src/app/algolia/algolia.service.ts

TypeScript

    import { inject, Injectable, signal, computed, WritableSignal } from '@angular/core';
    import algoliasearch from 'algoliasearch/lite';
    import { SearchClient, SearchIndex } from 'algoliasearch';
    import { environment } from '../../environments/environment';
    
    export interface SearchResult<T> {
      hits: T[];
      processingTimeMS: number;
      nbHits: number;
    }
    
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


# 4. Actualizar el archivo de entorno 🔑
Añade las claves de Algolia a tu archivo de entorno (environment.ts) para que el servicio pueda utilizarlas de manera segura.

Archivo: src/environments/environment.ts

TypeScript

    export const environment = {
      production: false,
      algolia: {
        appId: 'TU_ALGOLIA_APP_ID',
        searchOnlyApiKey: 'TU_ALGOLIA_SEARCH_API_KEY'
      }
    };
    Nota: Reemplaza TU_ALGOLIA_APP_ID y TU_ALGOLIA_SEARCH_API_KEY con las claves que obtendrás de tu cuenta de Algolia.

# 5. Reiniciar el servidor de desarrollo 🚀
Para que todos los cambios surtan efecto, reinicia tu servidor de desarrollo de Angular.

Bash

    ng serve
    
Con estos pasos, tendrás un servicio de Algolia completamente funcional y listo para ser utilizado en tus componentes de búsqueda.
