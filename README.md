# Zettai

Plataforma e-commerce especializada en hardware y componentes de PC con armador interactivo.

## 1. Arquitectura

### 1.1. Stack tecnológico

* **Plataforma:** .NET 10.0 con C# como runtime y lenguaje base unificado.
* **Frontend:** Blazor Web App con Static SSR para navegación y catálogo (optimizado para SEO), y WebAssembly (`InteractiveWebAssembly`) para el armador interactivo, carrito y panel de administración.
* **Backend:** ASP.NET Core con Minimal APIs para endpoints HTTP modulares y centralizados.
* **Base de datos:** PostgreSQL administrado en Supabase.
* **ORM:** Entity Framework Core con Npgsql y convención de nombres `snake_case`.
* **Autenticación:** Supabase Auth (JWT) sincronizado a la base de datos pública mediante triggers.
* **Almacenamiento:** Supabase Storage para persistencia de imágenes en bucket público.

### 1.2. Proyectos de la solución

* **`Zettai`:** Proyecto servidor ASP.NET Core que actúa como host principal, expone la API, implementa la lógica de servicios y gestiona el `DbContext`.
* **`Zettai.Client`:** Proyecto cliente Blazor WebAssembly que contiene las vistas interactivas, componentes UI y servicios HTTP de consumo.
* **`Zettai.Shared`:** Biblioteca de clases compartida con DTOs, interfaces, constantes y modelos comunes.

### 1.3. Modelo de atributos

El catálogo utiliza el patrón **Entity-Attribute-Value (EAV)** para admitir especificaciones técnicas variables según la categoría de hardware:

* **Atributo (`category_attributes`):** Define las especificaciones técnicas requeridas, tipos y filtros permitidos por categoría.
* **Valor (`product_attributes`):** Asigna el valor concreto a cada producto para su ficha técnica.
* **Compatibilidad (`builder_rules`):** Utiliza los atributos del modelo para validar reglas entre componentes en el armador de PC.

### 1.4. Autenticación y autorización

* **Credenciales vs. Perfil:** Las contraseñas, sesiones y autenticación residen exclusivamente en `auth.users` (gestionadas por Supabase). La tabla `public.users` almacena únicamente datos de perfil y negocio (sin contraseñas).
* **Registro (`/account/register`):** El cliente invoca el SDK de Supabase (`signUp`). Al crearse el usuario en `auth.users`, el trigger `on_auth_user_created` replica en el acto su fila en `public.users`.
* **Inicio de sesión (`/account/login`):** El cliente invoca el SDK de Supabase (`signInWithPassword`), que valida credenciales contra `auth.users` y emite un token JWT.
* **Persistencia de sesión:** El cliente almacena el token JWT y el token de refresco en `localStorage`, restaurando automáticamente el estado de autenticación al recargar la página.
* **Validación en API:** El cliente envía el JWT en la cabecera `Authorization: Bearer <token>`. ASP.NET Core lo valida mediante el middleware JWT Bearer (`JwtSecret`).
* **Identidad en backend:** Los endpoints obtienen el `id` del usuario autenticado a través del claim `sub` (`NameIdentifier`).
* **Roles:**
    * `customer` (rol por defecto asignado a todo usuario registrado).
    * `admin` (administrador).
* **Niveles de acceso (API):**
    * `public`: Acceso libre sin credenciales.
    * `authorized`: Requiere token JWT válido (usuarios con rol `customer` o `admin`).
    * `admin`: Requiere token JWT válido con rol `admin`.

### 1.5. Almacenamiento de archivos

* **Proveedor:** Supabase Storage para persistencia de activos multimedia.
* **Bucket público:** Bucket `products` configurado con acceso de lectura público.
* **Flujo de subida:** El cliente sube la imagen directamente al bucket mediante el SDK de Supabase, obtiene su URL pública y la envía a la API en el DTO del producto (`ProductInputDto`).

---

## 2. Base de datos

### 2.1. Tabla `users`

