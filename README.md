# Implementación del consumo de una otra API y la manipulación de la misma.

Este proyecto implementa un sistema de inicio de sesión que consume una API externa para autenticar a los usuarios, el consumo de una nueva API de productos donde se podrá realizar la visualización de la información mas detallada, la edición y eliminación de un producto. A continuación, se describen las partes clave del código donde se realiza este consumo.
## API Externa Utilizada
- **URL usuarios**: https://api.escuelajs.co/api/v1/users
- **URL productos**: https://fakestoreapi.com/products

- **Descripción:** Proporciona una lista de usuarios en formato JSON que se utiliza para validar las credenciales.

## **1. Servicio `UserService`**

El servicio `UserService` se encarga de interactuar con la API externa para manejar las interacciones relacionadas con los usuarios, como obtener datos del API y gestionar el usuario actualmente logueado.

```typescript
@Injectable({
    providedIn: 'root'
  })
  export class UserService {
    private apiUrl = 'https://api.escuelajs.co/api/v1/users'; 
    private loggedInUser: any = null;

    constructor(private http: HttpClient,@Inject(PLATFORM_ID) private platformId: Object) {}
    

      getUsers() {
        return this.http.get<any[]>(this.apiUrl);
      }
      setLoggedInUser(user: any): void {
        if (isPlatformBrowser(this.platformId)) {
          this.loggedInUser = user;
          localStorage.setItem('loggedInUser', JSON.stringify(user)); 
        }
      }
    
      getLoggedInUser(): any {
        if (isPlatformBrowser(this.platformId)) {
          if (!this.loggedInUser) {
            this.loggedInUser = JSON.parse(localStorage.getItem('loggedInUser') || 'null');
          }
        }
        return this.loggedInUser;
      }
    }
```
	
