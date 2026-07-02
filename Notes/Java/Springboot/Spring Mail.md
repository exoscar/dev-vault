###  `spring-boot-starter-mail`
This is the dependency that brings email support into Spring boot.
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

Internally it pulls: 
- Spring Mail
- Jakarta Mail(JavaMail)
- Java Activation Framework

### Email Configuration
```
spring:
  mail:
    host: smtp.gmail.com                       -- SMTP Server
    port: 587                                  -- SMTP Port
    username: your-email@gmail.com             -- Sender Email
    password: app-password                     -- App Password
    properties:                  
      mail:
        smtp:
          auth: true                           -- Authentication
          starttls:                            -- Encrypt Connection  
            enable: true
```

### JavaMailSender
`JavaMailSender` is a Spring Framework interface used to send emails from Java applications. It extends the standard JavaMail API and simplifies sending simple text emails, HTML emails, and emails with attachments.

```
@RequiredArgsConstructor
@Service
public class EmailService {
    private final JavaMailSender mailSender;
}
```


### SimpleMailMessage
`SimpleMailMessage` is a Spring class for sending **plain text emails**. It is lightweight
No HTML,Attachements or Inline images

```
SimpleMailMessage message =
        new SimpleMailMessage();

message.setTo("user@example.com");
message.setSubject("Welcome");
message.setText("Welcome to DevSync");

mailSender.send(message);
```
Email received
```
Subject: Welcome
Welcome to DevSync
```

When to use
Good for 
- Password reset
- OTP
- Verification Code
- Account Alerts

### MimeMessage
`MimeMessage` is part of the **Jakarta Mail API** (`jakarta.mail.internet.MimeMessage`). It represents a full MIME email and supports rich content such as HTML, attachments, inline images, multiple recipients, and multipart messages.

In Spring Boot, you typically create it through `JavaMailSender` and use `MimeMessageHelper` rather than interacting with `MimeMessage` directly.

```
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;

    public void sendHtmlEmail() throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();

        MimeMessageHelper helper = new MimeMessageHelper(message,true);

        helper.setFrom("noreply@example.com");
        helper.setTo("user@example.com");
        helper.setSubject("Welcome");
        helper.setText(
                "<h2>Welcome!</h2><p>Your account has been created.</p>",
                true // HTML
        );
        helper.addAttachment(  
				"report.pdf",  
				 new FileSystemResource(new File("report.pdf"))
		helper.setText("""  
			<h1>Hello</h1>  
			<img src='cid:logo'>  
			""", true);  
  
		helper.addInline(  
		"logo",  
		new ClassPathResource("logo.png")  
		);

        mailSender.send(message);
    }
}
```

#### Multiple Recipients & CC and VCC
```
helper.setTo(
    new String[]{
        "user1@gmail.com",
        "user2@gmail.com"
    }
);

helper.setCc("manager@gmail.com");
helper.setBcc("manager@gmail.com");
```


#### Exception Handling
```
SMTP Down
Wrong Credentials
Invalid Email
Timeout
```

```
try {
    mailSender.send(message);
}
catch (MailException ex) {
    throw new BusinessException(
        ErrorCode.EMAIL_SEND_FAILED
    );
}
```