| Campo        | Tipo          | Restricciones                                             | Descripción                            |
| ------------ | ------------- | --------------------------------------------------------- | -------------------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE` | Identificador único                    |
| `email`      | `TEXT`        | `UNIQUE NOT NULL`                                         | Correo electrónico                     |
| `role`       | `TEXT`        | `NOT NULL DEFAULT 'customer'`                             | Rol del usuario (`customer` o `admin`) |
| `name`       | `TEXT`        | `NOT NULL`                                                | Nombre completo                        |
| `birth`      | `DATE`        | `NULL`                                                    | Fecha de nacimiento                    |
| `phone`      | `TEXT`        | `NULL`                                                    | Número de teléfono                     |
| `address`    | `TEXT`        | `NULL`                                                    | Dirección de envío                     |
| `is_active`  | `BOOLEAN`     | `NOT NULL DEFAULT true`                                   | Estado de activación                   |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`           | Fecha y hora de creación               |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`           | Fecha y hora de actualización          |

```sql
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer set search_path = ''
as $$
begin
    insert into public.users (
        id,
        email,
        role,
        name
    )
    values (
        new.id,
        new.email,
        coalesce(new.raw_app_meta_data->>'role', 'customer'),
        coalesce(new.raw_user_meta_data->>'name', '')
    );
    return new;
end;
$$;
revoke execute on function public.handle_new_user() from public;
revoke execute on function public.handle_new_user() from anon;
revoke execute on function public.handle_new_user() from authenticated;
drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
    after insert on auth.users
    for each row execute function public.handle_new_user();
```

### 2.2. Tabla `groups`

| Campo        | Tipo          | Restricciones                                   | Descripción                   |
| ------------ | ------------- | ----------------------------------------------- | ----------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`         | Identificador único           |
| `name`       | `TEXT`        | `UNIQUE NOT NULL`                               | Nombre del grupo              |
| `position`   | `INTEGER`     | `NOT NULL`                                      | Posición de ordenamiento      |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())` | Fecha y hora de creación      |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())` | Fecha y hora de actualización |

### 2.3. Tabla `categories`

| Campo        | Tipo          | Restricciones                                            | Descripción                   |
| ------------ | ------------- | -------------------------------------------------------- | ----------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`                  | Identificador único           |
| `group_id`   | `UUID`        | `NULL REFERENCES groups(id) ON DELETE SET NULL`          | Grupo al que pertenece        |
| `name`       | `TEXT`        | `UNIQUE NOT NULL`                                        | Nombre de la categoría        |
| `position`   | `INTEGER`     | `NOT NULL`                                               | Posición de ordenamiento      |
| `is_active`  | `BOOLEAN`     | `NOT NULL DEFAULT true`                                  | Estado de activación          |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`          | Fecha y hora de creación      |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`          | Fecha y hora de actualización |

### 2.4. Tabla `category_attributes`

| Campo           | Tipo          | Restricciones                                           | Descripción                            |
| --------------- | ------------- | ------------------------------------------------------- | -------------------------------------- |
| `id`            | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`                 | Identificador único                    |
| `category_id`   | `UUID`        | `NOT NULL REFERENCES categories(id) ON DELETE RESTRICT` | Categoría a la que pertenece           |
| `name`          | `TEXT`        | `NOT NULL`                                              | Nombre del atributo                    |
| `type`          | `TEXT`        | `NOT NULL DEFAULT 'text'`                               | Tipo de dato del atributo              |
| `is_required`   | `BOOLEAN`     | `NOT NULL DEFAULT false`                                | Indica si es obligatorio               |
| `is_filterable` | `BOOLEAN`     | `NOT NULL DEFAULT false`                                | Indica si permite filtrado en catálogo |
| `created_at`    | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`         | Fecha y hora de creación               |
| `updated_at`    | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`         | Fecha y hora de actualización          |

> Restricción de unicidad compuesta: `UNIQUE (category_id, name)`

### 2.5. Tabla `products`

