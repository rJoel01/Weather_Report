# Weather Report

## Descrizione  
L'app Weather Report ti permette di ottenere previsioni accurate in tempo reale basate sulla tua posizione, offrendoti tutte le informazioni di cui hai bisogno. I dati vengono attentamente elaborati per generare grafici intuitivi che facilitano la lettura delle previsioni e temperature, a seconda delle preferenze dell'utente, possono essere convertiti in diverse unità di misura, ad esempio da °C a °F.

## Linguaggi e Tecnologie Utilizzate  
- **Kotlin**  
- **Jetpack Compose**
- **Rest API**
- **Dagger Hilt**
- **Clean Architecture**

## Funzionalità Principali  
✔️ **Localizzazione del dispositivo** → Scansione in tempo reale di codici QR i quali se sono url vengono scansionati di nuovo per verificarne la sicurezza.  
✔️ **Fetch del meteo in base alla posizione** → Permette di riconoscere del testo usando 'Vision API'.  
✔️ **Creazione di grafici** → Determinare la lingua del testo scansionato ed eventualmente tradurlo con Firebase mlKit.  
✔️ **Conversione unità di misura** → Permette di usere una libreria di google per scansionare dei documenti e salvarli nell'app come file pdf.  

## 📂 Architettura dell'App

<img src="https://i.imgur.com/7cWGLGe.png" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

## Domain
Il livello Domain è indipendente da framework e librerie di terze parti. Qui vengono definiti:
- **Repository Interface** (repository) – Definisce il contratto per il recupero dei dati.
- **Weather Use Cases** (weather) – Contiene la logica per il recupero dei dati meteo.
- **Utility Functions** (util) – Funzioni di supporto e helper generici.

## Data
Questo livello si occupa di gestire le sorgenti dati.

- **Location** (data/location)

DefaultLocationTracker.kt – Implementazione del tracking della posizione dell’utente.

locationName.kt – Gestione del nome della posizione.

- **Remote Data Source** (data/remote)

WeatherAPI.kt – Interfaccia Retrofit per la comunicazione con il servizio meteo.

WeatherDataDto.kt & WeatherDTO.kt – Data Transfer Objects (DTO) per la gestione dei dati ricevuti dall’API.

- **Repository Implementation** (data/repository)

WeatherRepositoryImpl.kt – Implementazione del repository che raccoglie i dati dalle fonti (API o altre).

- **Data Mapping** (data/mappers)

WeatherMappers.kt – Funzioni di conversione tra DTO e modelli di dominio.


## Presentation

Gestisce l’interfaccia utente e l’interazione con l’utente.

- **Stable Components** (presentation/stable)

Componenti UI riutilizzabili.

- **UI Theme** (presentation/ui.theme)

Gestisce i colori, le tipografie e il tema dell’app.

- **Composables & Screens**

backgroundVideo.kt – Gestione dei video di sfondo nell’UI.

HourlyWeatherDisplay.kt – Visualizzazione delle previsioni orarie.

MainActivity.kt – Punto di ingresso dell’app, gestisce la navigazione e il lifecycle.


## Dependency Injection

L’app utilizza Hilt per l’iniezione delle dipendenze.

- **DI Modules** (di/)

AppModule.kt – Configurazione generale delle dipendenze.

LocationModule.kt – Fornisce le dipendenze per il tracking della posizione.

RepositoryModule.kt – Configura l’implementazione del repository.



## Panoramica dell'App

<p align="start">
Schermata Principale<br/>
<img src="https://i.imgur.com/mkm3dCm.jpg" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

## Location Tracker

```kotlin
class DefaultLocationTracker @Inject constructor(
    private val locationClient: FusedLocationProviderClient,
    private val application: Application
): LocationTracker {
    override suspend fun getCurrentLocation(): Location? {
        val hasAccessFineLocationPermission = ContextCompat.checkSelfPermission(
            application,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
        val hasAccessCoarseLocationPermission = ContextCompat.checkSelfPermission(
            application,
            Manifest.permission.ACCESS_COARSE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED


        val locationManager = application.getSystemService(Context.LOCATION_SERVICE) as LocationManager
        val isGpsEnabled = locationManager.isProviderEnabled(LocationManager.NETWORK_PROVIDER) ||
                locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER)

        if(!hasAccessFineLocationPermission || !hasAccessCoarseLocationPermission || !isGpsEnabled){
            return null
        }

        return suspendCancellableCoroutine { cont ->
            locationClient.lastLocation.apply {
                if(isComplete){
                    if(isSuccessful){
                        cont.resume(result)
                    }
                    else{
                        cont.resume(null)
                    }
                    return@suspendCancellableCoroutine
                }
                addOnSuccessListener {
                    cont.resume(it)
                }
                addOnFailureListener {
                    cont.resume(null)
                }
                addOnCanceledListener {
                    cont.cancel()
                }
            }
        }
    }
}
```

