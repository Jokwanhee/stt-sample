# Android STT

Android에서 음성을 텍스트로 변환하기 위해서 사용할 방법들에 대해서 설명합니다.

- **SpeechRecognizer를 사용한 안드로이드 내장 STT API 사용하기**
- **SpeechRecognizer를 사용한 안드로이드 Google STT UI 사용하기**
- **STT 모델을 안드로이드에 적용해서 사용하기**
- **gRPC 기반의 Google API 인 Speech-to-text(STT) v1 사용하기**

위 4가지를 사용할 수 있습니다. 외부 클라우드를 사용한 API는 네이버, AWS 등 더 많이 있지만 Google만 다룹니다.

## 공통

**RECORD_AUDIO** **Pemrssion**

오디오 권한처리가 되어야합니다. 

```kotlin
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

## **안드로이드 내장 API - SpeechRecognizer STT**

안드로이드 내부에서 STT를 사용할 수 있게 API를 제공해줍니다.

### **등록 및 이벤트 콜백 리스너**

```kotlin
speechRecognizer = SpeechRecognizer.createSpeechRecognizer(context).apply {
	setRecognitionListener(object : RecognitionListener {
		// 음성 인식 준비 완료
    override fun onReadyForSpeech(params: Bundle?) {}
		// 사용자가 말하기 시작
    override fun onBeginningOfSpeech() {}
    // 음성 볼륨 변화 (rmsdB: -2 ~ 10 정도 범위)
    override fun onRmsChanged(rmsdB: Float) {}
	  // 음성 데이터를 버퍼로 수신
    override fun onBufferReceived(buffer: ByteArray?) {}
    // 사용자가 말하기 멈춤
    override fun onEndOfSpeech() {}
		// 에러 발생 시 처리 (SpeechRecognizer.ERROR_*)
    override fun onError(error: Int) {}
		// 인식된 음성의 부분 결과 (중간 인식)
    override fun onPartialResults(partialResults: Bundle?) {}
		// 최종 인식 결과
    override fun onResults(results: Bundle?) {}
    override fun onEvent(eventType: Int, params: Bundle?) {}
})
```

### **시작**

```kotlin
val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
  putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
  putExtra(RecognizerIntent.EXTRA_LANGUAGE, "ko-KR")
  putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
}
speechRecognizer?.startListening(intent)
```

- **RecognizerIntent.EXTRA_LANGUAGE** : 음성 인식 언어
- **RecognizerIntent.EXTRA_PARTIAL_RESULTS** : 부분 결과(중간 인식)도 수신 여부

### **결과 콜백**

음성 인식 텍스트 변환을 아래 콜백 함수로 받아올 수 있습니다.

```kotlin
override fun onPartialResults(partialResults: Bundle?) {
	val data = partialResults?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
}

override fun onResults(results: Bundle?) {
	val data = results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
}
```

### **에러 코드**

SpeechRecognizer 클래스에 ERROR 변수 확인 가능

```kotlin
SpeechRecognizer.ERROR_AUDIO
```

### **정지 및 종료**

음성 인식을 정지할 경우와 음성 인식을 그만할 경우 리소스 해제도 해주어야 합니다.

```kotlin
// 정지
speechRecognizer?.stopListening()
speechRecognizer?.cancel()

// 종료
speechRecognizer?.destroy()
```

### **장점**

- **상대적으로 간단한 구현**
    - 구글 클라우드 인증이나 외부 라이브러리를 추가하지 않아도 됩니다.
- **부분 인식(Partial results) 및 최종 인식(Results) 모두 콜백 가능**
- **사용자 기기에 이미 설치된 음성인식 엔진 활용**
    - 별도 서버 비용 없이 기기에서 네트워크(혹은 구글 음성인식 서버)와 연결되어 처리
- **무료 사용**
    - 구글 STT 클라우드 API를 직접 쓰면 일정 무료 한도 초과 시 과금이 발생

### **단점**

- **짧은 세션 제한**
    - 특정 기기에서는 **5초**로 고정되어, 장시간 연속 녹음(무한 스트리밍)이 어렵습니다.
    - `RecognizerIntent.EXTRA_SPEECH_INPUT_MINIMUM_LENGTH_MILLIS` 등 인텐트 파라미터로 세션 시간을 늘리려 해도, 실제로 반영되지 않는 경우가 많습니다.
- **기기·OS 의존성**
- **오프라인 지원 불확실**
- **커스텀 모델 적용 불가**

## **안드로이드  Google STT UI - SpeechRecognizer STT**

compose에서 사용하는 방법에 대해서 다룹니다.

RecognizerIntent.EXTRA_PROMPT로 activity result api를 사용하게 될 경우 Google 에서 제공하는 STT UI 를 사용할 수 있습니다.

```kotlin
val launcher =
  rememberLauncherForActivityResult(ActivityResultContracts.StartActivityForResult()) {
      if (it.resultCode == Activity.RESULT_OK) {
          val data = it.data
          val result = data?.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)
      } else { }
  }
  
