Spring Cloud Gateway


Давайте теперь познакомимся с Spring Cloud Gateway. Он тоже достаточно интересный но долгое время не находил достаточной популярности среди разработчиков.

Команда Spring Cloud представила собственное решение неблокирующего шлюза в виде Spring Cloud Gateway - реактивного шлюза, построенного на основе Project Reactor, Spring WebFlux и Spring Boot 2.0.

Как же он устроен под капотом, если там нет никаких предпоссылок на сервелты ?





Когда запрос достигает шлюза, он сопоставляет запрос с каждым доступным маршрутом на основе определенного предиката. Если маршрут соответствует, запрос переходит к веб-обработчику, который применяет фильтры к запросу.

В Spring Cloud Gateway есть такое понятие как маршрут (route). Маршрут - это основной строительный блок шлюза. Он определяется идентификатором, URI назначения, набором предикатов и набором фильтров.

Ниже приведена простая реализация маршрута, которая направляет все запросы, соответствующие регулярному выражению /api/book-service/, в book-service.

Для этого используется бин RouteLocator в котором и происходит вся магия.

@Bean
public RouteLocator gatewayRoutes(RouteLocatorBuilder builder) {
return builder.routes()
.route(r -> r.path("/api/book-service/**")
.filters(f -> f.rewritePath("/api/book-service/(?.*)", "/${remains}")
.addRequestHeader("X-book-Header", "book-service-header")
.uri("lb://book-service/")
.id("book-service"))
}

Здесь мы использовали встроенные фильтры для перезаписи URL-адреса и добавления настраиваемого заголовка в качестве X-book-Header на лету.

Также здесь мы можем использовать библиотеку отказоустойчивости Hystrix. Например

.hystrix(c -> c.setName("hystrix")
.setFallbackUri("forward:/fallback/book-service")))

Если запрос завершился неудачей или не отвечает можно сделать так, что означает, что запрос будет перенаправлен в соответствующий контроллер и там будет вызван fallback-метод.
Чуть позже мы будем детальней знакомиться с этой библиотекой, с помощью которой можно делать невероятные вещи!


Например, в book-service будет следующий контроллер с обязательным хедером, который мы указали в Gateway

@RestController
public class FirstController {

@GetMapping("/testCall")
public String test(@RequestHeader("X-book-Header") String header){
return headerValue;
}

}

Пока все элементарно просто. Идем дальше.
Есть другой способ сделать это все через файл свойств, что кажется наиболее удачным решением

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- Path=/api/client/**
filters:
- RewritePath=/api/client/(?.*), /$\{remains}
- AddRequestHeader=X-client-Header, client-service-header
- name: Hystrix
args:
name: hystrix
fallbackUri: forward:/fallback/client

Мне кажется здесь все очевидно как и в примере выше:
Мы указываем путь (route) и сопоставляем его с URL и навешиваем на него predicate. Также у нас есть фильтр, который к запросу добавляет некоторый хедер, который, например, является обязательным в client-service, на который мы пытаемся попасть.

В разделе predicates мы указали путь, на который мы идем (client-service).
Существует дополнительные предикаты для ваших нужд.
Давайте рассмотрим их.

Before принимает один параметр - это datetime (ZonedDateTime). Этот предикат соответствует запросам, которые происходят до указанного datetime.

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- Path=/api/client/**
- Before=2017-01-20T17:42:47.789-07:00[America/Denver]

After также принимает один параметр - datetime (ZonedDateTime). Этот предикат соответствует запросам, которые происходят после указанного datetime.

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- Path=/api/client/**
- After=2017-01-20T17:42:47.789-07:00[America/Denver]

Between принимает два параметра - datetime1 и datetime2 (Java ZonedDateTime). Этот предикат соответствует запросам, которые происходят после datetime1 и до datetime2. Параметр datetime2 должен быть после datetime1.

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- Path=/api/client/**
- Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]

Этот маршрут соответствует любому запросу, сделанному после 20 января 2017 г., 17:42 и до 21 января 2017 г., 17:42.

Method принимает аргумент метода, который представляет собой один или несколько параметров: HTTP методы для сопоставления.

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- Method=GET,POST

RemoteAddr принимает список источников (минимум 1), которые представляют собой алреса IPv4 или IPv6, например 192.168.0.1/16 (где 192.168.0.1 - это IP-адрес, а 16 - маска подсети. ).

spring:
cloud:
gateway:
routes:
- id: client-service
uri: lb://client-service
predicates:
- RemoteAddr=192.168.0.1/16

Как уже писалось выше, вы можете добавлять фильтры - они позволяют каким-либо образом изменять входящий HTTP-запрос или исходящий HTTP-ответ.

Например добавить хедер к запросу

filters:
- RewritePath=/api/client/(?.*), /$\{remains}
- AddRequestHeader=X-client-Header, client-service-header

Добавить параметр к запросу

filters:
- AddRequestParameter=red, blue

Добавить хедер к ответу

filters:
- AddResponseHeader=X-client-Header, client-service-header

Добавить обработчик Circuit Breaker

filters:
- name: CircuitBreaker
  args:
  name: myCircuitBreaker
  fallbackUri: forward:/inCaseOfFailureUseThis
- RewritePath=/consumingServiceEndpoint, /backingServiceEndpoint

Это означает, что в случае неудачи будет вызван Circuit Breaker и выполнена какая-то логика либо повторный запрос на текущий адрес будет недоступен, пока не будет устранена неполадка.
Более подробно с паттерном Circuit Breaker мы будем знакомиться позже.

Параметр PrefixPath будет добавлять ко всем запросам определенный префикс

filters:
- PrefixPath=/mypath

например здесь на запрос /book-service будет добавлен префик /mypath

RedirectTo принимает два параметра: статус и URL. Параметр статуса должен представлять собой HTTP-код перенаправления из серии статус кодов 300 (например 301). Параметр url должен быть действительным URL-адресом.

filters:
- RedirectTo=302, https://google.com

RemoveRequestHeader указывает имя хедера, который надо удалить из запроса

filters:
- RemoveRequestHeader=X-Request-Foo

RemoveResponseHeader указывает имя хедера, который надо удалить из ответа

filters:
- RemoveResponseHeader=X-Response-Foo

RemoveRequestParameter указывает имя параметра, который надо удалить из запроса

filters:
- RemoveRequestParameter=red

SetStatus устанавливает статус для текущего запроса

filters:
- SetStatus=404

Retry устанавливает количество повторных попыток в случае неуспешного выполнения запроса

filters:
- name: Retry
  args:
  retries: 3
  statuses: BAD_GATEWAY
  methods: GET,POST
  backoff:
  firstBackoff: 10ms
  maxBackoff: 50ms
  factor: 2
  basedOnPreviousValue: false

С ним, как и с паттерном CircuitBreaker, мы будем знакомиться чуть позже.
Как настроить таймаут соединения ? Очень просто для всех запросов

spring:
cloud:
gateway:
httpclient:
connect-timeout: 1000
response-timeout: 5s

для конкретного запроса

spring:
cloud:
gateway:
routes:
- id: per_route_timeouts
uri: https://example.org
predicates:
- name: Path
args:
pattern: /delay/{timeout}
metadata:
response-timeout: 200
connect-timeout: 200

или через java код, как мы делали это выше

@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder routeBuilder){
return routeBuilder.routes()
.route("test", r -> {
return r.host("*.mysite.com").and().path("/somepath")
.uri("http://someuri")
.metadata(RESPONSE_TIMEOUT_ATTR, 200)
.metadata(CONNECT_TIMEOUT_ATTR, 200);
})
.build();