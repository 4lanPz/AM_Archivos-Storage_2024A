# CV usando Ionic

Hacer un página que permita subir archivos jpg utilizando Ionic, Visual Studio Code y Android Studio

## Pasos

- 1 Pre requisitos

Tener instalado Node.JS y Npm
Tener un IDE para customizar nuestro proyecto en este caso Visual Studio Code
Tener una cuenta de Firebase y sus credenciales
Tener Android Studio configurado
Estos se pueden descargar desde la pagina web oficial de dependiendo de el OS que estes utilizando.

- 2 Empezar el proyecto

Para empezar el proyecto hay que ejecutar el siguiente comando

```bash
ionic Start nombreproyecto blank --type=angular
```

En este nombreproyecto es el nombre que le vamos a poner a nuestro proyecto, por lo que se puede poner el
que tu quieras para tu proyecto.

En este tambien debemos elegir que módulos vamos a ocupar, en este caso vamos a ocupar "NGModules"

Al finalizar no es necesario tener una cuenta de Ionic, asi que eso podemos indicar que no y con eso nuestro
proyecto se ha creado

- 3 Navegar al directorio e intalación dependencias
Utilizando la consola CMD podemos ir a el directorio de nuestro proyecto con 
```bash
cd nombreproyecto
```
Dentro de esta carpeta tendremos que instalar los módulos necesarios para que se ejecute nuestro proyecto:

```bash
npm install
```
Después de instalar los módulos del proyecto, es necesario instalar firebase authentication, ya que este nos dara los servicios necesarios para poder generar la logica del login y registro
Para ello necesitamos ejecutar el siguiente comando
```bash
npm install @angular/fire firebase@9.16.0 --legacy-peer-deps
npm install @ionic-native/core --legacy-peer-deps
ionic g page home
```
una configuracion en las reglas de nuestro Firebase Storage para que nos permita ingresar archivos aun sin estar autenticados es:
```bash
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null;
    }
  }
}
```
Estos lo que nos ayudan es a configurar nuestro Firebase para permitirnos guardar datos, en este caso imágenes no pasadas de formato jpg.
Cuando generemos nuestro apikey nos debe entregar algo como esto
```bash
firebaseConfig: {
    apiKey: "TU_API_KEY",
    authDomain: "TU_AUTH_DOMAIN",
    projectId: "TU_PROJECT_ID",
    storageBucket: "TU_STORAGE_BUCKET",
    messagingSenderId: "TU_MESSAGING_SENDER_ID",
    appId: "TU_APP_ID",
    measurementId: "TU_MEASUREMENT_ID"
  }
```
- 4 Funcionalidad
Para empezar a generar el código necesitamos realizar una importación de los modulos de Firestorage
```bash
import { AngularFireModule } from '@angular/fire/compat';
import { AngularFireStorage, AngularFireStorageModule } from '@angular/fire/compat/storage';
import { AngularFirestoreModule } from '@angular/fire/compat/firestore';
import { environment } from '../environments/environment';
@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule, 
    IonicModule.forRoot(), 
    AppRoutingModule, 
    AngularFireModule.initializeApp(environment.firebaseConfig), 
    AngularFirestoreModule, 
    AngularFireStorageModule
  ],
  providers: [{ provide: RouteReuseStrategy, useClass: IonicRouteStrategy }],
  bootstrap: [AppComponent],
})
export class AppModule {}
```
Pero al poner ese código necesitamos tambien poner nuestras credenciales en nuestro archivo environment.ts
Ahora con los módulos importados podemos empezar a realizar la lógica del login y que este nos redireccione a una página home