val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
  putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
  putExtra(RecognizerIntent.EXTRA_LANGUAGE, Locale.getDefault())
  putExtra(RecognizerIntent.EXTRA_PROMPT, "말씀하세요.")
}
launcher.launch(intent)
```

### **장점**

- **간단한 사용**
    - `SpeechRecognizer` 및 커스텀 콜백을 직접 구현할 필요가 없음.
- **기본 구글 디자인/UX**

### **단점**

- **세션 시간 제한**
    - 일반적으로 짧은 시간(기기별로 5~10초 정도) 후 자동 종료될 수 있습니다.
    - 장시간 연속 녹음/인식을 구현하기에는 부적합.
- **커스텀 불가**
    - UI를 수정하거나 인식 과정을 세밀하게 제어 불가
    - 부분인식(Partial Results)만 실시간으로 받기가 힘들고, 오직 최종 결과만 받을 수 있는 구조

## 안드로이드 STT 외부 모델 - VOSK STT Model

외부 STT 모델을 안드로이드 스튜디오에 다운로드하여 모델을 로드하고 STT를 사용하는 방법입니다.

```kotlin
// build.gradle.kts (:app)

android {
	...

  defaultConfig {
    ...
    ndkVersion = "25.2.9519653"
    ndk {
        abiFilters.addAll(arrayOf("armeabi-v7a", "arm64-v8a", "x86_64", "x86"))
    }
  }
}
  
dependencies {
	implementation(project(":models"))
	implementation("net.java.dev.jna:jna:5.13.0@aar")
	implementation("com.alphacephei:vosk-android:0.3.32+")
}
```

### **STT 모델을 가지는 모듈 분리 (models)**

```kotlin
// build.gradle.kts (:models)

android {
	...
	sourceSets {
    getByName("main") {
      assets.srcDirs(files("$buildDir/generated/assets"))
    }
  }
}

tasks.register("genUUID") {
    val uuid = UUID.randomUUID().toString()
    val odir = file("$buildDir/generated/assets/model-small-ko")
    val ofile = file("$odir/uuid")
    doLast {
        mkdir(odir)
        ofile.writeText(uuid)
    }
}

tasks.named("preBuild") {
    dependsOn("genUUID")
}
```

모델 모듈을 분리했다면, 해당 모델을 모듈 assets 경로에 위치시켜야 합니다.

```kotlin
model
├── src
│   └── main
│       └── assets
│           └── model-small-ko  (다운로드한 VOSK 모델)
└── build.gradle.kts
```

### **초기 셋팅**

```kotlin
// Vosk 관련 객체 상태
var model: Model? by remember { mutableStateOf(null) }
var speechService: SpeechService? by remember { mutableStateOf(null) }
var speechStreamService: SpeechStreamService? by remember { mutableStateOf(null) }

// model-small-ko를 Assets에서 "model" 폴더로 언팩
LaunchedEffect(Unit) {
  try {
      // 로그 레벨 (선택)
      LibVosk.setLogLevel(LogLevel.INFO)

      // 언팩 (model-small-ko -> 내부 저장소의 "model" 폴더)
      StorageService.unpack(
          context,
          "model-small-ko",   // assets 폴더 안의 모델 이름
          "model",            // 내부에서 사용될 폴더명
          { m -> // 성공 시
              model = m
          },
          { ex -> // 실패 시
          }
      )
  } catch (e: Exception) { }
}
```

### **콜백 리스너**

```kotlin
// RecognitionListener 구현
val recognitionListener = object : RecognitionListener {
  override fun onPartialResult(hypothesis: String?) {
      if (hypothesis == null) return
      val json = JSONObject(hypothesis)
      if (json.has("partial")) {
          val text = json.getString("partial")
      }
  }
  override fun onResult(hypothesis: String?) {
      if (hypothesis == null) return
      val json = JSONObject(hypothesis)
      if (json.has("text")) {
          val resultText = json.getString("text")
      }
  }
  override fun onFinalResult(hypothesis: String?) {
      // 파일 / 마이크 종료
      speechStreamService?.stop()
      speechStreamService = null
      speechService?.stop()
      speechService?.shutdown()
      speechService = null
  }
  override fun onError(e: Exception?) { }
  override fun onTimeout() { }
}
```

### **마이크 인식 API**

```kotlin
recognizeMic(
  model!!,
  recognitionListener,
  onStart = { // 마이크 인식 진행 중
  },
  onServiceCreated = { service -> // 마이크 인식 완료
      speechService = service
  },
  onError = { // 에러 발생
  }
)

