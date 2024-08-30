---
title: Firebase를 이용하여 웹 알림 구현해보기 - 서버(1)
date: YYYY-MM-DD HH:MM:SS +09:00
categories: [백엔드, 상돌] 
tags:
  [
    웹 알림, Firebase, 서버 구현
  ]
---

안녕하세요. 모우다 팀의 상돌(이상진) 입니다 😄. 드디어 모우다 서비스에서 **웹 알림 기능**이 구현되었습니다!👏👏

웹 푸시 알림은 **Firebase Cloud Messaging**을 사용하였는데요, 공식 문서가 꽤나 친절하게 작성되어있어 처음 사용했음에도 생각보다 빠르게 구현할 수 있었는데요, 이번 글에서는 전체 과정에 대해 **1. 개요 2. 서버에서의 구현 순서**로 전체 과정을 공유해 보겠습니다.

사용자 등록 및 개념적인 내용은 [공식 문서](https://firebase.google.com/docs/cloud-messaging?hl=ko)에 자세히 설명되어 있기에, 이번 글에서는 **실제 구현 과정 위주**로 작성해볼게요!

### 시작하기 전에..

중복이 될 수도 있는 용어를 미리 정리하고 시작할게요.

- **서버** : **우리 서버(모우다 서버)**
- **Firebase Cloud Messaging** : **FCM**
- **Firebase Cloud Messaging 서버** : **FCM 서버**

이제 FCM의 작동 과정부터 시작해볼게요!

<br>

### FCM의 작동 과정

FCM의 작동 순서를 간단하게 정리해볼게요. [공식 유튜브](https://youtu.be/sioEY4tWmLI?feature=shared)에 올라온 영상을 보셔도 됩니다!(1분밖에 안해요!)

1. 사용자를 FCM 서버에 등록한다.
  - 사용자는 **FCM 토큰**으로 식별합니다.
2. 서버에서는 FCM 토큰을 이용하여 FCM 서버에게 알림 전송을 지시한다.
  - 즉, 서버에서는 사용자와 토큰을 저장할 필요가 있겠네요!

<br>

FCM을 이용하여 알림을 보내는 과정은 위 처럼 간단하지만, 우리 서비스에서 진행하는 과정으로 조금 더 자세히 풀어보겠습니다.

1. 회원이 로그인을 한다.
2. 알림 허용 / 거부를 물어본다.
  - 지금은, 메인 페이지의 알림 아이콘을 누르면 허용 / 거부를 물어보고 있습니다!
3. 사용자가 허용을 누르면 클라이언트(프론트엔드)에서 Firebase에 토큰을 요청하고 받아옵니다.
4. 클라이언트가 회원 정보와 발급받은 FCM 토큰을 담아 미리 만들어진 서버의 API로 POST 요청을 보냅니다.
5. **서버는 회원 정보와 토큰을 DB에 저장**합니다.
6. 알림을 보낼 때는 **회원 정보를 이용하여 DB에 있는 토큰을 찾고**, 이 토큰을 이용하여 FCM 서버에 알림 전송을 요청합니다.
7. FCM 서버는 요청을 받아 사용자에게 알림을 보냅니다.

<br>

즉, 클라이언트는 FCM 토큰을 받아 서버에게 저장 요청만 해주면 되고, 서버는 저장된 토큰을 이용하여 메시지를 보내기만 하면 되는거에요! 클라이언트 구현 과정은 치코가 잘 작성해줄테니 아래부터는 서버단에서의 처리 과정을 작성해볼게요.

<br>

## FCM 인증 - 비공개 키 읽기

FCM 서버를 사용하려면 계정 인증 과정이 필요합니다. 인증 방법은 다양하게 있지만, 저희는 비공개 키(json)을 이용하고 있습니다. 비공개 키는 FirebaseApp을 실행할 때 사용되고, 저희는 `@PostConstruct` 를 이용하여 스프링의 의존성 주입이 완료된 이후에 실행하고 있습니다!

비공개 키는 프로젝트 설정 - 서비스 계정에서 생성할 수 있고, 생성된 json 파일을 프로젝트 경로로 옮깁니다. 저는 `src/main/resources/firebase/serviceAccountKey.json` 으로 저장했어요!

```java
@Configuration
public class FirebaseConfig {

    @PostConstruct
    public void init() {
      try {
        InputStream serviceAccount = getClass().getClassLoader()
          .getResourceAsStream("firebase/serviceAccountKey.json");
        FirebaseOptions options = new FirebaseOptions.Builder()
          .setCredentials(GoogleCredentials.fromStream(serviceAccount))
          .build();
        FirebaseApp.initializeApp(options);
      } catch (Exception e) {
        e.printStackTrace();
      }
    }
}
```

전체적인 코드는 공식 문서와 동일한데 json 파일을 불러오는 방법만 다릅니다!

간단하게만 적어보자면, 공식 문서에는 FileInputStream이 적혀있는데, 로컬에서 돌릴 때와 JAR로 배포하는 상황에서 파일이 저장되는 방식이 다르기 때문에 FileInputStream을 사용했을 때 배포 환경에서는 파일을 읽어오지 못했어요.

```java
@Profile("local")
private InputStream getServiceAccountLocal() {
    try {
      return new FileInputStream("src/main/resources/firebase/serviceAccountKey.json");
    } catch (Exception e) {
      e.printStackTrace();
      return null;
    }
}

@Profile("dev, prod")
private InputStream getServiceAccountDeploy() {
    try {
      return getClass().getClassLoader().getResourceAsStream("firebase/serviceAccountKey.json");
    } catch (Exception e) {
      e.printStackTrace();
      return null;
    }
}
```

물론 위 코드처럼 `@Profile`을 이용하여 처리할 수도 있겠지만, classpath에서 찾는 코드만 작성해도 로컬과 배포 환경에서 모두 잘 작동하기에 저희는 InputStream을 이용하는 방식으로 수정했습니다.

물론, 비공개 키는 당연히 공개 저장소에 공개되면 안 되기 때문에 github에는 올라가있지 않습니다. 휴먼에러 방지를 위해 `.gitignore`  설정을 미리 해두는 것을 추천드립니다!

- git action secret를 이용해서 관리할 수도 있고, submodule을 이용할 수도 있지만 저희는 ec2 내부에 넣고 사용하고 있어요.

<br>

## 메시지 전송하기

### 1. 개요

자바 코드를 이용하여 FCM 알림 전송을 요청하는 과정은 다음 순서로 이루어집니다.

1. Firebase에서 제공하는 `Message` 객체를 만든다.
2. `FirebaseMessaging.getInstance().send()` 에 1에서 만든 객체를 담아 호출한다.

물론 Message 외에도 다른 방법들이 있지만, 이해를 위해 Message 객체를 예시로 코드를 작성해볼게요.

### 2. Message 객체

[공식 문서](https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages?hl=ko)의 내용을 보면, Message는 아래와 같은 형식으로 이루어져 있습니다.

```json
{
  "name": "name",
  "data": {
    "string: string,",
    "key": "value"
  },
  "notification": {
    "object (Notification)"
  },
  "android": {
    "object (AndroidConfig)"
  },
  "webpush": {
    "object (WebpushConfig)"
  },
  "apns": {
    "object (ApnsConfig)"
  },
  "fcm_options": {
    "object (FcmOptions)"
  },

  "token": "string",
  "topic": "string",
  "condition": "string"
}
```

하나의 메시지에는 이름, 데이터, 알림, 토큰 및 기타 설정이 들어가 있습니다. 저희는 일단 notification과 token만을 사용해서 데이터를 보내볼텐데, 이를 자바 코드로 구현하면 다음과 같습니다.

```java
Message message = Message.builder()
	.setToken("fcmtoken")
	.setNotification(-Notification 객체-)
	.build();
```

- setToken에 한 개의 토큰만 넣으면 다중 전송은 어떻게 하지? 라는 생각이 들 수 있는데 다음 편에 작성됩니다 ㅎㅎ

Firebase에서 제공하는 객체들은, 빌더를 이용하여 생성하도록 되어있어요. 예를 들어 위 json에 있는 “android” 옵션은 setAndroidConfig()로 설정할 수 있습니다.

다음으로는 setNotification()에 들어갈 Notification에 대해 알아볼게요.

### 3. Notification 객체

Notification은 공식 문서에 따르면, **모든 플랫폼에서 사용할 기본 알림 템플릿** 입니다.

물론, android, web 등의 각 옵션 내에도 Notification을 지정할 수 있어요.(예를 들어, setAndroidConfig에 지정하는 Notification은 AndroidNotification으로 정의되어 있어요)

이번에는 구체적인 Notification이 아닌 기본 Notification에 대해 살펴볼게요.

```java
{
  "title": string,
  "body": string,
  "image": string
}
```

구조는 위와 같이 제목, 내용, 이미지로 되어있습니다. image는 알림에 표시될 이미지의 URL 경로를 넣으면 되는데 우선 이번에는 title과 body만 채워볼게요.

```java
Notification notification = Notification.builder()
    .setTitle("Portugal vs. Denmark")
    .setBody("great match!")
    .build();
```

Notification 객체 역시 빌더를 이용하여 생성할 수 있어요.

### 4. 전송하기

위에서 정의한 Notification과 Message 객체를 이용하여 알림 전송을 하는 코드를 작성해볼게요.

```java
Notification notification = Notification.builder()
    .setTitle("Portugal vs. Denmark")
    .setBody("great match!")
    .build();
			
Message message = Message.builder()
    .setToken("fcmToken")
    .setNotification(notification)
    .build();
		
String response = FirebaseMessaging.getInstance().send(message);
```

`FirebaseMessaging.getInstance().send()` 로 메시지를 보내게 되면, FCM 서버에는 아래 형식의 JSON으로 전송됩니다.

```java
{
  "message":{
    "token":"fcmtoken",
    "notification":{
      "title":"Portugal vs. Denmark",
      "body":"great match!"
    }
  }
}
```

그리고 FCM 서버가 입력된 토큰을 가진 사용자에게 알림 전송에 성공하면 `projects/{project_id}/messages/{message_id}` 형식의 문자열 응답을 보내주게 됩니다.

### 요약

1. Message 객체에 들어가는 Notification 등의 객체 역시 빌더로 만든다.
2. Message의 형식에 맞춰 빌더로 Message 객체를 만든다.
3. Message 객체를 FirebaseMessaging.getInstance().send()에 담아 전송한다.

메시지 전송은 위와 같이 간단하게 구현할 수 있습니다. 이제 서버에서 실제로 알림 기능을 구현한 과정을 작성해볼텐데, 내용이 길어질 것 같아 여기서 한번 끊고 갈게요.

더 자세한 내용은 중간중간에 첨부된 공식 문서를 확인해주세요. 감사합니다😄
