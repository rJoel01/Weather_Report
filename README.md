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
✔️ **Scansione Codici QR** → Scansione in tempo reale di codici QR i quali se sono url vengono scansionati di nuovo per verificarne la sicurezza.  
✔️ **Riconoscimento Testo** → Permette di riconoscere del testo usando 'Vision API'.  
✔️ **Traduzione Testo** → Determinare la lingua del testo scansionato ed eventualmente tradurlo con Firebase mlKit.  
✔️ **Scansione Documenti** → Permette di usere una libreria di google per scansionare dei documenti e salvarli nell'app come file pdf.  
✔️ **Riconoscimento Landmark** → Feature per riconoscere i punti di riferimento dalla galleria o dalle foto scattate nell'app.  

## Panoramica dell'App  

<p align="start">
Schermata Principale<br/>
<img src="https://i.imgur.com/bXSgKiN.jpeg" height="80%" width="40%" alt="Schermata App"/>
<br /><br />

## Camera Preview Composable
Implementazione della preview usando jetpack compose e cameraX:

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
