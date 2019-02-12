### Scala Future 에러처리

기존 에러 처리 코드
```scala
def save = auth.JWTAuthentication.async { implicit request =>
  val item = request.body.asJson.get.as[Item]

  try {
    itemService.save(item)
    .map( _ => Ok(Json.toJson(Map("result" -> "success"))))
  } catch {
    case e : Exception => BadRequest(Json.toJson(Map("result" -> "fail")))
  }
}
```
위 코드처럼 하면 Future객체의 처리가 끝나기 전에 try-catch 문을 벗어나기 때문에 정상적으로 에러처리를 할 수 없다.

수정후
```scala
def save = auth.JWTAuthentication.async { implicit request =>
  val item = request.body.asJson.get.as[Item]

  itemService.save(item)
    .map( _ => Ok(Json.toJson(Map("result" -> "success"))))
    .recover {
      case e: Exception => BadRequest(Json.toJson(Map("result" -> "fail")))
    }
}
```
