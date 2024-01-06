```markdown
# Retrofit
Нужны такие зависимости:

```kotlin
implementation("com.squareup.retrofit2:retrofit:2.9.0")
implementation("com.squareup.retrofit2:converter-gson:2.9.0")
```

Т.к. мы подключили GSON конвертор, то он он сам будет преобразовывать JSON в наш DataClass:

```json
// {
//   "id": 1,
//   "title": "iPhone 9",
//   "description": "An apple mobile which is nothing like apple",
//   "price": 549,
//   "discountPercentage": 12.96,
//   "rating": 4.69,
//   "stock": 94,
//   "brand": "Apple",
//   "category": "smartphones",
//   "thumbnail": "...",
//   "images": ["...", "...", "..."]
// }
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
    @GET("product/1")
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
```
