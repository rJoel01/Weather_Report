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
