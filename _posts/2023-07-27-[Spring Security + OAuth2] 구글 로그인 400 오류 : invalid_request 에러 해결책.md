---
 title: "[Spring Security + OAuth2] 구글 로그인 400 오류 : invalid_request 에러 해결책"
 date: 2023-07-27
 categories: [Spring Security, OAuth2]
 ---

 ### PART 1. 400 오류 invalid_request 에러 파악

 OAuth를 사용하여 구글 로그인을 연동하고 있었습니다.

 기존에는 로컬(`http://localhost:8080/oauth2/authorization/google`)에서 오류가 발생하지 않아서 서버 개발 중에는 구글 로그인을 잘 사용할 수 있었습니다. 

 🚨그러나, 서버 개발을 마치고 AWS EC2에 배포를 하고 나서,
 AWS EC2로 연결된 도메인 페이지(ex. `https://도메인 주소/oauth2/authorization/google`)에서 구글 로그인을 하려고 시도했을 때 아래와 같은 오류가 발생하였습니다.

 <img src="https://velog.velcdn.com/images/da_na/post/c49fefb2-b84f-4355-b856-4ef0fa50a36a/image.png" width="500" height="600"/>

 이처럼 `400 오류: invalid_request`가 발생하였습니다!

 400번대의 오류는 클라이언트의 오류이기 때문에 해당 url을 호출하는 제 프로젝트 서버의 오류였습니다!

 <img src="https://velog.velcdn.com/images/da_na/post/5e9600ca-30c5-437b-b651-4bfbd8f5e1d3/image.png" width="600" height="250"/>

 사진 출처 : https://support.google.com/accounts/answer/12917337?hl=ko

 <br/>


 오류의 메시지를 자세하게 살펴보면 **구글 validation 규칙을 준수하고 있지 않기 때문에** 구글에서 로그인을 제한하고 있음을 알 수 있습니다.

 <br/>

 그러나, 로컬에서는 잘 작동하는데 왜 도메인 페이지에서 안되는지 파악해야 했습니다!

 따라서 오류 세부정보를 보면서 파악하려고 했습니다!

 <img src="https://velog.velcdn.com/images/da_na/post/85b9873d-0e59-4e43-88f2-8b393c993413/image.png" width="500" height="600"/>

 이때, 확인해보니까 redirect_uri가 https가 아니라 http임을 확인할 수 있었습니다.

 🚨 **구글 validation 규칙에서는 redirect_uri에는 반드시 localhost만 http를 사용할 수 있고 다른 도메인에서는 https를 사용하면 안된다고 합니다!!**

 따라서 redirect_uri을 `http://도메인 주소/login/oauth2/code/google`에서 `https://도메인 주소/login/oauth2/code/google`로 변경해주어야 합니다.

 ---

 ### PART 2. 오류 해결하기

 google console에서 OAuth 설정 관련 부분에서 `https://도메인 주소/login/oauth2/code/google`을 사용했는데 어디에도 선언하지 않은 `http://도메인 주소/login/oauth2/code/google`를 호출하고 있는지 알아야 했습니다.

 ![](https://velog.velcdn.com/images/da_na/post/1692d64a-8a7a-41eb-8301-b7d03d88f62d/image.png)

 현재 프로젝트에서는 Spring security에서 구글 로그인을 연결하고 있기 때문에 Spring security 코드에서 redirect_uri를 설정하는 코드를 찾아보았습니다.

 하지만, redirect_uri를 설정하는 코드를 따로 설정하고 있지 않아도 Spring security 자체적으로 구현되어 있어서 `{baseurl}/login/oauth2/code/google`가 되어 있습니다. 

 ![](https://velog.velcdn.com/images/da_na/post/a7184ce9-4402-4dad-a6b8-ef5ad10b5a32/image.png)
 사진 출처 : https://docs.spring.io/spring-security/site/docs/5.2.12.RELEASE/reference/html/oauth2.html

 현재 사용하고 있는 프로젝트의 EC2의 baseurl 자체가 http를 사용하고 있어서 `http://도메인 주소/login/oauth2/code/google`가 되어 있었습니다.

 따라서 Spring security 코드에서 redirect_uri를 설정하는 코드를 security에서 제공해주는 형식이 아닌 새로운 redirect_uri를 직접 설정해주기로 하였습니다.

 Spring security를 설정해주는 yml 파일에서 `redirect-uri : https://{baseHost}{basePort}/login/oauth2/code/google`를 추가하여 직접 명시하여 코드를 변경한 뒤에 다시 EC2에 배포하였습니다!

 - 이전 코드

 ```yml
 spring:
   security:
     oauth2:
       client:
         registration:
           google:
             client-id: {구글 콘솔에서 제공받은 구글 oauth 클라이언트 아이디}
             client-secret: {구글 콘솔에서 제공받은 구글 oauth 클라이언트 비밀번호}
             scope: profile,email
 ```

 - 변경한 코드
 ```yml
 spring:
   security:
     oauth2:
       client:
         registration:
           google:
             client-id: {구글 콘솔에서 제공받은 구글 oauth 클라이언트 아이디}
             client-secret: {구글 콘솔에서 제공받은 구글 oauth 클라이언트 비밀번호}
             redirect-uri : https://{baseHost}{basePort}/login/oauth2/code/google
             scope: profile,email
 ```

 ✨ 아래의 사진처럼 성공적으로 도메인 페이지에서 구글 로그인을 할 수 있었습니다!!
 <img src="https://velog.velcdn.com/images/da_na/post/afea560f-d2ee-41d7-8047-fdcf44f67a0c/image.png" width="500" height="600"/>
