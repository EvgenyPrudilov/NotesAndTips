```markdown
# Retrofit
Нужны такие зависимости:

```kotlin
implementation("com.squareup.retrofit2:retrofit:2.9.0")
implementation("com.squareup.retrofit2:converter-gson:2.9.0")
```

Т.к. мы подключили GSON конвертор, то он он сам будет преобразовывать JSON в наш DataClass:

```json
{
   "id": 1,
   "title": "iPhone 9",
   "description": "An apple mobile which is nothing like apple",
   "price": 549,
   "discountPercentage": 12.96,
   "rating": 4.69,
   "stock": 94,
   "brand": "Apple",
   "category": "smartphones",
   "thumbnail": "...",
   "images": ["...", "...", "..."]
}
```

в

```kotlin
data class Product (
    val id: Int,
    val title: String,
    val description: String,
    val price: Int,
    val discountPercentage: Double,
    val rating: Double,
    val stock: Int,
    val brand: String,
    val category: String,
    val thumbnail: String,
    val images: List<String>
)
```

Теперь нам надо прописать некоторый интерфейс, который будет описывать наш API для взаимодействия с сервером:

```kotlin
import retrofit2.http.GET

interface ProductAPI {
    @GET("products/1")
    suspend fun getProductById(): Product
}
```

Здесь мы указываем путь к объекту относительно адреса сервера: `@GET("product/1")`. Это будет `suspend` функция, т.е. будем её запускать в корутине. Нужно также добавить в манифест разрешение на выход в интернет.

Теперь создаём экземпляр ретрофита:

```kotlin
val retrofit = Retrofit.Builder()
    .baseUrl("https://dummyjson.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

, где:

- `baseUrl` - указываем имя сервера
- `addConverterFactory` - указываем конвертор, который будет преобразовывать наши данные - экземпляр `GsonConverterFactory`
- `build` - создать экземпляр

А вот теперь мы с помощью экземпляра указываем, ЧТО мы хотим создать - какой интерфейс:

```kotlin
val productApi = retrofit.create(ProductAPI::class.java)
```

После чего библиотека ретрофит создаст класс на основе этого интерфейса. Теперь можно им пользоваться:

```kotlin
productApi.getProductById()
```

Но делать это с главного потока нельзя, т.к. он будет тормозить. Поэтому мы будем использовать корутины на второстепенном потоке (например, `Dispatchers.IO`):

```kotlin
CoroutineScope(Dispatchers.IO)
```

Если мы хотим, чтобы можно было указать часть пути в функции GET запроса, то можно сделать так:

```kotlin
interface ProductAPI {
    @GET("products/{id}")
    suspend fun getProductById(@Path("id") id: Int): Product
}
```

Весь код:

```kotlin
package com.example.retrofit1

import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import com.example.retrofit1.retrofit.ProductAPI
import com.example.retrofit1.ui.theme.Retrofit1Theme
import kotlinx.coroutines.launch
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val retrofit = Retrofit.Builder()
            .baseUrl("https://dummyjson.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        
        val productApi = retrofit.create(ProductAPI::class.java)

        setContent {
            val coroutineScope = rememberCoroutineScope()
            Retrofit1Theme {
                // Создаем состояние для заголовка
                var title by remember { mutableStateOf("Loading...") }

                // A surface container using the 'background' color from the theme
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    Column {
                        Text(text = title)
                        Button(onClick = {
                            coroutineScope.launch {
                                try {
                                    val product = productApi.getProductById(3)
                                    title = product.title.orEmpty()
                                } catch (e: Exception) {
                                    title = "Failed to load"
                                    Log.e("MainActivity", "Error fetching product", e)
                                }
                            }
                        }) {
                            Text("Get Product")
                        }
                    }
                }
            }
        }
    }
}
```

Часто бывает такое, что при работе с retrofit что-то не работает, и при этом мы не знаем, какой запрос отправил ретрофит, или неправильно приняли данные. Короче надо видеть, какие запросы мы отправляем и что получаем. Мы можем сделать это с помощью библиотеки OkHttp:

```kotlin
implementation("com.squareup.okhttp3:okhttp:4.12.0")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
```

Сначала мы создаём интерсептор:

```kotlin
val interceptor = HttpLoggingInterceptor()
interceptor.level = HttpLoggingInterceptor.Level.BODY
```

в свойстве `level` мы будем указывать Что мы перехватываем - в данном случае - тело запроса. При создании экземпляра ретрофита мы можем в параметре `client` указать объект `OkHttpClient`, который будет заниматься логингом:

```kotlin
val client = OkHttpClient.Builder()
    .addInterceptor(interceptor)
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://dummyjson.com")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

Теперь в панели `Logcat` у нас будет выводиться логирование при отправке и получении данных по сети.

------------------------------------------------

Теперь поработаем с аутентификацией. Мы будем отправлять имя пользователя(username: 'kminchelle') и пароль(password: '0lelplR'), и если такой имеется, то нам выдадут результат:

```json
{
  "id": 15,
  "username": "kminchelle",
  "email": "kminchelle@qq.com",
  "firstName": "Jeanne",
  "lastName": "Halvorson",
  "gender": "female",
  "image": "https://robohash.org/autquiaut.png?size=50x50&set=set1",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MTUsInVzZXJuYW1lIjoia21pbmNoZWxsZSIsImVtYWlsIjoia21pbmNoZWxsZUBxcS5jb20iLCJmaXJzdE5hbWUiOiJKZWFubmUiLCJsYXN0TmFtZSI6IkhhbHZvcnNvbiIsImdlbmRlciI6ImZlbWFsZSIsImltYWdlIjoiaHR0cHM6Ly9yb2JvaGFzaC5vcmcvYXV0cXVpYXV0LnBuZz9zaXplPTUweDUwJnNldD1zZXQxIiwiaWF0IjoxNjM1NzczOTYyLCJleHAiOjE2MzU3Nzc1NjJ9.n9PQX8w8ocKo0dMCw3g8bKhjB8Wo7f7IONFBDqfxKhs"
}
```

, где token - это токен доступа: он необходим для получения доступа к данным пользователя, ... 
Для отправки запроса мы будем использовать запрос POST и передать в теле(body) username и password
```kotlin
data class User(
  val id: Int,
  val username: String",
  val email: String,
  val firstName: String,
  val lastName: String,
  val gender: String,
  val image: String,
  val token: String
)
```

Важно создавать идентификаторы в таком же регистре, как указано на сервере - иначе мы не сможем их принять(они сравниваются по именам).
Создадим класс, в котором будут данные для запроса - тело запроса(body):

```json
{
  username: 'kminchelle',
  password: '0lelplR'
}
```

```kotlin
data class AuthRequestBody(
    val username: String,
    val password: String
)
```

Теперь нужно изменить интерфейс для библиотеки retrofit:

```kotlin
import retrofit2.http.GET

interface MainAPI {
    @GET("products/{id}")
    suspend fun getProductById(@Path("id") id: Int): Product

    @POST("auth/login")
    suspend fun auth(@Body authRequest: AuthRequestBody): User
}
```

С помощью аннотации @POST мы указываем, что мы отправляем запрос на сервер и указываем пусть в аргементе(относительно адреса сервера).
С помощью аннотации @Body мы указываем тело запроса - объект, который мы передадим при отправке запроса. В ответ мы получим объект типа User.

Для отображения картинок мы будем использовать библиотеку Coil:

```kotlin
runtimeOnly("io.coil-kt:coil:2.5.0")
```

```kotlin
val mainApi = retrofit.create(MainAPI::class.java)
...
val user = mainApi.auth(
    AuthRequestBody(
        username = ...
        password = ...
    )
)
```

------------------------------------------------

Теперь получим весь список:

```json
{
  "products": [
    {
      "id": 1,
      "title": "iPhone 9",
      "description": "An apple mobile which is nothing like apple",
      "price": 549,
      "discountPercentage": 12.96,
      "rating": 4.69,
      "stock": 94,
      "brand": "Apple",
      "category": "smartphones",
      "thumbnail": "...",
      "images": ["...", "...", "..."]
    },
    {...},
    {...},
    {...}
    // 30 items
  ],

  "total": 100,
  "skip": 0,
  "limit": 30
}
```

Для отправки запроса мы будем использовать запрос GET

```kotlin
data class Products(
  val products: List<Product>
)
```

Т.о. мы можем указывать не все поля, а только те, которые нам нужны.

```kotlin
import retrofit2.http.GET

interface MainAPI {
    @GET("products/{id}")
    suspend fun getProductById(@Path("id") id: Int): Product

    @POST("auth/login")
    suspend fun auth(@Body authRequest: AuthRequestBody): User

    @GET("products")
    suspend fun getAllProducts(): Products
}
```

```kotlin
val mainApi = retrofit.create(MainAPI::class.java)
...
val productsObject = mainApi.getAllProducts()
title = productsObject.products[1].orEmpty()
```

------------------------------------------------

Теперь сделаем поиск продуктов - мы моожем передавать параметры поиска. Как именно их указывать в запросе - зависит от разработчиков на сервере. Мы также получаем список продуктов. Будем отправлять запрос на 'https://dummyjson.com/products/search?q=phone', где q - имя параметра.

```kotlin
data class Products(
  val products: List<Product>
)
```

```kotlin
interface MainAPI {
    @GET("products/{id}")
    suspend fun getProductById(@Path("id") id: Int): Product

    @POST("auth/login")
    suspend fun auth(@Body authRequest: AuthRequestBody): User

    @GET("products")
    suspend fun getAllProducts(): Products

    @GET("products/search")
    suspend fun getProductsByName(@Query("q") name: String): Products
}
```

Библиотека Retrofit сама будет добавлять парметры в запрос - какие именно параметры нужно передать мы указали с помощью аннотациии @Query.

```kotlin
val mainApi = retrofit.create(MainAPI::class.java)
...
val productsObject = mainApi.getProductsByName( имя )
title = productsObject.products[1].orEmpty()
```

------------------------------------------------

теперь попробуем получить доступ к данным пользователя. Для этого мы будем стучаться по адресу 'https://dummyjson.com/auth/RESOURCE', т.е. мы указываем все пути так же, то перед ними, но после адреса сервера дописываем auth/ , и тогда мы будем действовать как авторизированный пользователь.

```kotlin
interface MainAPI {
    ...
    @GET("auth/products/search")
    suspend fun getProductsByName(@Query("q") name: String): Products
}
```

Если мы действуем как авторизированный пользователь, то нужно предоставить токен, который мы получили при логировании. В таком случае вместе с запросом @GET мы должны передать объект заголовок(header), в который мы поместим токен. 

```json
{
  'Authorization': 'Bearer /* YOUR_TOKEN_HERE */', 
  'Content-Type': 'application/json'
},
```

Укажем их:

```kotlin
interface MainAPI {
    ...
    @HEADERS(
        "Content-Type: application/json"
    )
    @GET("auth/products/search")
    suspend fun getProductsByName(@Query("q") name: String): Products
}
```

Через аннотацию @HEADERS мы указываем через запятую статические заголовки. Для указания динамических заголовков, например, токена(он динамический, т.к. он обычно выдаётся на какой-то промежуток времени, а значит он меняется, и потому его нельзя зашить - сделать статическим), мы должны использовать аннотацию @HEADER("headerName") name: type, например:

```kotlin
interface MainAPI {
    ...
    @HEADERS(
        "Content-Type: application/json"
    )
    @GET("auth/products/search")
    suspend fun getProductsByName(@HEADER("Authorization") token: String, @Query("q") name: String): Products
}
```

Если мы не передаём токен в заголовках, то у нас будет исключение, а сервер вернёт сообщение: {"message": "Authentication Problem"} - у нас нет разрешения для получения данных пользователя.

Пробуем получить доступ:

```kotlin
val mainApi = retrofit.create(MainAPI::class.java)
...
var user: User? = null
user = mainApi.auth(
    AuthRequestBody(
        username = ...
        password = ...
    )
)
...
val productsObject = mainApi.getProductsByName(user?.token ?: "", имя )
title = productsObject.products[1].orEmpty()
```















```