## Rest API

```kotlin
class WeatherRepositoryImpl @Inject constructor(
    private val api: WeatherAPI
) : WeatherRepository {
    override suspend fun getWeatherData(lat: Double, long: Double): Resource<WeatherInfo> {
        return try {
            Resource.Success(
                data = api.getWeatherData(
                    lat = lat,
                    long = long
                ).toWeatherInfo()
            )
        }
        catch (e : Exception){
            e.printStackTrace()
            Resource.Error(e.message ?: "An unknown error occurred")
        }
    }
}
```

## ViewModel

```kotlin
@HiltViewModel
class WeatherViewModel @Inject constructor(
    private val repository: WeatherRepository,
    private var locationTracker: LocationTracker
) : ViewModel() {

    var state by mutableStateOf(WeatherState())
        private set

    var lat by mutableStateOf(0.0)
    var long by mutableStateOf(0.0)

    fun loadWeatherInfo(){
        viewModelScope.launch {
            state = state.copy(
                isLoading = true,
                error = null
            )
            locationTracker.getCurrentLocation()?.let{ location->
                lat = location.latitude
                long = location.longitude
                when(val result = repository.getWeatherData(location.latitude,location.longitude)){
                    is Resource.Success ->{
                        state = state.copy(
                            weatherInfo = result.data,
                            isLoading = false,
                            error = null
                        )
                    }

                    is Resource.Error ->{
                        state = state.copy(
                            weatherInfo = null,
                            isLoading = false,
                            error = result.message
                        )
                    }
                }
            } ?: kotlin.run {
                state = state.copy(
                    isLoading = false,
                    error = "Couldnt retrive location. Make sure to garant permission and enable GPS."
                )
            }
        }
    }

    private var _temperature = MutableStateFlow(false)
    var temperature = _temperature

    private var _windSpeed = MutableStateFlow(false)
    var windSpeed = _windSpeed

    private var _airPressure = MutableStateFlow(false)
    var airPressure = _airPressure

    fun changeTemp(t : Boolean){
        _temperature.value = t
    }

    fun changeSpeed(s: Boolean){
        _windSpeed.value = s
    }

    fun changePressure(p : Boolean){
        _airPressure.value = p
    }

    fun loadPreferences(sharedPreferences: SharedPreferences) {
        _temperature.value = sharedPreferences.getBoolean("temp", false)
        _windSpeed.value = sharedPreferences.getBoolean("windSpeed", false)
        _airPressure.value = sharedPreferences.getBoolean("pressure", false)
    }

}
```

## Weather Forecast UI

```kotlin
@Composable
fun WeatherForecast(
        state: WeatherState,
        modifier: Modifier = Modifier,
        day: Int,
        dayName: String,
        temperature: Boolean
){
    state.weatherInfo?.weatherDataPerDay?.get(day)?.let {data ->
        Column(
            modifier = modifier
                .fillMaxWidth()
        ) {

            val finalData: List<WeatherData> = if (day == 0) hoursContinuation(state,data) else data

            Text(
                text = if (day == 0) stringResource(id = R.string.hours) else dayName.replaceFirstChar { it.uppercaseChar() },
                fontSize = 20.sp,
                color= Color.White,
                style = androidx.compose.ui.text.TextStyle(fontWeight = FontWeight.Bold, shadow = Shadow(
                    color = Color.Black,
                    offset = Offset(2f, 2f),
                    blurRadius = 8f
                )),
                modifier = Modifier.padding(start= 16.dp)
            )

            Box(modifier = Modifier.padding(10.dp,10.dp)){
                val context = LocalContext.current
                Box(
                    modifier = Modifier
                        .clippedShadow(8.dp, shape = RoundedCornerShape(16.dp))
                        .padding(10.dp)
                ) {
                    LazyRow(content = {
                        items(finalData) {weatherData ->
                                HourlyWeatherDisplay(
                                    weatherData = weatherData,
                                    modifier = Modifier.fillMaxSize(),
                                    temperature = temperature
                                )
                            Spacer(modifier = Modifier.width(16.dp))
                        }
                    })
                }
            }
        }
    }
}

fun hoursContinuation(
    state: WeatherState,
    data: List<WeatherData>
) : List<WeatherData>{
    val tomorrowData = state.weatherInfo?.weatherDataPerDay?.get(1)
    val data1 = data.toMutableList()
    val time = LocalDateTime.now()

    for(d in data){
        if (d.time <= time){
            data1.remove(d)
        }
    }

    for(d in tomorrowData!!){
        if(data1.size != 23){
            data1.add(d)
        }
        else{
            break
        }
    }
    return data1.toList()
}
```

