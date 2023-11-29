# Usage in Java spring boot

First make sure you have `site_key` and `secret_key` in your pocket!

## 1. Storing the API Key-Pair

We store the keys in the `application.properties`:

```
arcaptcha.key.site=21smftb02t
arcaptcha.key.secret=8jerm4yilhakw37ygiz2
```

And expose them to Spring using a bean annotated with `@ConfigurationProperties`:

```java
@Component
@ConfigurationProperties(prefix = "arcaptcha.key")
public class CaptchaSettings {

    private String site;
    private String secret;

    public String getARCaptchaSecret() {
        return this.secret;
    }

    public String getARCaptchaSite() {
        return this.site;
    }
}
```

## 2.Displaying the Widget

We suppose you have a html file named `registration.html` that is responsible for user registration.

Inside our registration form, we add the ARCaptcha widget which expects the attribute `data-site-key` to contain the site-key.

The widget will append the request parameter **arcaptcha-token** when submitted:

```html
<!DOCTYPE html>
<html>
  <head>
    ...

    <script src="https://172.24.105.155/widget/1/api.js" async defer></script>
  </head>
  <body>
    ...

    <form method="POST" enctype="utf8">
      ...
      <div class="arcaptcha" data-site-key="21smftb02t"></div>
    </form>
  </body>
</html>
```

## 3.Server-Side Validation

The new request parameter encodes our site key and a unique string identifying the user’s successful completion of the challenge.

However, since we cannot discern that ourselves, we cannot trust what the user has submitted is legitimate. A server-side request is made to validate the captcha response with the web-service API.

The endpoint accepts an HTTP request on the URL `https://172.24.105.155/api/arcaptcha/api/verify`. It returns a JSON response having the schema:

```json
{
    "success": true|false,
}
```

## 3.1. Retrieve User’s Response

The user’s response to the ARCaptcha challenge is retrieved from the request parameter `arcaptcha-token` using `HttpServletRequest` and validated with our CaptchaService. Any exception thrown while processing the response will abort the rest of the registration logic:

```java
public class RegistrationController {

    @Autowired
    private ICaptchaService captchaService;

    ...

    @RequestMapping(value = "/user/registration", method = RequestMethod.POST)
    @ResponseBody
    public GenericResponse registerUserAccount(@Valid UserDto accountDto, HttpServletRequest request) {
        String response = request.getParameter("arcaptcha-token");
        captchaService.processResponse(response);

        // Rest of implementation
    }

    ...
}
```

## 3.2. Validation Service

In this step we make a request to the web service with the secret-key, the captcha token and the site_key:

```java
public class CaptchaService implements ICaptchaService {

    @Autowired
    private CaptchaSettings captchaSettings;

    @Autowired
    private RestOperations restTemplate;

    @Override
    public void processResponse(String token) {

        String url = "https://172.24.105.155/api/arcaptcha/api/verify";

        String requestBody = new JSONObject()
                  .put("challenge_id", token)
                  .put("site_key", getARCaptchaSite())
                  .put("secret_key",getARCaptchaSecret())
                  .toString();

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        HttpEntity<String> request = new HttpEntity<>(json, headers);

        ARCaptchaResponse arcaptchaResponse = restTemplate.postForObject(url,request, ARCaptchaResponse.class);

        if(!arcaptchaResponse.isSuccess()) {
            // ex: throw an exception
        }
    }
}
```

## 3.3. Objectifying the Validation

A Java bean decorated with `Jackson` annotations encapsulates the validation response:

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonIgnoreProperties(ignoreUnknown = true)
@JsonPropertyOrder({
    "success",
})
public class ARCaptchaResponse {

    @JsonProperty("success")
    private boolean success;

    @JsonIgnore
    public boolean hasClientError() {
        ErrorCode[] errors = getErrorCodes();
        if(errors == null) {
            return false;
        }
        for(ErrorCode error : errors) {
            switch(error) {
                case InvalidResponse:
                case MissingResponse:
                    return true;
            }
        }
        return false;
    }

    static enum ErrorCode {
        MissingSecret,     InvalidSecret,
        MissingResponse,   InvalidResponse,
        MissingSite,       InvalidSite;

        private static Map<String, ErrorCode> errorsMap = new HashMap<String, ErrorCode>(4);

        static {
            errorsMap.put("missing-input-secret",   MissingSecret);
            errorsMap.put("invalid-input-secret",   InvalidSecret);
            errorsMap.put("missing-input-response", MissingResponse);
            errorsMap.put("invalid-input-response", InvalidResponse);
            errorsMap.put("missing-input-sitekey", MissingSite);
            errorsMap.put("invalid-input-sitekey", InvalidSite);
        }

        @JsonCreator
        public static ErrorCode forValue(String value) {
            return errorsMap.get(value.toLowerCase());
        }
    }

    // standard getters and setters
}
```

As implied, a truth value in the `success` property means the user has been validated. Otherwise, the errorCodes property will populate with the reason.

## 3.4. Validation Failure

In the event of a validation failure, an exception is thrown. The ARCaptcha library needs to instruct the client to create a new challenge.

We do so in the client’s registration error handler, by invoking reset on the library’s arcaptcha widget:

```javascript
register(event){
    event.preventDefault();

    var formData= $('form').serialize();
    $.post(serverContext + "user/registration", formData, function(data){
        if(data.message == "success") {
            // success handler
        }
    })
    .fail(function(data) {
        arcaptcha.reset();
    }
}
```
