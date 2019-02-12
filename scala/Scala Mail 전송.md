### Scala Mail 전송

필요한 라이브러리 (build.sbt에 추가)
```sbt
"com.typesafe.play" %% "play-mailer" % "6.0.1"
"com.typesafe.play" %% "play-mailer-guice" % "6.0.1"
```

application.conf
```
play.mailer {
  host = "smtp.gmail.com"
  port = 465
  ssl = yes
  user = "Htaehyun@gmail.com"
  password = “password"
}
```

MailUtil
```scala
@Singleton
trait SendMail {
  def sendMail(to: Seq[String], subject: String, message: String)
}

class MailUtils @Inject()(mailerClient: MailerClient) extends SendMail {
  override def sendMail(to: Seq[String], subject: String, message: String) = {
    val email = Email(
      from = s"taehyun <Htaehyun@gmail.com>",
      to = to,
      subject = subject,
      bodyHtml = Some(message)
    )
    mailerClient.send(email)
  }
}
```
Util 사용
```scala
mailUtils.sendMail(Seq(info._1.email.get), GROUP_INVITE_SUBJECT, views.html.groupInvite(info._2.name.get, inviteConfirmLink).toString())
```