## Weather Card UI

```kotlin
@Composable
fun WeatherCard(
    state: WeatherState,
    temperature: Boolean,
    windSpeed: Boolean,
    pressure: Boolean
){
    state.weatherInfo?.currentWeatherData.let { data ->
            Box(
                modifier = Modifier
                    .clippedShadow(8.dp, shape = RoundedCornerShape(16.dp)),
                contentAlignment = Alignment.Center
            ) {

                Column(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    val today = LocalDate.now().dayOfWeek.getDisplayName(
                        java.time.format.TextStyle.FULL,
                        Locale.getDefault()
                    ).replaceFirstChar { it.uppercaseChar() }

                    if (data != null) {
                        val temp = if (!temperature) {
                            "${data.temperatureCelsius}°C"
                        } else {
                            "${(data.temperatureCelsius * 9.0 / 5) + 32}°F"
                        }

                        Text(
                            text = "$today ${data.time.format(DateTimeFormatter.ofPattern("HH:mm"))}",
                            color = Color.White,
                            fontSize = 18.sp,
                            style = TextStyle(
                                shadow = Shadow(
                                color = Color.Black,
                                offset = Offset(2f, 2f),
                                blurRadius = 4f
                            )
                            ),
                            modifier = Modifier.padding(bottom = 8.dp)
                        )

                        val redirectedId = when (data.weatherType.weatherDesc) {
                            R.string.clear -> R.drawable.ic_moon
                            R.string.mainlyClear -> R.drawable.ic_clearnight
                            R.string.partlyCloudy -> R.drawable.ic_partlycloudynight
                            else -> data.weatherType.iconRes
                        }

                        val isNight = isNightTime()
                        val id = if (isNight) redirectedId else data.weatherType.iconRes

                        Image(
                            painter = painterResource(id = id),
                            contentDescription = null,
                            modifier = Modifier.size(120.dp)
                        )

                        Text(
                            text = temp,
                            fontSize = 50.sp,
                            color = Color.White,
                            style = TextStyle(
                                fontWeight = FontWeight.ExtraLight,
                                shadow = Shadow(
                                    color = Color.Black,
                                    offset = Offset(2f, 2f),
                                    blurRadius = 4f
                                ),
                                textAlign = TextAlign.Center
                            ),
                            modifier = Modifier.padding(top = 8.dp)
                        )


                        Text(
                            text = stringResource(id = data.weatherType.weatherDesc),
                            fontSize = 22.sp,
                            color = Color.White,
                            style = TextStyle(
                                shadow = Shadow(
                                    color = Color.Black,
                                    offset = Offset(2f, 2f),
                                    blurRadius = 4f
                                ),
                                textAlign = TextAlign.Center
                            ),
                            modifier = Modifier.padding(top = 8.dp)
                        )

                        Spacer(modifier = Modifier.height(11.dp))

                        Row(
                            modifier = Modifier
                                .clippedShadow(8.dp, RoundedCornerShape(25.dp))
                                .fillMaxWidth()
                                .padding(5.dp),
                            horizontalArrangement = Arrangement.SpaceAround
                        ) {
                            WeatherDataDisplay(
                                value = if (!pressure) data.pressure.roundToInt() else (data.pressure / 33.8639).roundToInt(),
                                unit = if (!pressure) "hPa" else "inHg",
                                icon = ImageVector.vectorResource(id = R.drawable.ic_pressure),
                                iconTint = Color.White,
                                textStyle = TextStyle(color = Color.White)
                            )

                            WeatherDataDisplay(
                                value = data.humidity.roundToInt(),
                                unit = "%",
                                icon = ImageVector.vectorResource(id = R.drawable.ic_drop),
                                iconTint = Color.White,
                                textStyle = TextStyle(color = Color.White)
                            )

                            WeatherDataDisplay(
                                value = if (!windSpeed) data.windSpeed.roundToInt() else (data.windSpeed * 0.621371).roundToInt(),
                                unit = if (!windSpeed) "km/h" else "mph",
                                icon = ImageVector.vectorResource(id = R.drawable.ic_wind),
                                iconTint = Color.White,
                                textStyle = TextStyle(color = Color.White)
                            )
                        }
                    }
                }
            }
    }

}

fun isNightTime(): Boolean {
    val now = LocalTime.now()
    val start = LocalTime.of(18, 0)
    val end = LocalTime.of(6, 0)

    return now.isAfter(start) || now.isBefore(end)
}
```

