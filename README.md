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

## Struttura del progetto - Weather Report
L’app Weather Report è strutturata secondo i principi della Clean Architecture, garantendo modularità, testabilità e una chiara separazione delle responsabilità. Il codice è organizzato in pacchetti distinti per ogni livello architetturale.

## 📂 Strati dell'architettura

<img src="https://i.imgur.com/7cWGLGe.png" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

```kotlin
@Composable
fun CameraPreview(
    modifier: Modifier = Modifier,
    viewModel: MainViewmodel,
    controller: LifecycleCameraController
) {
    val torchState by viewModel.torchState.collectAsState()
    var tapPosition by remember { mutableStateOf<Offset?>(null) }

    Box(modifier = modifier.fillMaxSize()) {
        AndroidView(factory = {
            PreviewView(it).apply {
                this.controller = controller

                this.setOnTouchListener { view, event ->
                    if (event.action == MotionEvent.ACTION_DOWN) {
                        val factory = this.meteringPointFactory
                        val meteringPoint = factory.createPoint(event.x, event.y)
                        val action = FocusMeteringAction.Builder(meteringPoint).build()
                        this.controller?.cameraControl?.startFocusAndMetering(action)

                        val viewLocation = IntArray(2)
                        view.getLocationOnScreen(viewLocation)
                        val adjustedX = event.rawX - viewLocation[0]
                        val adjustedY = event.rawY - viewLocation[1]
                        tapPosition = Offset(adjustedX, adjustedY)
                    }
                    true
                }
            }
        }, modifier = Modifier.fillMaxSize())

        tapPosition?.let { position ->
            FocusIndicator(position)
        }

        if (torchState) {
            controller.enableTorch(true)
        } else {
            controller.enableTorch(false)
        }
    }
}
```

## Scansione QR
Implementazione della scansionde del qr sull'Image Analysis Analyzer

```kotlin
controller = LifecycleCameraController(this@SecondActivity).apply {
            setEnabledUseCases(
                CameraController.IMAGE_CAPTURE or CameraController.IMAGE_ANALYSIS
            )
            setImageAnalysisAnalyzer(
                ContextCompat.getMainExecutor(applicationContext),
                QrCodeAnalyzer{
                    viewModel.OnQRanalized0(it)
                }
            )
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