fun recognizeMic(
  model: Model,
  recognitionListener: RecognitionListener,
  onStart: () -> Unit,
  onServiceCreated: (SpeechService) -> Unit,
  onError: (String) -> Unit
) {
  onStart()
  try {
    val rec = Recognizer(model, 16000f)
    val service = SpeechService(rec, 16000f)
    onServiceCreated(service)
    service.startListening(recognitionListener)
  } catch (e: IOException) { // 마이크 인식 실패
  }
}
```

### **파일 인식 API**

```kotlin
fun recognizeFile(
  model: Model,
  recognitionListener: RecognitionListener,
  onStart: () -> Unit,
  onServiceCreated: (SpeechStreamService) -> Unit,
  onError: (String) -> Unit
) {
  onStart()
  try {
    // 예시용: assets/폴더에 있는 WAV 파일로 테스트
    // 파일명과 샘플레이트, Recognizer 초기 파라미터 등 적절히 설정
    val rec = Recognizer(model, 16000f)

    // 파일 열기 (예: "10001-90210-01803.wav")
    // 실제 파일명은 바꿔야 함
    val ais: InputStream = applicationContext.assets.open("10001-90210-01803.wav")
    // WAV 헤더(44바이트) 건너뛰기
    if (ais.skip(44) != 44L) throw IOException("WAV 파일이 너무 짧습니다.")

    val speechStreamService = SpeechStreamService(rec, ais, 16000f)
    onServiceCreated(speechStreamService)

    speechStreamService.start(recognitionListener)
  } catch (e: IOException) { // 파일 인식 실패
  }
}
```

### **장점**

- **오프라인 STT**
    - 네트워크 없이도 작동하며, 별도의 서버비나 과금 문제가 없음.
    - 장시간 연속 녹음 인식 가능(내장 SpeechRecognizer처럼 5초~1분 제한 없음).
- **자유로운 커스터마이징**
    - 원하는 모델(언어·도메인)에 따라 VOSK 모델을 교체 가능.
    - 앱 내에서 **부분 인식**·**최종 인식** 등 세밀한 처리가 용이.

### **단점**

- **정확도**
    - 구글 클라우드 등 상용 서비스에 비해 정확도가 떨어질 수 있습니다.
    - 특히 잡음이 있거나, 전문용어가 많으면 인식률이 낮음.
- **앱 용량 증가**
    - 모델 파일이 수십~수백 MB에 달할 수 있습니다.
    - 모든 사용자에게 동일 모델을 배포해야 하므로, 설치 용량이 커질 수 있음.
- **모델 업데이트 번거로움**
    - 모델 성능을 향상하려면 **새 버전을 앱에 탑재** 후 배포.
    - 클라우드처럼 자동 모델 업데이트가 되지 않음.

### Reference

Vosk Github Sample : https://github.com/alphacep/vosk-android-demo

Vosk Documents with Android : https://alphacephei.com/vosk/android

## 안드로이드 Google STT API - Speech-to-text(STT) v1 with gRPC

구글 STT API를 사용합니다.

### google console & api

동영상 시청 후 콘솔 프로젝트 생성 및 api 사용 여부 체크하면 됩니다. 

(추가로, json key 파일 가지고 있어야 합니다.)

https://www.youtube.com/watch?v=SPcFViKU_xU

### gradle

google STT 의존성을 추가하면, gRPC 사용을 위해서 Stub을 만듭니다. 그래서 클래스간 충돌이 발생할 수 있습니다.

```kotlin
android {
	...
	packaging {
	  resources {
	    excludes += "META-INF/INDEX.LIST"
	    excludes += "META-INF/DEPENDENCIES"
		}
	}
}

