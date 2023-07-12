---
title: "[크롬 익스텐션] OAuth2 로그인 구현하기 (JWT + OAuth2 + Spring Security)"
date: 2023-06-30
---

## PART 1. 크롬 익스텐션 로그인 구현 로직 결정하기
OAuth2를 이용하여 로그인을 구현하는 서버 구조는 크게 3가지로 나누어질 수 있습니다.
- 프론트에서 처리할지 아니면 백엔드에서 처리할지 아니면 두개를 혼합해서 처리할 지 3가지로 나누어집니다.

1) **프론트**에서 모든 인증 과정을 수행
- 프론트에서 로그인 요청을 처리하는 방식

2) **백엔드**에서 모든 인증 과정을 수행

3) **프론트 + 백엔드** 혼합으로 인증 과정을 수행
- 프론트에서 백엔드에게 로그인 요청을 보내고, 백엔드에서 로그인 요청을 처리하는 방식

<br/>

처음에는 프론트(크롬 익스텐션)에서 모든 인증 과정을 수행하려고 했으나, 다음의 이유로 프론트+백엔드을 선택하게 되었습니다.

- 프론트에서 인증 과정 수행하는 과정 
➡️ 크롬 익스텐션 OAuth2 관련 라이브러리 : chorme.identity

<img src="https://velog.velcdn.com/images/da_na/post/2e6485c1-a3ac-4ada-aa11-b682231df9e9/image.png" width="600" height="150"/>

출처 : https://developer.chrome.com/docs/extensions/reference/identity/

<br/>

### 1가지 이유 : 크롬 익스텐션 OAuth2 관련 자료가 많지 않다. 🏷️
- 작년 1월부터 크롬 익스텐션 관련 버전이 Manifest V2에서 V3로 변경하라는 공지가 나왔습니다. 현재는 크롬 익스텐션을 V3로만 개발해야 했으며, 이전에 개발된 크롬 익스텐션은 V2가 더 많았습니다. 따라서 관련 자료가 V2 위주로 되어 있어서, 구현 과정에서 에러가 많이 발생했습니다.
<img src="https://velog.velcdn.com/images/da_na/post/58b60a49-8df0-44a2-a241-2c7786d3013b/image.png" width="600" height="150"/>
출처 : https://developer.chrome.com/docs/extensions/mv3/intro/

- 크롬 익스텐션 자료는 공식 문서와 여러 외국 자료가 많았지만, 한국 관련 문서가 많지 않았습니다. 현재 구현하려는 서비스는 한국인을 대상으로 하기 때문에 카카오, 네이버, 구글 로그인을 모두를 구현해야 해서 구글 로그인을 위주로 되어 있는 크롬 익스텐션 OAuth2 관련 라이브러리를 통해서 구현하기 어려웠습니다.


### 2가지 이유 : 보안상의 이유 👮‍♀️
- 로그인은 사용자의 정보와 관련된 사항이기 때문에 보안에 민감한 작업입니다. 따라서 보안 처리를 해줄 수 있는 백엔드를 사용하는 것이 좋습니다. 
➡️ 백엔드 Spring에서는 인증 절차를 간편하게 해주는 Spring Security를 사용하면, 웹 공격으로 부터 안전하게 지켜줄 수 있는 작업을 수행해줍니다.
 (추후, Spring Security 관련 글에서 더 자세하게 이야기 하겠습니다.)

### 3가지 이유 : 중복된 로그인 구현 과정 👥
- 현재 구현하고 있는 서비스는 '크롬 익스텐션'과 '웹 사이트'로 두 가지의 유형의 서비스에서 로그인을 구현해야 합니다. 프론트로 구현하게 될 경우, 두 가지 서비스 모두 로그인을 구현해야 합니다. 그러나, 백엔드로 구현하게 될 경우, 백엔드에서만 로그인을 구현하고 프론트에서 API를 호출하는 형식으로 진행되어 1가지에만 구현하면 됩니다.

### 4가지 이유 : 비즈니스 로직 분리 🗂️
- 로그인은 사용자 인증에 관련된 비즈니스 로직입니다. 백엔드에서 로그인 로직을 처리하게 되면, 프론트엔드는 비즈니스 로직에 신경 쓰지 않고, 로그인을 호출할 수 있습니다. 따라서, 코드의 유지 보수와 테스트에도 도움이 됩니다.

### 5가지 이유 : 토큰 관리 🗝️
- 백엔드에서 액세스 토큰을 발급하고 관리하면, 토큰의 보안과 유효성 검사를 책임질 수 있습니다. 프론트엔드에서는 단지 토큰을 전달하고 사용하기만 하면 되며, 토큰의 안전성에 대해 걱정할 필요가 없습니다.

