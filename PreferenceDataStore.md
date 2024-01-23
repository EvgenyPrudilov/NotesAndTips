Применение Preference DataStore для хранения примитивных данных таких как: Int, Boolean, Long и.т.д

Необходимая зависимость:

```
implementation("androidx.datastore:datastore-preferences:1.0.0")
```

Создадим класс DataStoreManager, с помощью которого мы будет сохранять и получать данные их хранилища. Он принимает контекст, чтобы мы могли инициализировать переменную dataStore с нашим хранилищем.
Нам нужно будет инициализировать наш DataStore, когда мы будем запускать наш класс.
Мы создаём SharedPreferences как таблицу для хранения данных, и у каждой такблицы может быть уникальное имя.

```
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore("data_store")
class DataStoreManager(val context: Context) {
}
```

Мы будем создавать suspend функции, т.е. функции записи и чтения данных из хранилища могут занять много времени. Мы можем сохранять как отдельные значения, так и датаклассы.

```
data class SettingsData(
    val textSize: Int,
    val bgColor: Long
)

class DataStoreManager(val context: Context) {
    suspend fun saveSettings(settingsData: SettingsData) {
        context.dataStore.edit { pref ->
            pref[intPreferencesKey("text_size")] = settingsData.textSize
            pref[longPreferencesKey("bg_color")] = settingsData.bgColor
        }
    }
}
```

метод edit() принимает функцию, которая принимает параметром MutablePreferences.
pref - тип MutablePreferences - мы через него записываем данные.
intPreferencesKey - функция для сохранения объекта типа Int - и мы сразу передаём имя параметра.

Для Получения данных:

```
val Red = Color(0xFFDD3737)
val Green = Color(0xFF56CE43)
val Blue = Color(0xFF4770E8)

import kotlinx.coroutines.flow.map

fun getSettings() = context.dataStore.data.map { pref ->
    return@map SettingsData(
        pref[intPreferencesKey("text_size")] ?: 40,
        pref[longPreferencesKey("bg_color")] ?: Red.value.toLong()
    )
}
```

pref имеет тип Preferences, т.е. не изменяемый тип
Мы не пишем suspend fun, т.к. у нас функция map из kotlinx.coroutines.flow

```
val dataStoreManager = DataStoreManager(this)
setContent {
    DataStore1Theme {
        val bgColor = remember {
            mutableStateOf(Red.value)
        }
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = Color(bgColor.value)
        ) {
            Greeting()
        }
    }
}
```

bgColor - наше состоянеие. При его изменении, у нас будет перерисовываться Surface, т.к. его параметр color зависит от bgColor

Получать данные и их изменять мы можем только в корутин скоуп. Для этого можно использовать LaunchedEffect - у неё параметром будет suspend лямбда, в который мы уже можем вызывать:

```
LaunchedEffect(key1 = true) {
    Log.d("*_*", "\tLaunchedEffect")
    dataStoreManager.getSettings().collect { settings ->
        bgColorState.value = settings.bgColor.toULong()
        textSizeState.value = settings.textSize
    }
}
```

Полный код:

```
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val dataStoreManager = DataStoreManager(this)

        setContent {
            Log.d("*_*", "setContent")
            DataStore1Theme {
                val bgColorState = remember {
                    mutableStateOf(Red.value)
                }
                val textSizeState = remember {
                    mutableStateOf(40)
                }
                LaunchedEffect(key1 = true) {
                    Log.d("*_*", "\tLaunchedEffect")
                    dataStoreManager.getSettings().collect { settings ->
                        bgColorState.value = settings.bgColor.toULong()
                        textSizeState.value = settings.textSize
                    }
                }

                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = Color(bgColorState.value)
                ) {
                    Log.d("*_*", "\tSurface")
                    MainScreen(dataStoreManager, textSizeState)
                }
            }
//            MyComposable()
        }
    }
}

@Composable
fun MainScreen(
    dataStoreManager: DataStoreManager,
    textSizeState: MutableState<Int>
) {
    Log.d("*_*", "\t\tMainScreen")
    val coroutine = rememberCoroutineScope()
    Column(
        modifier = Modifier
            .fillMaxSize()
        ,
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Log.d("*_*", "\t\t\tColumn")
        Box(
            modifier = Modifier
                .fillMaxSize(0.5f)
                .wrapContentHeight(align = Alignment.CenterVertically)
                .wrapContentWidth(align = Alignment.CenterHorizontally)
        ) {
            Log.d("*_*", "\t\t\t\tBox")
            Text(
                text = "text",
                color = Color.White,
                fontSize = textSizeState.value.sp
            )
        }
        Button(
            onClick = {
                Log.d("*_*", "\t\t\t\tButton Blue onClick")
                coroutine.launch {
                    dataStoreManager.saveSettings(
                        SettingsData(
                            textSize = 10,
                            bgColor = Blue.value.toLong()
                        )
                    )
                }
            }
        ) {
            Text(text = "Blue")
        }
        Spacer(modifier = Modifier.height(10.dp))

        Button(
            onClick = {
                Log.d("*_*", "\t\t\t\tButton Red onClick")
                coroutine.launch {
                    dataStoreManager.saveSettings(
                        SettingsData(
                            textSize = 30,
                            bgColor = Red.value.toLong()
                        )
                    )
                }
            }
        ) {
            Text(text = "Red")
        }
        Spacer(modifier = Modifier.height(10.dp))

        Button(
            onClick = {
                Log.d("*_*", "\t\t\t\tButton Green onClick")
                coroutine.launch {
                    dataStoreManager.saveSettings(
                        SettingsData(
                            textSize = 20,
                            bgColor = Green.value.toLong()
                        )
                    )
                }
            }
        ) {
            Text(text = "Green")
        }
    }
}
```

Можно сделать работу с состоянием проще:

```
...
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore("data_store")
class DataStoreManager(val context: Context) {
    suspend fun saveSettings(settingsData: SettingsData) {
        context.dataStore.edit { pref ->
            pref[intPreferencesKey("text_size")] = settingsData.textSize
            pref[longPreferencesKey("bg_color")] = settingsData.bgColor.toLong()
        }
    }

    fun getSettings() = context.dataStore.data.map { pref ->
        return@map SettingsData(
            pref[intPreferencesKey("text_size")] ?: 40,
            pref[longPreferencesKey("bg_color")]?.toULong() ?: Red.value
        )

    }
}
...
val dataStoreManager = DataStoreManager(this)
...
setContent {
    Log.d("*_*", "setContent")
    DataStore1Theme {
        val settingsState = dataStoreManager
            .getSettings()
            .collectAsState(initial = SettingsData())
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = Color(settingsState.value.bgColor)
        ) {
            Log.d("*_*", "\tSurface")
            MainScreen(dataStoreManager, settingsState.value.textSize)
        }
    }
}
...
```