dependencies {
	implementation("com.google.cloud:google-cloud-speech:4.50.0")
	implementation("io.grpc:grpc-okhttp:1.70.0")
}
```

### 인증

json key 파일을 가지고 있다면, assets 폴더에 배치하여 코드 레벨에서 등록 시 처리해주기 위해 추가해줍니다.

```kotlin
app
├── src
│   └── main
│       └── assets
│           └── speech-quickstart.json  (구글 인증 json key)
└── build.gradle.kts
```

### 코드

주석으로 설명합니다.

```kotlin
suspend fun startSTTInfiniteStreaming(
    onPartialResult: (String) -> Unit,
    onFinalResult: (String) -> Unit,
) {

    // 1) gRPC 채널 생성 - Google STT 서버와의 연결 (OkHttp 기반)
    val channel = OkHttpChannelBuilder
        .forAddress(HOST, PORT)
        .useTransportSecurity()
        .build()

    // 2) 인증 (Service Account JSON) - 애셋에서 파일 열기
    val inputStream = context.assets.open("speech-quickstart.json")
    val credentials = ServiceAccountCredentials.fromStream(inputStream)

    // 3) SpeechStubSettings - gRPC Stt Stub 세팅
    val speechStubSettings = SpeechStubSettings.newBuilder()
        .setTransportChannelProvider(
            FixedTransportChannelProvider.create(
                GrpcTransportChannel.create(
                    channel
                )
            )
        )
        .setCredentialsProvider(FixedCredentialsProvider.create(credentials))
        .build()

    // 4) 실제 gRPC STT 호출을 위한 Stub 생성
    val speechStub = GrpcSpeechStub.create(speechStubSettings)

    // 녹음에 필요한 버퍼 사이즈 계산
    val minBufSize = AudioRecord.getMinBufferSize(
        SAMPLE_RATE,
        AudioFormat.CHANNEL_IN_MONO,
        AudioFormat.ENCODING_PCM_16BIT
    )
    if (minBufSize <= 0) return

    // 마이크 녹음 가능 여부 확인
    if (checkRecordAudioPermission(context)) return

    // 5) AudioRecord 생성 (마이크로부터 PCM 16bit, mono, 16kHz)
    val audioRecord = AudioRecord(
        MediaRecorder.AudioSource.MIC,
        SAMPLE_RATE,
        AudioFormat.CHANNEL_IN_MONO,
        AudioFormat.ENCODING_PCM_16BIT,
        minBufSize
    )

    // AudioRecord가 정상 초기화되었는지 확인
    if (audioRecord.state != AudioRecord.STATE_INITIALIZED) return

    // 6) 마이크 녹음 시작
    audioRecord.startRecording()

		try {
		    // outerLoop: 무한 반복하여 STT 세션을 5분마다 재시작(Infinite streaming)
		    outerLoop@ while (true) {
		        // responseObserver: Google STT로부터 받은 응답(부분/최종 인식 결과)을 처리
		        val responseObserver = object : ResponseObserver<StreamingRecognizeResponse> {
		            override fun onStart(controller: StreamController) {
		                Log.e("LOG", "onStart")
		            }
		
		            override fun onResponse(value: StreamingRecognizeResponse) {
		                // STT 서버에서 받은 응답에 인식 결과(result)가 있는지 체크
		                if (value.resultsCount > 0) {
		                    val result = value.resultsList[0]
		                    // 대안(Alternative) 중 첫 번째 (가장 확률이 높은 결과)
		                    if (result.alternativesCount > 0) {
		                        val alt = result.alternativesList[0]
		                        val transcript = alt.transcript
		                        // 최종 결과 여부
		                        if (!result.isFinal) {
		                            // 부분 인식 결과 -> 콜백
		                            onPartialResult(transcript)
		                        } else {
		                            // 최종 인식 결과 -> 콜백
		                            onFinalResult(transcript)
		                        }
		                    }
		                }
		            }
		
		            override fun onError(t: Throwable) {
		                Log.e("LOG", "onError : ${t.message}")
		
		            }
		
		            override fun onComplete() {
		                Log.e("LOG", "onComplete ")
		
		            }
		        }
		
		        // 7) gRPC 호출 준비 - streamingRecognizeCallable()로 양방향 스트리밍 세션 생성
		        val callable = speechStub.streamingRecognizeCallable()
		        val clientStream = callable.splitCall(responseObserver)
		
		        // 8) RecognitionConfig: 입력 오디오 형식, 언어 등 설정
		        val recognitionConfig = RecognitionConfig.newBuilder()
		            .setEncoding(RecognitionConfig.AudioEncoding.LINEAR16)
		            .setSampleRateHertz(SAMPLE_RATE)
		            .setLanguageCode(LANGUAGE)
		            .build()
		
		        // 9) StreamingRecognitionConfig: 스트리밍 시 InterimResults(부분 인식) 받을지 여부 등
		        val streamingConfig = StreamingRecognitionConfig.newBuilder()
		            .setConfig(recognitionConfig)
		            .setInterimResults(true)
		            .build()
		
		        // 10) 첫 요청에는 audioContent 대신 STT 설정(StreamingConfig)만 전송
		        val streamingRequest = StreamingRecognizeRequest.newBuilder()
		            .setStreamingConfig(streamingConfig)
		            .build()
		
		        // config request 전송(스트리밍 설정)
		        clientStream.send(streamingRequest)
		
		        val sessionStart = System.currentTimeMillis()
		        // 내부 while: 5분 제한 (STREAMING_LIMIT_MS) 이전까지 마이크 데이터를 서버로 전송
		        while (true) {
		            val elapsed = System.currentTimeMillis() - sessionStart
		            if (elapsed >= STREAMING_LIMIT_MS) {
		                // 5분이 지나면 closeSend()로 이번 세션 종료 -> outerLoop에서 재시작
		                clientStream.closeSend()
		                break
		            }
		            // 마이크에서 읽은 PCM 오디오를 서버에 전송
		            val buffer = ByteArray(minBufSize)
		            val readSize = audioRecord.read(buffer, 0, buffer.size)
		            if (readSize > 0) {
		
		                // gRPC 요청 생성 (AudioContent에 녹음 데이터 담기)
		                val audioData = ByteString.copyFrom(buffer, 0, readSize)
		                val audioReq = StreamingRecognizeRequest.newBuilder()
		                    .setAudioContent(audioData)
		                    .build()
		                // 서버에 전송
		                clientStream.send(audioReq)
		            }
		        }
		    }
	    } finally {
	        // 끝나면 자원 정리
	        audioRecord.stop()
	        audioRecord.release() // 마이크 해제
	        speechStub.close() // STT stub 종료
	        channel.shutdownNow() // gRPC 채널 종료
	    }
	}