<br/>

## PART 2. 소셜 로그인 로직 정리하기
소셜 로그인 로직을 정리하기 전에, 어떤 기술을 사용할 것인지 살펴보겠습니다.
- 프론트는 '크롬 익스텐션(바닐라 자바 스크립트)'와 '웹 사이트(React)'를 사용할 예정입니다.
- 백엔드는 스프링과 Spring Security을 사용할 것 입니다.
- 토큰 관리는 JWT를 사용합니다.

➡️ React + JWT + OAuth2 + Spring Security
➡️ Chrome Extension(JavaScript) + JWT + OAuth2 + Spring Security

![](https://velog.velcdn.com/images/da_na/post/a34dde67-4272-4252-adc3-bbfe5519dd4b/image.jpg)


✨ 소셜 로그인 로직 정리를 위의 사진으로 총 정리하였습니다.

1. 사용자는 웹 사이트나 크롬 익스텐션 팝업 창에서 로그인을 하기 위해서, 소셜 로그인 버튼을 누르게 됩니다.
2. 프론트에서는 해당 버튼이 누리는 행동을 인식하여, 'https://{서비스 도메인 주소}/oauth2/authorization/{provide-id}'를 호출합니다.

```
//로컬 서버에서 구글 로그인 버튼 클릭시
https://localhost:8080/oauth2/authorization/google

//로컬 서버에서 네이버 로그인 버튼 클릭시
https://localhost:8080/oauth2/authorization/naver
```

3. 백엔드에서는 사용자를 리다이렉트 URI로 인가 코드를 요청하는 페이지(소셜 로그인하는 페이지)로 리다이렉트 시켜줍니다.


```
//카카오 예시
https://kauth.kakao.com/oauth/authorize
?client_id=${kakao.clientID}
&redirect_uri=${kakao에 등록한 redirectUri}
&response_type=code
```

4-5. 사용자는 아래의 사진과 같이 소셜 로그인하는 페이지(구글 계정으로 로그인하는 창)로 이동하게 되고, 소셜 계정으로 로그인을 완료합니다.

<img src="https://velog.velcdn.com/images/da_na/post/bf4b5528-cbc9-46e9-a849-2f0d4e6e4b88/image.png" width="600" height="700"/>

6. Google API Server는 해당 클라이언트 ID의 리다렉션 URI에 인가코드(Authorization code)를 제공해줍니다.

   <img src="https://velog.velcdn.com/images/da_na/post/0644050e-1525-4b10-a36f-bdcb3c216d98/image.png" width="600" height="250"/>
   
- 위의 사진은 구글 로그인 OAuth2.0 계정 생성시 입력한 리디렉션 URI입니다.
7. 백엔드는 제공받은 인가코드를 사용하여 Access 토큰을 요청합니다.
8. Google API Server는 인가 코드를 확인하고, Access 토큰를 발급해서 백엔드에 전달해줍니다.
9. 백엔드는 전달받은 Access 토큰을 사용하여 구글 계정의 유저 정보를 요청합니다.
10.  Google API Server는 Access 토큰를 확인하고, 구글 계정의 유저 정보를 백엔드에 전달해줍니다.

11-13. 백엔드는 받아온 유저 정보가 DB에 있는지 체크하여 없다면 DB에 넣고 있다면 따로 넣어주지는 않습니다. 
- 이때, 없다면 DB에 넣는 과정은 회원가입 과정이고 있다면 기존 로그인 과정입니다. 따라서, 회원가입 버튼은 생략되고 로그인 버튼 하나로 회원가입과 로그인을 모두 구현할 수 있습니다.

14-15. 백엔드 서버에서 자체 토큰(JWT)와 토큰 만료 시 재발급을 도와주는 refresh 토큰을 생성 및 저장하고, 프론트에게 JWT와 Refresh 토큰을 전달해줍니다.

16. 프론트는 전달받은 JWT와 Refresh 토큰을 스토리지(쿠키/로컬 스토리지)에 저장합니다. 
17. 프론트는 저장한 JWT를 사용해서 API를 호출합니다.

18-19. 백엔드는 JWT를 확인하고, 유효성 검사를 마치고 만료되지 않은 JWT인 경우 프론트에게 해당 사용자의 정보를 제공해줍니다.

<br>

## 참고자료
참고 자료 1
https://velog.io/@codesusuzz/React-Spring-Boot-JWT-소셜-로그인-정복기-1

참고 자료 2
https://velog.io/@jkijki12/Spring-Boot-OAuth2-JWT-적용해보리기