| Campo         | Tipo             | Restricciones                                           | Descripción                     |
| ------------- | ---------------- | ------------------------------------------------------- | ------------------------------- |
| `id`          | `UUID`           | `PRIMARY KEY DEFAULT gen_random_uuid()`                 | Identificador único             |
| `category_id` | `UUID`           | `NOT NULL REFERENCES categories(id) ON DELETE RESTRICT` | Categoría a la que pertenece    |
| `name`        | `TEXT`           | `UNIQUE NOT NULL`                                       | Nombre del producto             |
| `slug`        | `TEXT`           | `UNIQUE NOT NULL`                                       | Identificador amigable para URL |
| `description` | `TEXT`           | `NULL`                                                  | Descripción del producto        |
| `price`       | `NUMERIC(12, 2)` | `NOT NULL CHECK (price >= 0)`                           | Precio unitario                 |
| `stock`       | `INTEGER`        | `NOT NULL CHECK (stock >= 0)`                           | Cantidad disponible en stock    |
| `is_active`   | `BOOLEAN`        | `NOT NULL DEFAULT true`                                 | Estado de activación            |
| `created_at`  | `TIMESTAMPTZ`    | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`         | Fecha y hora de creación        |
| `updated_at`  | `TIMESTAMPTZ`    | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`         | Fecha y hora de actualización   |

### 2.6. Tabla `product_attributes`

