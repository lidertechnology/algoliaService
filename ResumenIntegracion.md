Resumen de la Integración
Aquí tienes un resumen de los pasos completos y numerados para integrar un buscador en tu proyecto Lidertech, siguiendo todas tus convenciones de desarrollo.

1. Backend (Cloud Functions)
Este es el corazón de la sincronización y la seguridad.

Configuración Segura: Almacena tus claves de API de Algolia (o Meilisearch) como variables de entorno de Firebase con el comando firebase functions:config:set para mantenerlas fuera de tu código fuente.

Instalación de Dependencias: Navega a la carpeta functions y ejecuta npm install firebase-admin algoliasearch. La primera biblioteca es para conectar con Firebase, y la segunda, para interactuar con tu servicio de búsqueda.

Función de Sincronización: En un archivo dedicado (functions/src/algolia/algolia.ts), crea una Cloud Function que se active automáticamente con cada cambio en Firestore. Esta función se encargará de crear, actualizar o eliminar documentos en tu índice de búsqueda.

Despliegue: Desde la carpeta functions, despliega tu función con el comando firebase deploy --only functions para que comience a trabajar en tiempo real.

2. Frontend (Angular)
Aquí está la lógica de búsqueda que usarás en tu aplicación.

Instalación de Dependencias: En la raíz de tu proyecto Angular, instala la versión ligera del cliente de Algolia con npm install algoliasearch/lite. Esto mantendrá tu aplicación optimizada y ligera.

Configuración de Entorno: Almacena la clave de API de búsqueda (la clave segura para el frontend) en tu archivo de entorno (src/environments/environment.ts) para que tu servicio pueda acceder a ella.

Servicio Genérico de Búsqueda: Crea un servicio (algolia.service.ts) que encapsule toda la lógica de búsqueda. Este servicio se comunicará con Algolia, manejará el estado (loading, results) con señales y proveerá un método buscar() reutilizable para cualquier componente.

Componente del Buscador: Crea un componente que inyecte este servicio. Este componente debe usar Angular Material para la interfaz de usuario, emplear MatDialog para el modal de búsqueda y utilizar señales para manejar el estado del buscador.



# Aquí están los códigos para la integración completa de tu buscador, ajustados a tus convenciones.

# 1. Backend: Cloud Function de Sincronización
Este código se encargará de mantener tu base de datos de Firestore y Algolia sincronizados en tiempo real.

Archivo: functions/src/algolia/algolia.ts

TypeScript

    import * as functions from 'firebase-functions';
    import * as admin from 'firebase-admin';
    import algoliasearch from 'algoliasearch';
    
    admin.initializeApp();
    
    const ALGOLIA_ID = functions.config().algolia.app_id;
    const ALGOLIA_ADMIN_KEY = functions.config().algolia.admin_key;
    const INDEX_NAME = 'products';
    
    const algoliaClient = algoliasearch(ALGOLIA_ID, ALGOLIA_ADMIN_KEY);
    
    export const sincronizarFirestoreConAlgolia = functions.firestore
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
  
Archivo: functions/src/index.ts

TypeScript

    import { sincronizarFirestoreConAlgolia } from './algolia/algolia';
    
    export { sincronizarFirestoreConAlgolia };

# 2. Frontend: Servicio Genérico de Búsqueda
Este servicio es el encargado de interactuar con Algolia desde tu aplicación Angular.

Archivo: src/app/algolia/algolia.service.ts

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

# 3. Componentes del Buscador (Botón y Modal)
Estos componentes te permiten interactuar con el servicio de búsqueda.

Archivo: src/app/search-button/search-button.component.ts

TypeScript

    import { Component, Input, OnDestroy, signal, WritableSignal } from '@angular/core';
    import { MatDialog } from '@angular/material/dialog';
    import { SearchModalComponent } from '../search-modal/search.modal.component';
    import { StatesEnum } from '../states/states.enum';
    
    @Component({
      selector: 'lidertech-search-button',
      standalone: true,
      imports: [],
      templateUrl: './search-button.component.html',
      styleUrl: './search-button.component.css'
    })
    export class SearchButtonComponent implements OnDestroy {
    
      @Input() conector: any;
    
      public states: WritableSignal<StatesEnum> = signal(StatesEnum.DEFAULT);
    
      constructor(private dialog: MatDialog) {}
    
      public abrirBuscador(): void {
        this.states.set(StatesEnum.LOADING);
        this.dialog.open(SearchModalComponent, {
          width: '90%',
          maxWidth: '500px'
        });
        this.states.set(StatesEnum.SUCCESS);
      }
    
      ngOnDestroy(): void {}
    }

# Archivo: src/app/search-button/search-button.component.html

HTML

    <section class="box-responsive">
      <button mat-icon-button (click)="abrirBuscador()" aria-label="Abrir buscador">
        <mat-icon>search</mat-icon>
      </button>
    </section>


# Archivo: src/app/search-modal/search.modal.component.ts

TypeScript

    import { Component, OnDestroy, signal, WritableSignal, inject } from '@angular/core';
    import { MatDialogRef } from '@angular/material/dialog';
    import { AlgoliaService } from '../algolia/algolia.service';
    import { StatesEnum } from '../states/states.enum';
    
    @Component({
      selector: 'lidertech-search-modal',
      standalone: true,
      imports: [],
      templateUrl: './search.modal.component.html',
      styleUrl: './search.modal.component.css'
    })
    export class SearchModalComponent implements OnDestroy {
    
      private algoliaService = inject(AlgoliaService);
      private dialogRef = inject(MatDialogRef<SearchModalComponent>);
    
      public states: WritableSignal<StatesEnum> = signal(StatesEnum.DEFAULT);
      public resultadosBusqueda = this.algoliaService.searchResults;
    
      public buscar(event: Event): void {
        this.states.set(StatesEnum.LOADING);
        const query = (event.target as HTMLInputElement).value;
        this.algoliaService.buscar('products', query);
      }
    
      public cerrar(): void {
        this.dialogRef.close();
      }
    
      ngOnDestroy(): void {}
    }




# Archivo: src/app/search-modal/search.modal.component.html

HTML

    <section class="box-responsive">
      <mat-dialog-content>
        <mat-form-field appearance="outline" class="box-responsive">
          <input matInput placeholder="Buscar..." (input)="buscar($event)">
          <button mat-icon-button matSuffix (click)="cerrar()">
            <mat-icon>close</mat-icon>
          </button>
        </mat-form-field>
        
        @if (resultadosBusqueda().length > 0) {
          @for (resultado of resultadosBusqueda(); track resultado.objectID) {
            <div class="resultado-item">
              <p>{{ resultado.title }}</p>
            </div>
          }
        } @else if (states() === StatesEnum.LOADING) {
          <div class="loading-state">
            <mat-spinner diameter="30"></mat-spinner>
          </div>
        } @else {
          <div class="no-results-state">
            <p>No se encontraron resultados.</p>
          </div>
        }
    
      </mat-dialog-content>
    </section>