Para poder subir archivos necesitaremos un archivo en el cual esté la logica de que archivos vamos a subir y de que tamaño son admitidos.
En este caso dentro de la carpeta Home vamos a crear un archivo "format-file-size.pipe.ts" en este se implementa el un cambio de formato de las imagenes para redondearlo a 2 decimales 
```bash
nombreproyecto/
├── node_modules/
├── src/   <----------
│   ├── app/ <----------
│   │   ├── tab1/ <----------
│   │   ├── home/ <----------
│   │   │   ├── format-file-size.pipe.ts/    crear<----------
```
Este contará con la logica de los archivos, como el formato y el peso.
```bash
import {Pipe, PipeTransform} from '@angular/core';
const FILE_SIZE_UNITS = ['B', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];
const FILE_SIZE_UNITS_LONG = ['Bytes', 'Kilobytes', 'Megabytes', 'Gigabytes', 'Pettabytes', 'Exabytes', 'Zettabytes', 'Yottabytes'];
@Pipe({
  name: 'formatFileSize'
})
export class FormatFileSizePipe implements PipeTransform {
  transform(sizeInBytes: number, longForm: boolean): string {
    const units = longForm
      ? FILE_SIZE_UNITS_LONG
      : FILE_SIZE_UNITS;
    let power = Math.round(Math.log(sizeInBytes)/Math.log(1024));
  	power = Math.min(power, units.length - 1);
  	const size = sizeInBytes / Math.pow(1024, power); // size in new units
  	const formattedSize = Math.round(size * 100) / 100; // keep up to 2 decimals
  	const unit = units[power];
  	return `${formattedSize} ${unit}`;
  }
}
```
Ahora podemos ir a nuestro archivo "home.modules.ts" en el cual agregaremos la importación del archivo que hemos creado anteriormente para nuestra página home.
```bash
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonicModule } from '@ionic/angular';
import { FormsModule } from '@angular/forms';
import { HomePage } from './home.page';
import { HomePageRoutingModule } from './home-routing.module';
import { FormatFileSizePipe } from './format-file-size.pipe';
@NgModule({
  imports: [
    CommonModule,
    FormsModule,
    IonicModule,
    HomePageRoutingModule
  ],
  declarations: [
    HomePage,
    FormatFileSizePipe
  ]
})
export class HomePageModule {}
```
En este momento necesitamos agregar la lógica de nuestra página home para que permita subir archivos a firestorage y como se van a almacenar estos
```bash
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { finalize, tap } from 'rxjs/operators';
import { AngularFireStorage, AngularFireUploadTask} from '@angular/fire/compat/storage';
import { AngularFirestore, AngularFirestoreCollection } from '@angular/fire/compat/firestore';

export interface imgFile {
  name: string;
  filepath: string;
  size: number;
}

@Component({
  selector: 'app-home',
  templateUrl: 'home.page.html',
  styleUrls: ['home.page.scss'],
})

export class HomePage {
  fileUploadTask: AngularFireUploadTask;
  percentageVal: Observable<any>;
  trackSnapshot: Observable<any>;
  UploadedImageURL: Observable<string>;
  files: Observable<imgFile[]>;
  imgName: string;
  imgSize: number;
  isFileUploading: boolean;
  isFileUploaded: boolean;
  private filesCollection: AngularFirestoreCollection<imgFile>;
  constructor(
    private afs: AngularFirestore,
    private afStorage: AngularFireStorage
  ) {
    this.isFileUploading = false;
    this.isFileUploaded = false;
    // Define uploaded files collection
    this.filesCollection = afs.collection<imgFile>('imagesCollection');
    this.files = this.filesCollection.valueChanges();
  }
  uploadImage(event: FileList) {
    const file: any = event.item(0);
    // Image validation
    if (file.type.split('/')[0] !== 'image') {
      console.log('File type is not supported!');
      return;
    }
    this.isFileUploading = true;
    this.isFileUploaded = false;
    this.imgName = file.name;
    // Direccion de almacenamiento y nombre archivo
    const fileStoragePath = `Carpeta/Alan`;
    const imageRef = this.afStorage.ref(fileStoragePath);
    this.fileUploadTask = this.afStorage.upload(fileStoragePath, file);
    this.percentageVal = this.fileUploadTask.percentageChanges();
    this.trackSnapshot = this.fileUploadTask.snapshotChanges().pipe(
      finalize(() => {
        this.UploadedImageURL = imageRef.getDownloadURL();
        this.UploadedImageURL.subscribe(
          (resp) => {
            this.storeFilesFirebase({
              name: file.name,
              filepath: resp,
              size: this.imgSize,
            });
            this.isFileUploading = false;
            this.isFileUploaded = true;
          },
          (error) => {
            console.log(error);
          }
        );
      }),
      tap((snap: any) => {
        this.imgSize = snap.totalBytes;
      })
    );
  }
  storeFilesFirebase(image: imgFile) {
    const fileId = this.afs.createId();
    this.filesCollection
      .doc(fileId)
      .set(image)
      .then((res) => {
        console.log(res);
      })
      .catch((err) => {
        console.log(err);
      });
  }
}
```
Este código hay una parte muy importate
```bash
const fileStoragePath = `Carpeta/Alan`;
```
Esta parte nos indica como va a almacenarse el archivo que vayamos a subir, en este caso "Carpeta" es una carpeta que se va a crear para almacenar la imagen y "Alan" es como se va a llamar el archivo al subir a firebase

