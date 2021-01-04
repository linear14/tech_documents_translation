# 외부 서버를 사용하지 않고 Device-to-Device Notification을 전송하기 (FCM 활용)

## 글의 저작권 문제가 있는 경우

저작권 문제가 있는 경우, 아래의 이메일로 연락주세요.  
If there is copyright issues, please contact me by email below.

📧 linear114.andro@gmail.com

<br>

## 참고

- [원문 링크](https://medium.com/mindorks/send-device-to-device-push-notification-using-firebase-cloud-messaging-without-using-external-769476c79ffd)

<br>

## 서론

Firebase Cloud Messaging(이하 FCM)은 비용없이 안전하게 메세지를 전달해주는 크로스 플랫폼이다.  
Firebase Console에서 push notification을 바로 날릴 수도 있고, 서버를 통해 이러한 작업을 수행할 수도 있다.  
이 글은 외부 서버를 사용하지 않고 디바이스끼리 push notification을 전송하는 방법을 다룰 것이다. 마치, 대부분의 채팅 앱들이 사용하는 것처럼 말이다.

<br>

## Step1 : 새로운 프로젝트 생성 및 의존성 추가

우선, 새로운 안드로이드 스튜디오 프로젝트를 만들어야한다. 그리고 관련된 의존성을 추가해줘야 한다.

[링크](https://firebase.google.com/docs/android/setup)에 있는 튜토리얼에 따라 Firebase 설정을 진행해준다.  
FCM을 사용하기 위해, 아래와 같은 의존성을 프로젝트에 추가해준다.
```
dependencies {
    implementation 'com.google.firebase:firebase-messaging:19.0.1'
}
```

<br>

## Step2 : 파이어베이스 Service 생성 

`MyFirebaseMessagingService` 클래스를 만들어준다.  
위의 서비스는 전송된 notification을 받아내고 화면에 보여주는데 사용될 것이다.  
<br>

서비스란 어떠한 시각적 인터페이스도 보이지 않으며, 백그라운드 환경에서 동작하도록 구성된다.  
app 폴더에서 마우스 우측클릭을 통해 <b>New -> Service -> Service</b> 순서로 서비스를 생성한다.  
이후, 서비스를 다룰 클래스네임을 작성 후, <b>Finish</b>를 눌러 완료한다.  
<br>

`AndroidManifest.xml`에서 `application`태그 내에 만들어진 서비스에 대한 선언을 해준다.  
또한, `INTERNET` 및 `CLOUD TO DEVICE MESSAGING` 권한 허용 작업을 처리해주어 FCM 서버와 상호작용을 할 수 있도록 만들어준다.

```xml
<manifest ...>
    <uses-permission android:name="android.permission.INTERNET" />
          
    <application
            ...>
        <service
                android:name=".MyFirebaseMessagingService"
                android:permission="com.google.android.c2dm.permission.SEND">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
                <action android:name="com.google.android.c2dm.intent.RECEIVE" />
            </intent-filter>
        </service>

        <activity android:name=".MainActivity">
            ...
        </activity>
    </application>

</manifest>

```

<br>

## Step3 : 서비스 설정

`MyFirebaseMessagingService`는 `FirebaseMessagingService` 클래스를 상속한다. 이를 통해, FCM 서버로부터 메세지를 받을 수 있다. 왜냐하면 이 서비스는 notification을 받아들이며, 화면에 표시하는 작업을 관리하기 때문이다.  
<br>

`onMessageReceived()` 함수의 오버라이딩은, 새로운 메세지가 들어올 때 마다 이 함수가 호출되도록 하며, 이후의 작업을 처리할 수 있도록 한다.

```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onMessageReceived(remoteMessage: RemoteMessage?) {
        super.onMessageReceived(remoteMessage) 

        val intent = Intent(this, MainActivity::class.java)

        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent,
            PendingIntent.FLAG_ONE_SHOT
        )

        val notificationBuilder = NotificationCompat.Builder(this, ADMIN_CHANNEL_ID)
            .setSmallIcon(/* resource */)
            .setLargeIcon(/* resource */)
            .setContentTitle(remoteMessage?.data?.get("title"))
            .setContentText(remoteMessage?.data?.get("message"))
            .setAutoCancel(true)
            .setSound(/* resource */)
            .setContentIntent(pendingIntent)


        val notificationManager = getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        val notificationID = Random().nextInt(3000)
        notificationManager.notify(notificationID, notificationBuilder.build())
    }

}
```

<br>

## Step4 : 메세지 전송 로직 구현

우선, Firebase Console에서 `Server Key`를 얻어내야한다.

1. `Project Overview` 옆의 톱니바퀴 아이콘을 클릭한다. 이후, `Project settings`를 클릭한다.
2. `Cloud Messaging` 탭을 클릭하여 `Server Key`를 복사한다.
<br>
<br>


push notification을 전송하는 것은 <b>FCM 서버에 HTTP POST 메소드에 의한 요청</b>에 따라서만 동작한다.

<p style="font-size: 18px">URL</p>

```
https://fcm.googleapis.com/fcm/send
```

<p style="font-size: 18px">Headers</p>

```
Authorization: key="Firebase server key"  
Content-Type: application/json
```

<p style="font-size: 18px">Body</p>

```json
{
  "to": "/topics/topic_name" or "user token",
  "data": {
    "title": "Notification title",
    "message": "Notification message",
    "key1" : "value1",
    "key2" : "value2"
    ...
   }
}
```
<br>

위의 구조를 명심한다. 메세지를 서버로 보내기 위해 notificaiton body 부분의 JSONObject를 구성해야한다.  
이 객체는 수취인의 topic (혹은, 수취인의 token 값), notification 제목과 메세지, 혹은 필요한 key/value 쌍을 포함한다.

```kotlin
class MainActivity : AppCompatActivity() {

    private val FCM_API = "https://fcm.googleapis.com/fcm/send"
    private val serverKey = "key=" + "Enter your Key"
    private val contentType = "application/json"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding = DataBindingUtil.setContentView<ActivityMainBinding>(this, R.layout.activity_main)

        binding.btnSubmit.setOnClickListener {
            
            // 토픽 구독 대상을 상대로 noti를 보내는 경우
            val topic = "/topics/Enter_your_topic_name"

            val notification = JSONObject()
            val notifcationBody = JSONObject()

            try {    
                notifcationBody.put("title", "테스트 제목")
                notifcationBody.put("message", "테스트 메세지")
                notification.put("to", /* topic or 사용자 token */)
                notification.put("data", notifcationBody)
            } catch (e: JSONException) {
                Log.e("TAG", "onCreate: " + e.message)
            }

            sendNotification(notification)
            
        }
    }
}
```
<br>

FCM 서버로의 요청을 위해 `Volley`라이브러리를 사용한다.  
서버는 요청 파라미터를 사용하여 notification이 타겟 디바이스로 전송되도록 도울 것이다.

```kotlin
class MainActivity : AppCompatActivity() {

     private val requestQueue: RequestQueue by lazy {
        Volley.newRequestQueue(this.applicationContext)
    }
  
   private fun sendNotification(notification: JSONObject) {
        val jsonObjectRequest = object : JsonObjectRequest(FCM_API, notification,
            Response.Listener<JSONObject> { response ->
                // 성공시 처리 로직
            },
            Response.ErrorListener {
                // 실패시 처리 로직
            }) {

            override fun getHeaders(): Map<String, String> {
                val params = HashMap<String, String>()
                params["Authorization"] = serverKey
                params["Content-Type"] = contentType
                return params
            }
        }
        requestQueue.add(jsonObjectRequest)
    }
}

```
<br><br>

## 결론 

이러한 구성을 통해, 앱 구축이 완료되었다. 이제 별도의 서버 없이 디바이스간 notification 전송이 가능하다. 수취인의 topic 혹은 token값이 올바른지 항상 염두해두어야 하며, 잘못됐을 경우 전송은 이루어지지 않을 것이다.