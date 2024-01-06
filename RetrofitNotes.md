
--------------------
Retrofit
--------------------

Нужны такие зависимовти:

implementation("com.squareup.retrofit2:retrofit:2.9.0")
implementation("com.squareup.retrofit2:converter-gson:2.9.0")

Т.к. мы подключили GSON конвертор, то он он сам будет преобразовывать JSON в наш DataClass:

//{
//    "id": 1,
//    "title": "iPhone 9",
//    "description": "An apple mobile which is nothing like apple",
//    "price": 549,
//    "discountPercentage": 12.96,
//    "rating": 4.69,
//    "stock": 94,
//    "brand": "Apple",
//    "category": "smartphones",
//    "thumbnail": "...",
//    "images": ["...", "...", "..."]
//}

в

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
    val images: List<String>,
)

Теперь нам надо прописать некоторый интерфейс, коротый будет описывать наш API для взаимодействия с сервером:

import retrofit2.http.GET

interface ProductAPI {
    @GET("product/1")
    suspend fun getProductById(): Product
}

Здесь мы указываем путь к объекту относительно адреса сервера: @GET("product/1").
Это будет suspend функция, т.е. будем её запускать в корутине.
Нужно также добавить в манифест разрешение на выход в интернет:

<uses-permission android:name="android.permission.INTERNET" />

Теперь создаём экземпляр ретрофита:

val retrofif = Retrofit.Builder()
  .baseUrl("https://dummyjson.com")
  .addConverterFactory(GsonConverterFactory.create())
  .build()

, где:
* baseUrl - указываем имя сервера
* addConverterFactory - указываем конвертор, который будет преобразовывать наши данные - экземпляр GsonConverterFactory
* build - создать экземпляр

А вот теперь мы с помощью экземпляра указываем ЧТО мы хотим создать - какой интерфейс:

val productApi = retrofif.create(ProductAPI::class.java)

После чего библиотека ретрофит создаст класс на основе этого интерфейса. Теперь можно им пользоваться:

productApi.getProductById()

Но делать это с главного потока нельзя, т.к. он будет тормозить. Поэтому мы будем использовать корутины на второстепенном потоке(например, Dispatchers.IO):

CoroutineScope(Dispatchers.IO)

Если мы хотим, чтобы можно было указыть часть пути в функции GET запроса, то можно сделать так:

interface ProductAPI {
    @GET("products/{id}")
    suspend fun getProductById(@Path("id") id: Int): Product
}

Весь код:

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
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import com.example.retrofit1.retrofit.ProductAPI
import com.example.retrofit1.ui.theme.Retrofit1Theme
import kotlinx.coroutines.launch
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val retrofif = Retrofit.Builder()
            .baseUrl("https://dummyjson.com")
            .addConverterFactory(GsonConverterFactory.create())

            .build()
        val productApi = retrofif.create(ProductAPI::class.java)


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
                    Column(

                    ) {
                        Text(text = title)
                        Button(onClick = {
                            coroutineScope.launch {
                                coroutineScope.launch {
                                    try {
                                        val product = productApi.getProductById(3)
                                        title = product.title.orEmpty()
                                    } catch (e: Exception) {
                                        title = "Failed to load"
                                        Log.e("MainActivity", "Error fetching product", e)
                                    }
                                }
                            }
                        }) {

                        }

                    }
                }
            }
        }
    }
}




