Ahora ya con esa lógica podemos pasar a hacer el html que va a contar con los espacios necesarios para poder subir los archivos, ás una barra de progreso cuando se suban los archivos.
```bash
<ion-header [translucent]="true">
  <ion-toolbar>
    <ion-title> Ionic Firebase Demo subir Archivos </ion-title>
  </ion-toolbar>
</ion-header>
<ion-content>
  <ion-card class="ion-text-center" *ngIf="!isFileUploading && !isFileUploaded">
    <ion-card-header>
      <ion-card-title>Elegir imagen</ion-card-title>
    </ion-card-header>
    <ion-card-content>
      <ion-button color="primary" size="medium">
        <input type="file" (change)="uploadImage($event.target.files)" />
      </ion-button>
    </ion-card-content>
  </ion-card>
  <!-- File upload progress bar -->
  <div *ngIf="percentageVal | async as percentage">
    Progress: {{ percentage | number }}%
    <ion-progress-bar value="{{ percentage / 100 }}"></ion-progress-bar>
  </div>
  <div *ngIf="trackSnapshot | async as snap">
    File size: {{ snap.totalBytes | formatFileSize }} Data transfered: {{
    snap.bytesTransferred | formatFileSize }}
  </div>
</ion-content>
```
- 5 Ejecución
Para poder probar nuestro proyecto y ver los cambios que hemos hecho a nuestro proyecto se debe ejecutar
```bash
npx ionic start 
```
ahora el programa empezará a generar nuestro proyecto para el sistema en el que estamos ejecutando, en este caso
Windows.
Para poder revisarlo en android es necesario tener un programa que pueda generar el paquete APK
En este caso Android Studio.
Para esto primero necesitaremos hacer un build para android por lo que ejecutamos:
Un problema que suele ocurrir al momento de generar el build de Android es que no suele encontrar las credenciales de Firebase, para solucionar esto hay que pasar las credenciales de enviroment.ts y copiarlas tambien en enviroment.prod.ts
```bash
npx ionic build android
```
Al ejecutar ese código empezará a generar los archivos necesarios para que el mismo proyecto que vimos en web se
pueda ver en Android.
Luego de tener generado el build para android se debe ejecutar el comando

```bash
npx ionic capacitor open android
```
Con esto se abrirá Android Studio y si ya tenemos un dispositivo configurado, podremos ver como se ve nuestro
proyecto en android

## Capturas
### Web
![image](https://github.com/4lanPz/AM_Archivos-Storage_2024A/assets/117743495/c5b0fd09-31ed-458e-bec2-f09b6d58c3fa)

![image](https://github.com/4lanPz/AM_Archivos-Storage_2024A/assets/117743495/9fe4a0db-100b-42d0-891d-6cdbb7d019a7)

![image](https://github.com/4lanPz/AM_Archivos-Storage_2024A/assets/117743495/f63f1c32-685d-4902-a473-9d784b4257d1)


### Android
Error por dependencias que ya no tienen soporte y posible cambio a las nuevas librerias "npm install @awesome-cordova-plugins" no existe un plugin que lo reemplace