| Campo          | Tipo          | Restricciones                                                    | Descripción                   |
| -------------- | ------------- | ---------------------------------------------------------------- | ----------------------------- |
| `id`           | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`                          | Identificador único           |
| `product_id`   | `UUID`        | `NOT NULL REFERENCES products(id) ON DELETE CASCADE`             | Producto al que pertenece     |
| `attribute_id` | `UUID`        | `NOT NULL REFERENCES category_attributes(id) ON DELETE RESTRICT` | Atributo al que pertenece     |
| `value`        | `TEXT`        | `NOT NULL`                                                       | Valor asignado al atributo    |
| `created_at`   | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`                  | Fecha y hora de creación      |
| `updated_at`   | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`                  | Fecha y hora de actualización |

> Restricción de unicidad compuesta: `UNIQUE (product_id, attribute_id, value)`

### 2.7. Tabla `product_images`

| Campo        | Tipo          | Restricciones                                        | Descripción                     |
| ------------ | ------------- | ---------------------------------------------------- | ------------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`              | Identificador único             |
| `product_id` | `UUID`        | `NOT NULL REFERENCES products(id) ON DELETE CASCADE` | Producto al que pertenece       |
| `url`        | `TEXT`        | `NOT NULL`                                           | URL pública en Supabase Storage |
| `order`      | `INTEGER`     | `NOT NULL`                                           | Posición de ordenamiento        |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`      | Fecha y hora de creación        |

### 2.8. Tabla `orders`

| Campo        | Tipo             | Restricciones                                      | Descripción                   |
| ------------ | ---------------- | -------------------------------------------------- | ----------------------------- |
| `id`         | `UUID`           | `PRIMARY KEY DEFAULT gen_random_uuid()`            | Identificador único           |
| `user_id`    | `UUID`           | `NOT NULL REFERENCES users(id) ON DELETE RESTRICT` | Usuario que realiza el pedido |
| `status`     | `TEXT`           | `NOT NULL`                                         | Estado del pedido             |
| `total`      | `NUMERIC(12, 2)` | `NOT NULL CHECK (total >= 0)`                      | Monto total del pedido        |
| `created_at` | `TIMESTAMPTZ`    | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`    | Fecha y hora de creación      |
| `updated_at` | `TIMESTAMPTZ`    | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`    | Fecha y hora de actualización |

### 2.9. Tabla `order_products`

| Campo        | Tipo             | Restricciones                                      | Descripción                          |
| ------------ | ---------------- | -------------------------------------------------- | ------------------------------------ |
| `id`         | `UUID`           | `PRIMARY KEY DEFAULT gen_random_uuid()`            | Identificador único                  |
| `order_id`   | `UUID`           | `NOT NULL REFERENCES orders(id) ON DELETE CASCADE` | Pedido al que pertenece              |
| `product_id` | `UUID`           | `NULL REFERENCES products(id) ON DELETE SET NULL`  | Producto comprado                    |
| `name`       | `TEXT`           | `NOT NULL`                                         | Nombre histórico del producto        |
| `price`      | `NUMERIC(12, 2)` | `NOT NULL CHECK (price >= 0)`                      | Precio unitario al momento de compra |
| `units`      | `INTEGER`        | `NOT NULL CHECK (units > 0)`                       | Cantidad de unidades compradas       |

### 2.10. Tabla `builder_steps`

| Campo         | Tipo          | Restricciones                                                  | Descripción                |
| ------------- | ------------- | -------------------------------------------------------------- | -------------------------- |
| `id`          | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`                        | Identificador único        |
| `category_id` | `UUID`        | `UNIQUE NOT NULL REFERENCES categories(id) ON DELETE RESTRICT` | Categoría asignada al paso |
| `position`    | `INTEGER`     | `NOT NULL`                                                     | Posición de ordenamiento   |
| `created_at`  | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`                | Fecha y hora de creación   |

### 2.11. Tabla `builder_rules`

| Campo        | Tipo          | Restricciones                                                   | Descripción                        |
| ------------ | ------------- | --------------------------------------------------------------- | ---------------------------------- |
| `id`         | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`                         | Identificador único                |
| `target_id`  | `UUID`        | `NOT NULL REFERENCES category_attributes(id) ON DELETE CASCADE` | Atributo dependiente de la regla   |
| `source_id`  | `UUID`        | `NOT NULL REFERENCES category_attributes(id) ON DELETE CASCADE` | Atributo condicionante de la regla |
| `type`       | `TEXT`        | `NOT NULL`                                                      | Tipo de regla de compatibilidad    |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())`                 | Fecha y hora de creación           |

> Restricción de unicidad compuesta: `UNIQUE (target_id, source_id, type)`

### 2.12. Tabla `logs`

| Campo         | Tipo          | Restricciones                                   | Descripción                          |
| ------------- | ------------- | ----------------------------------------------- | ------------------------------------ |
| `id`          | `UUID`        | `PRIMARY KEY DEFAULT gen_random_uuid()`         | Identificador único                  |
| `user_id`     | `UUID`        | `NULL REFERENCES users(id) ON DELETE SET NULL`  | Usuario que ejecutó la acción        |
| `action`      | `TEXT`        | `NOT NULL`                                      | Acción realizada                     |
| `entity_name` | `TEXT`        | `NOT NULL`                                      | Nombre de la entidad afectada        |
| `entity_id`   | `UUID`        | `NULL`                                          | Identificador de la entidad afectada |
| `details`     | `TEXT`        | `NULL`                                          | Detalles adicionales del evento      |
| `ip_address`  | `TEXT`        | `NULL`                                          | Dirección IP de origen               |
| `created_at`  | `TIMESTAMPTZ` | `NOT NULL DEFAULT timezone('utc'::TEXT, now())` | Fecha y hora de creación             |

---

## 3. API

> Códigos HTTP:
>* `200 OK`
>* `201 Created`
>* `204 No Content`
>* `400 Bad Request`
>* `401 Unauthorized`
>* `403 Forbidden`
>* `404 Not Found`
>* `409 Conflict`

| Recurso           | Método   | Ruta (`/api`)                       | Cuerpo                | Acceso       | Respuesta                                          | Descripción                              |
| ----------------- | -------- | ----------------------------------- | --------------------- | ------------ | -------------------------------------------------- | ---------------------------------------- |
| **Users**         | `GET`    | `/users`                            | -                     | `admin`      | `200` \| `401` \| `403`                            | Lista todos los usuarios                 |
|                   |          | `/users/me`                         | -                     | `authorized` | `200` \| `401`                                     | Obtiene usuario actual                   |
|                   |          | `/users/{id}`                       | -                     | `admin`      | `200` \| `401` \| `403` \| `404`                   | Obtiene usuario por ID                   |
|                   | `PUT`    | `/users/me`                         | `UserInputDto`        | `authorized` | `204` \| `400` \| `401`                            | Actualiza usuario actual                 |
|                   |          | `/users/{id}`                       | `UserInputDto`        | `admin`      | `204` \| `400` \| `401` \| `403` \| `404`          | Actualiza usuario por ID                 |
|                   | `DELETE` | `/users/me`                         | -                     | `authorized` | `204` \| `401`                                     | Elimina usuario actual                   |
|                   |          | `/users/{id}`                       | -                     | `admin`      | `204` \| `401` \| `403` \| `404`                   | Elimina usuario por ID                   |
| **Groups**        | `GET`    | `/groups`                           | -                     | `public`     | `200`                                              | Lista todos los grupos de categorías     |
|                   |          | `/groups/{id}`                      | -                     | `public`     | `200` \| `404`                                     | Obtiene grupo por ID                     |
|                   | `POST`   | `/groups`                           | `GroupInputDto`       | `admin`      | `201` \| `400` \| `401` \| `403` \| `409`          | Crea grupo                               |
|                   | `PUT`    | `/groups/{id}`                      | `GroupInputDto`       | `admin`      | `204` \| `400` \| `401` \| `403` \| `404` \| `409` | Actualiza grupo                          |
|                   | `DELETE` | `/groups/{id}`                      | -                     | `admin`      | `204` \| `401` \| `403` \| `404`                   | Elimina grupo                            |
| **Categories**    | `GET`    | `/categories`                       | -                     | `public`     | `200`                                              | Lista todas las categorías de productos  |
|                   |          | `/categories?groupId={groupId}`     | -                     | `public`     | `200`                                              | Lista categorías de un grupo             |
|                   |          | `/categories/{id}`                  | -                     | `public`     | `200` \| `404`                                     | Obtiene categoría por ID                 |
|                   | `POST`   | `/categories`                       | `CategoryInputDto`    | `admin`      | `201` \| `400` \| `401` \| `403` \| `409`          | Crea categoría con atributos             |
|                   | `PUT`    | `/categories/{id}`                  | `CategoryInputDto`    | `admin`      | `204` \| `400` \| `401` \| `403` \| `404` \| `409` | Actualiza categoría y atributos          |
|                   | `DELETE` | `/categories/{id}`                  | -                     | `admin`      | `204` \| `401` \| `403` \| `404` \| `409`          | Elimina categoría                        |
| **Products**      | `GET`    | `/products`                         | -                     | `public`     | `200`                                              | Lista todos los productos                |
|                   |          | `/products?categoryId={categoryId}` | -                     | `public`     | `200`                                              | Lista productos de una categoría         |
|                   |          | `/products?slug={slug}`             | -                     | `public`     | `200` \| `404`                                     | Obtiene producto por slug                |
|                   |          | `/products/{id}`                    | -                     | `public`     | `200` \| `404`                                     | Obtiene producto por ID                  |
|                   | `POST`   | `/products`                         | `ProductInputDto`     | `admin`      | `201` \| `400` \| `401` \| `403` \| `409`          | Crea producto con atributos e imágenes   |
|                   | `PUT`    | `/products/{id}`                    | `ProductInputDto`     | `admin`      | `204` \| `400` \| `401` \| `403` \| `404` \| `409` | Actualiza producto, atributos e imágenes |
|                   | `DELETE` | `/products/{id}`                    | -                     | `admin`      | `204` \| `401` \| `403` \| `404`                   | Elimina producto                         |
| **Orders**        | `GET`    | `/orders`                           | -                     | `admin`      | `200` \| `401` \| `403`                            | Lista todos los pedidos                  |
|                   |          | `/orders/me`                        | -                     | `authorized` | `200` \| `401`                                     | Lista pedidos del usuario actual         |
|                   |          | `/orders?userId={userId}`           | -                     | `admin`      | `200` \| `401` \| `403`                            | Lista pedidos de un usuario              |
|                   |          | `/orders/{id}`                      | -                     | `authorized` | `200` \| `401` \| `403` \| `404`                   | Obtiene pedido por ID                    |
|                   | `POST`   | `/orders`                           | `OrderInputDto`       | `authorized` | `201` \| `400` \| `401`                            | Crea pedido con productos                |
|                   | `PUT`    | `/orders/{id}`                      | `OrderInputDto`       | `admin`      | `204` \| `400` \| `401` \| `403` \| `404`          | Actualiza estado del pedido              |
| **Builder Steps** | `GET`    | `/builder/steps`                    | -                     | `public`     | `200`                                              | Lista pasos del armador de PC            |
|                   |          | `/builder/steps/{id}`               | -                     | `public`     | `200` \| `404`                                     | Obtiene paso del armador por ID          |
|                   | `POST`   | `/builder/steps`                    | `BuilderStepInputDto` | `admin`      | `201` \| `400` \| `401` \| `403` \| `409`          | Crea paso del armador                    |
|                   | `PUT`    | `/builder/steps/{id}`               | `BuilderStepInputDto` | `admin`      | `204` \| `400` \| `401` \| `403` \| `404` \| `409` | Actualiza paso del armador               |
|                   | `DELETE` | `/builder/steps/{id}`               | -                     | `admin`      | `204` \| `401` \| `403` \| `404`                   | Elimina paso del armador                 |
| **Builder Rules** | `GET`    | `/builder/rules`                    | -                     | `public`     | `200`                                              | Lista reglas de compatibilidad           |
|                   |          | `/builder/rules/{id}`               | -                     | `public`     | `200` \| `404`                                     | Obtiene regla por ID                     |
|                   | `POST`   | `/builder/rules`                    | `BuilderRuleInputDto` | `admin`      | `201` \| `400` \| `401` \| `403` \| `409`          | Crea regla de compatibilidad             |
|                   | `DELETE` | `/builder/rules/{id}`               | -                     | `admin`      | `204` \| `401` \| `403` \| `404`                   | Elimina regla de compatibilidad          |
| **Logs**          | `GET`    | `/logs`                             | -                     | `admin`      | `200` \| `401` \| `403`                            | Lista registros de auditoría             |
|                   |          | `/logs/{id}`                        | -                     | `admin`      | `200` \| `401` \| `403` \| `404`                   | Obtiene registro por ID                  |

---

## 4. Frontend

### 4.1. Páginas

| Ruta                       | Componente                         | Proyecto   | Descripción                        |
| -------------------------- | ---------------------------------- | ---------- | ---------------------------------- |
| `/`                        | `Home.razor`                       | `Servidor` | Página principal                   |
| `/error`                   | `Error.razor`                      | `Servidor` | Error general (500)                |
| `/not-found`               | `NotFound.razor`                   | `Servidor` | Página no encontrada (404)         |
| `/products`                | `Products/ProductList.razor`       | `Servidor` | Catálogo de productos              |
| `/products/{slug}`         | `Products/ProductDetails.razor`    | `Servidor` | Detalle de producto                |
| `/builder`                 | `Builder.razor`                    | `Cliente`  | Armador interactivo de PC          |
| `/checkout`                | `Checkout.razor`                   | `Cliente`  | Finalización de compra y pago      |
| `/account`                 | `Account/AccountDetails.razor`     | `Cliente`  | Perfil del usuario autenticado     |
| `/account/register`        | `Account/Register.razor`           | `Cliente`  | Registro de nuevo usuario          |
| `/account/login`           | `Account/Login.razor`              | `Cliente`  | Inicio de sesión                   |
| `/account/verify-email`    | `Account/VerifyEmail.razor`        | `Cliente`  | Verificación de correo electrónico |
| `/account/update-password` | `Account/UpdatePassword.razor`     | `Cliente`  | Cambio de contraseña               |
| `/account/orders`          | `Account/Orders/OrderList.razor`   | `Cliente`  | Historial de pedidos del usuario   |
| `/admin`                   | `Admin/Dashboard.razor`            | `Cliente`  | Panel principal de administración  |
| `/admin/categories`        | `Admin/CategoryManagement.razor`   | `Cliente`  | Gestión de grupos y categorías     |
| `/admin/products`          | `Admin/ProductManagement.razor`    | `Cliente`  | Gestión de productos               |
| `/admin/orders`            | `Admin/OrderManagement.razor`      | `Cliente`  | Gestión de pedidos                 |
| `/admin/users`             | `Admin/UserManagement.razor`       | `Cliente`  | Gestión de usuarios                |
| `/admin/builder`           | `Admin/BuilderConfiguration.razor` | `Cliente`  | Configuración del armador de PC    |

### 4.2. Modo borrador

* Toda pantalla de gestión, edición o modal opera en memoria del cliente.
* **Aceptar:** Envía los cambios en lote al backend y los persiste en una única transacción atómica.
* **Cancelar:** Descarta los cambios en memoria sin alterar la base de datos.

### 4.3. Campo `position` de `groups` y `categories`

* **Nivel padre:** Compartido entre `groups` y `categories` independientes (`group_id = NULL`).
* **Nivel hijo:** Orden relativo interno de las categorías dentro de su grupo (`group_id != NULL`).
* **Arrastre (*drag & drop*):**
    * **Agrupar:** Asigna `group_id` del grupo destino y recalcula su `position` interna.
    * **Desagrupar:** Asigna `group_id = NULL` y recalcula su `position` en el nivel padre.
    * **Reordenar:** Desplaza elementos en el mismo nivel recalculando posiciones contiguas.
* **Eliminar grupo:** Las categorías hijas pasan a `group_id = NULL` y adoptan la posición del grupo eliminado en el nivel padre.

### 4.4. Configuración del armador de PC

* **Pasos (`builder_steps`):** Tarjetas arrastrables (*drag & drop*) para definir el orden secuencial del armador paso a paso mediante `position`.
* **Reglas (`builder_rules`):** Creación visual de compatibilidades arrastrando una conexión entre un atributo origen (`source_id`) y uno destino (`target_id`), asignando el `type` de validación (ej. `equals`).

---

## 5. Estructura de la solución

```
Zettai/
│
├── Zettai/                         # Proyecto servidor (ASP.NET Core)
│   ├── Components/                 # Componentes Razor del servidor
│   │   ├── Layout/...              # Diseños base SSR (MainLayout)
│   │   ├── Pages/...               # Páginas SSR (Home, Products, Error, etc.)
│   │   ├── App.razor               # Componente raíz y host HTML
│   │   └── Routes.razor            # Enrutador principal de la aplicación
│   ├── Data/                       # Capa de datos y persistencia
│   │   ├── Entities/...            # Entidades del modelo de datos
│   │   ├── Mappings/...            # Mapeos entre entidades y DTOs
│   │   ├── Migrations/...          # Migraciones de EF Core
│   │   └── AppDbContext.cs         # Contexto de base de datos
│   ├── Endpoints/...               # Definición de Minimal APIs
│   ├── Services/...                # Lógica de negocio del servidor
│   ├── wwwroot/...                 # Archivos estáticos globales (CSS, etc.)
│   ├── appsettings.json            # Configuración y conexión a base de datos
│   └── Program.cs                  # Punto de entrada y configuración del host
│
├── Zettai.Client/                  # Proyecto cliente (Blazor WebAssembly)
│   ├── Components/                 # Componentes Razor del cliente
│   │   ├── Pages/...               # Vistas interactivas (Builder, Admin, Account, etc.)
│   │   └── Shared/...              # Componentes de diseño, layout e interfaz
│   ├── Services/...                # Servicios HTTP de consumo de API
│   └── Program.cs                  # Punto de entrada y servicios cliente
│
└── Zettai.Shared/                  # Biblioteca compartida
    ├── Constants/...               # Rutas API, páginas, roles y constantes
    ├── Dtos/...                    # Objetos de transferencia de datos
    └── Interfaces/...              # Contratos e interfaces de servicios
```

---

## 6. Desarrollo local

### 6.1. Prerrequisitos

- [.NET 10.0 SDK](https://dotnet.microsoft.com/download/dotnet/10.0)
- Proyecto en [Supabase](https://supabase.com/) con base de datos PostgreSQL activa y un bucket público llamado `products` en Storage.
- Herramienta de Entity Framework Core (`dotnet-ef`):

    ```bash
    dotnet tool install --global dotnet-ef
    ```

### 6.2. Variables de configuración

**Servidor (`Zettai/appsettings.json` o `User Secrets`):**

```json
{
    "ConnectionStrings": {
        "DefaultConnection": "Host=...;Database=postgres;Username=...;Password=..."
    },
    "Supabase": {
        "Url": "https://TU_PROYECTO.supabase.co",
        "AnonKey": "TU_SUPABASE_ANON_KEY",
        "JwtSecret": "TU_SUPABASE_JWT_SECRET"
    }
}
```
