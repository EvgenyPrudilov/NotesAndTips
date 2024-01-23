
С помощью PreferenceDataStore мы можем сохранять примитивные данные, но не можем сохранять объекты - мы можем это делать только сохранением отдельных компонентов этих объектов. Причём, PreferenceDataStore не типобезопасно, т.к. мы можем сохранитрь данные как intPreferencesKey, а получить как stringPreferencesKey - мы сами должны помнить типы значений. ProtoDataStore решает эту проблему.

нужно добавить зависимости и плагины:

```
id("org.jetbrains.kotlin.plugin.serialization") version "1.6.10" apply false
...
id("org.jetbrains.kotlin.plugin.serialization")
...
implementation("androidx.datastore:datastore:1.0.0")
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.3.2")
```

теперь нужно пометить датакласс как сериализованный:

```
@Serializable
data class SettingsData(
    val textSize: Int = 40,
    val bgColor: ULong = Red.value
)
```

Теперь нужно создать сериализатор, с помощью чего мы указываем как мы данные будем сохранять и как получать и значение по умолчанию. Мы будем записывать в память с помощью метода write, но ему надо будет передать массив байт, а для этого нам нужно будет как-то наши данные перевести в байты:

```
object SettingsSerializer : Serializer<SettingsData> {
    override val defaultValue: SettingsData
        get() = SettingsData()

    override suspend fun readFrom(input: InputStream): SettingsData {
        return try {
            Json.decodeFromString(
                deserializer = SettingsData.serializer(),
                string = input.readBytes().toString()
            )
        } catch (e: SerializationException) {
            e.printStackTrace()
            SettingsData()
        }
    }

    override suspend fun writeTo(t: SettingsData, output: OutputStream) {
        withContext(Dispatchers.IO) {
            output.write(
                Json.encodeToString(
                    serializer = SettingsData.serializer(),
                    value = t
                ).encodeToByteArray()
            )
        }
    }
}
```

Json.encodeToString - переводит объект в строку. Для этого нам нужно указать шаблон-сериалайзер, чтобы Json знал, как можно закодировать наш класс. Это возможно благодаря тому, что мы класс SettingsData пометили как Serializable. Записываем мы значение t. Т.о мы указываем данные, и сериалайзер для их преобразования. После, мы преобразовываем строку в массив байт:

```
Json.encodeToString(
    serializer = SettingsData.serializer(),
    value = t
).encodeToByteArray()
```

Но это может занять время, поэтому это можно делать в конутине на второстепенном потоке:

```
withContext(Dispatchers.IO) {
    output.write(
        Json.encodeToString(
            serializer = SettingsData.serializer(),
            value = t
        ).encodeToByteArray()
    )
}
```

Далее мы создадим менеджер:

```
private val Context.protoDataStore by dataStore("settings.json", SettingsSerializer)
class ProtoDataStoreManager(val context: Context) {
}
```

мы передаём делегату имя файла, куда будут сохраняться данные и сериалайзер, чтобы знать, как записывать и читать данные.

```
suspend fun saveColor(color: ULong) {
    context.protoDataStore.updateData { data ->
        data.copy(bgColor = color)
    }
}
```

Здесь data - это данные(тип в данном случае SettingsData), которые уже записаны в DataStore(или значение по умолчанию). После мы его копируем, создавая новый.

```
class ProtoDataStoreManager(val context: Context) {
    suspend fun saveColor(color: ULong) {
        context.protoDataStore.updateData { data ->
            data.copy(bgColor = color)
        }
    }

    suspend fun saveTextSize(size: Int) {
        context.protoDataStore.updateData { data ->
            data.copy(textSize = size)
        }
    }

    suspend fun saveTextSize(settingsData: SettingsData) {
        context.protoDataStore.updateData { data ->
            settingsData
        }
    }

    fun getSettings() = context.protoDataStore.data
}
```

Т.к. мы здесь также как и в PreferencesDataStore возвращаем Flow, то пишем также:

```
val dataStoreManager = ProtoDataStoreManager(this)

        setContent {
            Log.d("*_*", "setContent")
            DataStore1Theme {
                val settingsState = dataStoreManager
                    .getSettings()
                    .collectAsState(initial = SettingsData())
```