## Temperature Chart

```kotlin
@Composable
fun temperatureChart(
    state: WeatherState,
    temperature: Boolean,
    day: Int
){

    val todayData = state.weatherInfo?.weatherDataPerDay?.get(day)
    val todayTemperature = if(todayData != null) getTemperature(todayData,temperature) else null

    if (todayTemperature != null){

        val minTemperature = todayTemperature.minOf { it.y }
        val maxTemperature = todayTemperature.maxOf { it.y }

        val t = if(!temperature) "°C" else "°F"

        val xAxisData = AxisData.Builder()
            .axisStepSize(100.dp)
            .steps(todayTemperature.size -1)
            .labelData { i ->
               if(i == 0){
                   "             0$i:00"
               }else{
                   if(i<10) "0$i:00" else "$i:00"
               }
            }
            .labelAndAxisLinePadding(15.dp)
            .axisLineColor(Color.White)
            .axisLabelColor(Color.White)
            .build()

        val yAxisData = AxisData.Builder()
            .steps(5)
            .labelAndAxisLinePadding(20.dp)
            .labelData { i ->
                val scale = (maxTemperature - minTemperature) / 5
                val value = minTemperature + i * scale
                value.toInt().toString() + t
            }
            .axisLineColor(Color.White)
            .axisLabelColor(Color.White)
            .backgroundColor(Color.DarkGray)
            .build()

        val lineChartData = LineChartData(
            linePlotData = LinePlotData(
                lines = listOf(
                    Line(
                        dataPoints = todayTemperature,
                        LineStyle(
                            color = Color.White,
                            lineType = LineType.Straight(true)
                        ),
                        IntersectionPoint(
                            color = Color.LightGray
                        ),
                        SelectionHighlightPoint(
                            color = Color.White
                        ),
                        ShadowUnderLine(
                            alpha = 0.5f,
                            brush = Brush.verticalGradient(
                                colors = listOf(
                                    Color.White,
                                    Color.Transparent
                                )
                            )
                        ),
                        SelectionHighlightPopUp(
                            popUpLabel = { x: Float, y: Float ->
                                "Temp: $y$t"
                            }
                        )
                    )
                ),
                plotType = PlotType.Line
            ),
            xAxisData = xAxisData,
            yAxisData = yAxisData,
            gridLines = null,
            isZoomAllowed = false,
            backgroundColor = Color.DarkGray,
            paddingRight = 0.dp,
        )

        Box(
            modifier = Modifier.padding(10.dp)
        ) {
            Column(
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.spacedBy(10.dp),
                modifier = Modifier
                    .clip(RoundedCornerShape(16.dp))
                    .background(Color.DarkGray)
            ) {

                Spacer(modifier = Modifier.height(10.dp))

                Text(
                    text = if(day == 0) stringResource(id = R.string.todayTempChart) else stringResource(
                        id = R.string.temperatureChart
                    ),
                    fontSize = 20.sp,
                    color= Color.White,
                    style = androidx.compose.ui.text.TextStyle(fontWeight = FontWeight.Bold)
                )

                LineChart(
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(250.dp),
                    lineChartData = lineChartData,
                )
            }
        }
    }


}

fun getTemperature(data: List<WeatherData>, temperature: Boolean) : List<co.yml.charts.common.model.Point> {

    val list : ArrayList<co.yml.charts.common.model.Point> = ArrayList()

    for(i in data){
        val xTime = i.time.hour.toFloat()
        val yTemperature = if(!temperature){
            i.temperatureCelsius.toFloat()
        }else{
            i.temperatureCelsius.toFahrenheit().toFloat()
        }
        list.add(co.yml.charts.common.model.Point(xTime,yTemperature))
    }

    return list.toList()

}

private fun Double.toFahrenheit() : Double{
    return (this * 9.0/5) + 32
}
```

</p>

<!--
 ```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
--!>