private fun checkRecordAudioPermission(context: Context): Boolean {
    return ActivityCompat.checkSelfPermission(
        context,
        Manifest.permission.RECORD_AUDIO
    ) == PackageManager.PERMISSION_DENIED
}

companion object {
    // 구글 STT 호스트와 포트
    private const val HOST = "speech.googleapis.com"
    private const val PORT = 443

    // 오디오 샘플링 레이트, 스트리밍 타임아웃(5분)
    private const val SAMPLE_RATE = 16000
    private const val STREAMING_LIMIT_MS = 290000L

    // 음성 인식 언어 (한국어)
    private const val LANGUAGE = "ko-KR"
}
```

### **장점**

- **높은 정확도 & 클라우드 기반**
    - 구글이 제공하는 최신 음성인식 모델을 사용, 언어 지원이 풍부하고 정확도가 높음
- **실시간 스트리밍**
    - gRPC 스트리밍으로 **부분 인식**을 빠르게 받아볼 수 있고,
    - 5분마다 세션만 재시작해 주면 **장시간** 사용 가능
- **유연한 설정**
    - 샘플레이트, 언어코드, 모델, InterimResults 등 다양한 파라미터를 조절 가능
    - `SpeechStubSettings`로 인증/커스텀 호스트 등을 세밀하게 제어

### **단점**

- **비용**
    - **무료 사용량**이 제한적(매월 60분까지 등). 이를 넘으면 **유료 과금**
    - 장시간/대규모 사용 시 요금이 매우 높아질 수 있음
- **네트워크 의존**
    - 항상 인터넷이 연결되어 있어야 함
    - 오프라인 환경에서는 사용 불가능
- **복잡도**
    - gRPC 설정, 인증, 프로토콜 설정 등 초기 세팅이 다른 방법들보다 복잡
    - 구글 클라우드 콘솔/API 키 관리도 필요

### Reference

**Github Google STT Sample :** https://github.com/GoogleCloudPlatform/android-docs-samples/tree/master/speech/Speech

**Price** :  https://cloud.google.com/speech-to-text/pricing?hl=ko

**STT Request & Response** : https://cloud.google.com/speech-to-text/docs/speech-to-text-requests?hl=ko
