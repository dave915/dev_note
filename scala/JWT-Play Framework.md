### JWT - Play Framework

## JWT?
JWT(Json Web Token)은 사용자 정보나 데이터 속성 정보를 갖고있는 클레임 기반 토큰이며, Json으로 되어있고, 웹 표준(RFC 7519)를 구현한 것이다.

헤더.내용.서명 세 부분으로 구성되어있으며 Base64로 인코딩 된다.

헤더: 타입, 해싱 알고리즘

내용: 인증에 필요한 정보, 만료시간 등(보통 세션에 담기는 정보)

서명: 인코딩된 헤더, 인코딩된 내용의 값을 비밀키로 해쉬함

자세한 내용은 https://jwt.io/ 참고

## JWT 만료시간 문제
JWT에 만료시간이 없다면 JWT만 탈취하면 유저의 권한을 무한대로 사용 가능함.

** 해결 **<br>
외부로 노출되지 않는 Refresh Token을 발급하여 Refresh Token의 만료시간은 길고, JWT 만료시간은 짧게 가져간다
  * ex) Refresh Token 30분 / JWT 만료시간 2분

Refresh Token은 상황에 맞춰서 갱신해준다.
  * ex) api 요청때마다 or 만료시간이 지정된 시간보다 적게 남았을때

## JWT Flow
* JWT 발급 요청
  * JWT 발급 요청 (발급 요청 api or 로그인)
  * JWT, Refresh Token 발급
    * Refresh Token은 외부로 노출시키면 안되기 때문에 사용자에게 넘겨주지 않고 Redis와 Mysql에 넣어둠
    * 쿠키에 JWT담아서 리턴
      * 로그인 하면서 JWT발급까지 하는 경우가 아닌 별도 api를 이용하여 발급 할 경우 response로 JWT로 넘겨주거나 response에는 성공 실패 응답, JWT는 쿠키에 담아서 보냄.
* JWT가 유효할때
  * Api 요청
  * 쿠키에 있는 JWT를 꺼내서 유효체크, 만료시간 체크
  * 모두 정상일 경우 정상 api 수행 후 응답
* JWT 만료시간이 지났을때
  * Api 요청
  * JWT 유효체크, 만료시간 체크
  * 먼저 캐시(Redis)에서 Refresh Token 조회, 없으면 DB(Mysql) 조회
  * Refresh Token 만료시간 체크 후 만료시간이 남아있으면 JWT갱신후 api응답
    * Api 발급 및 쿠키에 담는 과정을 Api Server가 아닌 Client에서 했다면, 만료되었다는 에러 보낸 후 Client에서 JWT다시 재발급 후 쿠키 갱신 해주어야함.
    * Refresh Token 만료시간이 10분 이하 남았으면 Refresh Token갱신
    * Refresh Token은 캐시에만 저장해도 되지만 혹시 캐시가 죽을수도 있어서 DB에도 저장해줌
![](/images/scala/scala_jwt.png)

## Play Framework에서 JWT 구현
play에서는 ActionBuilder를 구현하여 커스텀 액션을 만들 수 있다.

필요한 라이브러리 (build.sbt에 추가)
```sbt
"com.jason-goodwin" %% "authentikat-jwt" % "0.4.5"
```
구현
```scala

object JWTAuthentication extends ActionBuilder[UserRequest, AnyContent] {
  def invokeBlock[A](request: Request[A], block: UserRequest[A] => Future[Result]): Future[Result] = {
    // 구현
  }
}
```
사용
```scala
def addGroup = JWTAuthentication.async { implicit  request =>

}
```