- **getUsers:** Hace una solicitud HTTP para obtener la lista de usuarios desde el API (https://api.escuelajs.co/api/v1/users).
Retorna un observable con los datos de los usuarios.

- **setLoggedInUser(user):** Guarda el usuario logueado en una propiedad interna (loggedInUser) y también en el localStorage para persistirlo en el navegador.
Utiliza isPlatformBrowser para asegurarse de que está en un entorno de navegador (necesario para SSR con Angular Universal).

- **getLoggedInUser:** Devuelve el usuario logueado desde la memoria (loggedInUser) o, si no está disponible, lo obtiene del localStorage.

- ## Método iniciarSesion
- Valida las credenciales del usuario ingresadas en el formulario contra los datos obtenidos de la API.
```typescript
iniciarSesion() {
  const UsuarioExistente = this.user.find(
    (user) => user.email === this.username && user.password === this.password
  ); 
  if (UsuarioExistente) {
    console.log("Login exitoso");
    this.router.navigate(['/sidebar']);
  } else {
    console.log("Usuario no encontrado");
    this.snackBar.open('Usuario o contraseña incorrectos', 'Cerrar', {
      duration: 3000, 
      horizontalPosition: 'center',
      verticalPosition: 'top', 
    });
  }
  }
```
- **ngOnInit:** Llama a getUsers desde UserService para obtener todos los usuarios registrados.
- **iniciarSesion:** Busca en user un usuario cuyo email y password coincidan con los valores ingresados (username y password).
- **Si el usuario existe: **
 - Llama a setLoggedInUser en UserService para guardar la sesión.
 - Redirige al componente /sidebar.
- Si no, muestra un mensaje de error usando MatSnackBar.

## MenuComponent
Este componente muestra información del usuario logueado, como el nombre y la imagen de perfil.
```typescrypt
@Input() userImage: string | null = null; // Imagen del usuario logueado
  loggedInUser: any = null;
  
 ngOnInit(): void {
    this.loggedInUser = this.userService.getLoggedInUser();
    console.log('Usuario logueado:', this.loggedInUser); 
  }
```
- **loggedInUser**: Almacena el usuario actualmente logueado. Se obtiene al inicializar el componente (ngOnInit) desde UserService.
- **userImage:** Se usa para mostrar la imagen de perfil del usuario.

## SiderbarComponent
Este componente maneja la barra lateral de navegación y la funcionalidad de cerrar sesión.
```typescript
logout(): void {
    localStorage.removeItem('authToken');
    this.router.navigate(['/login']); 
  }
```
**logout:**
- Borra el token de autenticación almacenado en el localStorage.
- Redirige al usuario al componente de inicio de sesión (/login).

## ProductService
```typescript
@Injectable({
  providedIn: 'root'
})
export class ProductService {
  private apiUrl = 'https://fakestoreapi.com/products';

  constructor(private http: HttpClient) {}

  getProducts(): Observable<any> {
    return this.http.get<any>(this.apiUrl);
  }
  deleteProduct(productId: number): Observable<any> {
  return this.http.delete(`${this.apiUrl}/${productId}`);
}

}
```
- Proveer funciones para interactuar con una API REST.
- En este caso, la API (https://fakestoreapi.com/products) gestiona productos.
- getProducts(): Recupera la lista de productos desde la API.
- deleteProduct(productId: number): Elimina un producto específico usando su ID.
## ProductComponent
La plantilla usa datos obtenidos por el método getProducts() del servicio para mostrar una tabla con la lista de productos.
```typescript
loadProducts(): void {
    this.productService.getProducts().subscribe((data: any) => {
      this.products = data;
      this.filteredProducts = [...this.products]; 
      this.updatePaginatedProducts(); 
    });
  }
  ```
- En el método loadProducts(), se utiliza productService.getProducts() para obtener los productos desde la API y almacenarlos en products y filteredProducts.
- Los productos cargados se dividen en páginas mediante el método updatePaginatedProducts().
 ```javascript
 filterProducts(): void {
    console.log("Buscando productos:", this.searchQuery);  
    if (this.searchQuery.trim() === '') {
      this.filteredProducts = [...this.products]; 
    } else {
      this.filteredProducts = this.products.filter(product =>
        product.title.toLowerCase().includes(this.searchQuery.toLowerCase())
      );
    }
  
    console.log("Productos filtrados:", this.filteredProducts); 
  
    this.currentPage = 0; 
    this.updatePaginatedProducts(); 
  }
```
- Cuando el usuario escribe en el campo de búsqueda, el evento (input)="filterProducts()" llama al método filterProducts().
- Este método filtra products para encontrar aquellos cuyo título coincida con el texto ingresado en searchQuery.
- Después de filtrar, se reinicia la paginación (currentPage = 0) y se actualiza la tabla.
 ```javascript
updatePaginatedProducts(): void {
    const startIndex = this.currentPage * this.pageSize;
    const endIndex = startIndex + this.pageSize;
  
    console.log("Índice de inicio:", startIndex, "Índice de fin:", endIndex);
    console.log("Total productos filtrados:", this.filteredProducts.length);
  
    this.paginatedProducts = this.filteredProducts.slice(startIndex, endIndex);
  
    console.log("Productos paginados:", this.paginatedProducts);
  }
```
```javascript
onPageChange(event: PageEvent) {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.currentPage = event.pageIndex;
    this.updatePaginatedProducts();
  }
```
```html
<mat-paginator [pageSizeOptions]="[5, 10, 20]"
  showFirstLastButtons
  [length]="filteredProducts.length" 
  [pageSize]="pageSize"
  (page)="onPageChange($event)">
```
- El evento (page)="onPageChange($event)" detecta cambios en el paginador.
- El método onPageChange() actualiza pageIndex, pageSize y currentPage, y llama a updatePaginatedProducts() para mostrar los productos correspondientes a la nueva página.
```html
<td class="botoncontent">
            <button class="btn btn-info btn-sm button" (click)="viewProductInfo(product)">
              <mat-icon>info</mat-icon>
            </button>
            <button class="btn btn-warning btn-sm button" (click)="editProduct(product)">
              <mat-icon>edit</mat-icon>
            </button>
            <button mat-icon-button class="button" (click)="deleteProduct(product.id)">
              <mat-icon>delete</mat-icon>
            </button>
```
```javascript
viewProductInfo(product: any): void {
    const dialogRef = this.dialog.open(ProductDetallesComponent, {
      width: '650px',
      data: { product, mode: 'view' },
    });

    dialogRef.afterClosed().subscribe((result: any) => {
      if (result) {
        console.log('Acción realizada:', result);
      }
    });
  }
```
  
  ```javascript
editProduct(product: any): void {
    const dialogRef = this.dialog.open(ProductDetallesComponent, {
      width: '500px',
      data: { product, mode: 'edit' },
    });
  
    dialogRef.afterClosed().subscribe((updatedProduct: any) => {
      if (updatedProduct) {
        const index = this.products.findIndex((p) => p.id === updatedProduct.id);
        if (index !== -1) {
          this.products[index] = updatedProduct;
          this.updatePaginatedProducts();
        }
      }
    });
  }
```
  

 ```javascript
 deleteProduct(productId: number): void {
    if (confirm('¿Estás seguro de que deseas eliminar este producto?')) {
      this.productService.deleteProduct(productId).subscribe(
        () => {
          this.products = this.products.filter((p) => p.id !== productId);
          this.filteredProducts = this.filteredProducts.filter((p) => p.id !== productId);
          this.updatePaginatedProducts();
          this.cdr.detectChanges(); 
        },
        (error) => {
          console.error('Error eliminando el producto:', error);
        }
      );
    }
  }
```
- **Ver detalles:**
El botón con (click)="viewProductInfo(product)" abre un cuadro de diálogo (MatDialog) para mostrar detalles del producto seleccionado.
- **Editar producto:**
El botón con (click)="editProduct(product)" abre otro cuadro de diálogo con el modo de edición activado.
Si el producto se edita, se actualiza la lista de productos en products y se llama a updatePaginatedProducts().
- **Eliminar producto:**
El botón con (click)="deleteProduct(product.id)" elimina el producto al llamar a deleteProduct() en el componente, que utiliza el servicio ProductService para la operación
