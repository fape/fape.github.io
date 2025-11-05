---
layout: post
title: Spring boot update-nél régi OAuth alapú backend megoldás upgrade
---

# Probléma
Spring boot update-nél már teljesen személyre szabott korábbi [OAuth2RestTemplate](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/client/OAuth2RestTemplate.html)-t helyett már más megoldásra van szükség. 

# Előfeltételek
A szabványtól pár helyen eltérnek a végpontok, amiket használnunk kell:
* POST helyett GET van használva, mint access mint refresh token-él.
* Más headerben megy a basic auth.
* Proxy-n keresztül kell menni az összes kérésnek
* Cacheljük a tokeneket

# Megoldás 1
WebClient-t és ServletOAuth2AuthorizedClientExchangeFilterFunction-t használjunk. Apró szépség hiba a DefaultRefreshTokenTokenResponseClient és DefaultClientCredentialsTokenResponseClient is deprecated már.
```java
@Configuration
public class WebClientConfig {

    @Value("${proxy.host}")
    String proxyHost;
    @Value("${proxy.port}")
    int proxyPort;
    @Value("${proxy.username}")
    String proxyUsername;
    @Value("${proxy.password}")
    String proxyPassword;

    private ClientHttpConnector getClientHttpConnector() {
        HttpClient httpClient = HttpClient.create()
                .proxy(proxy -> proxy
                        .type(ProxyProvider.Proxy.HTTP)
                        .host(proxyHost)
                        .port(proxyPort)
                        .username(proxyUsername)
                        .password(s -> proxyPassword)
                )
                ;
        return new ReactorClientHttpConnector(httpClient);
    }

    @Bean("oAuthProxyWebClient")
    public WebClient oAuthProxyWebClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        var oauth2 = new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        oauth2.setDefaultClientRegistrationId("testoidc");

        return WebClient.builder()
                .apply(oauth2.oauth2Configuration())
                 // Proxy-n keresztül kell menni az összes kérésnek
                .clientConnector(getClientHttpConnector())
                .build();
    }

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientService authorizedClientService) {

        var provider = OAuth2AuthorizedClientProviderBuilder.builder()
                .refreshToken(r -> r.accessTokenResponseClient(
                        getDefaultRefreshTokenTokenResponseClient()

                ))
                .clientCredentials(c -> c.accessTokenResponseClient(
                        getDefaultClientCredentialsTokenResponseClient()
                ))
                .build();

        var manager = new AuthorizedClientServiceOAuth2AuthorizedClientManager(clientRegistrationRepository,
                authorizedClientService);
        manager.setAuthorizedClientProvider(provider);

        return manager;
    }

    private DefaultRefreshTokenTokenResponseClient getDefaultRefreshTokenTokenResponseClient() {
        DefaultRefreshTokenTokenResponseClient accessTokenResponseClient = new DefaultRefreshTokenTokenResponseClient();
        OAuth2RefreshTokenGrantRequestEntityConverter requestEntityConverter
                = new OAuth2RefreshTokenGrantRequestEntityConverter() {
            /** @see AbstractOAuth2AuthorizationGrantRequestEntityConverter#convert */
            @Override
            public RequestEntity<?> convert(OAuth2RefreshTokenGrantRequest authorizationGrantRequest) {
                RequestEntity<?> converted = super.convert(authorizationGrantRequest);
                // POST helyett GET van használva mint access mint refresh token-él.
                return new RequestEntity<>(null, converted.getHeaders(), HttpMethod.GET, converted.getUrl());
            }
        };

        /** @see DefaultOAuth2TokenRequestHeadersConverter#convert */
        requestEntityConverter.setHeadersConverter(grantRequest -> {
            HttpHeaders headers = new HttpHeaders();
            headers.setAccept(List.of(MediaType.APPLICATION_JSON));
            ClientRegistration clientRegistration = grantRequest.getClientRegistration();

            headers.set(OAuth2ParameterNames.CLIENT_ID, clientRegistration.getClientId());

            headers.setBasicAuth(clientRegistration.getClientId(), clientRegistration.getClientSecret());
            // Más headerben megy a basic auth.
            headers.set("customer_header_name", headers.getFirst(HttpHeaders.AUTHORIZATION));
            headers.remove(HttpHeaders.AUTHORIZATION);

            headers.set(OAuth2ParameterNames.REFRESH_TOKEN, grantRequest.getRefreshToken().getTokenValue());

            return headers;

        });

        accessTokenResponseClient.setRequestEntityConverter(requestEntityConverter);

        // Proxy-n keresztül kell menni az összes kérésnek
        accessTokenResponseClient.setRestOperations(getRestTemplateWithProxy());
        return accessTokenResponseClient;
    }


    private DefaultClientCredentialsTokenResponseClient getDefaultClientCredentialsTokenResponseClient() {
        DefaultClientCredentialsTokenResponseClient accessTokenResponseClient = new DefaultClientCredentialsTokenResponseClient();

        OAuth2ClientCredentialsGrantRequestEntityConverter requestEntityConverter
                = new OAuth2ClientCredentialsGrantRequestEntityConverter() {
            /** @see AbstractOAuth2AuthorizationGrantRequestEntityConverter#convert   */
            @Override
            public RequestEntity<?> convert(OAuth2ClientCredentialsGrantRequest authorizationGrantRequest) {
                RequestEntity<?> converted = super.convert(authorizationGrantRequest);
                // POST helyett GET van használva mint access mint refresh token-él.
                return new RequestEntity<>(null, converted.getHeaders(), HttpMethod.GET, converted.getUrl());
            }
        };

        /** @see DefaultOAuth2TokenRequestHeadersConverter#convert  */
        requestEntityConverter.setHeadersConverter(grantRequest -> {
            HttpHeaders headers = new HttpHeaders();
            headers.setAccept(List.of(MediaType.APPLICATION_JSON));
            ClientRegistration clientRegistration = grantRequest.getClientRegistration();

            headers.set(OAuth2ParameterNames.CLIENT_ID, clientRegistration.getClientId());

            headers.setBasicAuth(clientRegistration.getClientId(), clientRegistration.getClientSecret());
            // Más headerben megy a basic auth
            headers.set("customer_header_name", headers.getFirst(HttpHeaders.AUTHORIZATION));
            headers.remove(HttpHeaders.AUTHORIZATION);

            return headers;

        });
        accessTokenResponseClient.setRequestEntityConverter(requestEntityConverter);

        // Proxy-n keresztül kell menni az összes kérésnek
        accessTokenResponseClient.setRestOperations(getRestTemplateWithProxy());
        return accessTokenResponseClient;
    }

    private RestTemplate getRestTemplateWithProxy() {
        ClientHttpRequestFactory factory = getClientHttpRequestFactory();

        RestTemplate restTemplate = new RestTemplate(factory);
        restTemplate.setMessageConverters(List.of(new OAuth2AccessTokenResponseHttpMessageConverter()));
        restTemplate.setErrorHandler(new OAuth2ErrorResponseErrorHandler());

        return restTemplate;
    }

    private ClientHttpRequestFactory getClientHttpRequestFactory() {
        HttpHost proxy = new HttpHost("http", proxyHost, proxyPort);

        BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(
                new AuthScope(proxyHost, proxyPort),
                new UsernamePasswordCredentials(proxyUsername, proxyPassword.toCharArray())
        );
        CloseableHttpClient httpClient = HttpClients.custom()
                .setRoutePlanner(new DefaultProxyRoutePlanner(proxy))
                .setDefaultCredentialsProvider(credentialsProvider)
                .build();

        return new HttpComponentsClientHttpRequestFactory(httpClient);
    }

    @Bean
    public OAuth2AuthorizedClientService authorizedClientService(ClientRegistrationRepository clientRegistrationRepository) {
	    // Cacheljük a tokeneket
        return new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
    }
}

```
valamint
```properties
# Client registration
spring.security.oauth2.client.registration.dtoidc.client-id=client-id
spring.security.oauth2.client.registration.dtoidc.client-secret=client-secret
spring.security.oauth2.client.registration.dtoidc.authorization-grant-type=client_credentials
spring.security.oauth2.client.registration.dtoidc.client-name=client-name

# Provider
spring.security.oauth2.client.provider.dtoidc.token-uri=https://example.com/generate/accessToken
spring.security.oauth2.client.provider.dtoidc.refresh-token-uri==https://example.com/generate/refreshToken

# Proxy
proxy.host=proxy.example.com
proxy.port=8080
proxy.username=username
proxy.password=password
```
és egy TestController ami csak átproxyza a hívást
```java
@RestController
public class TestController {

    @Autowired
    @Qualifier("oAuthProxyWebClient")
    WebClient webClient;

    @GetMapping(value = "oAuthProxyWebClient", produces = MediaType.TEXT_PLAIN_VALUE)
    public ResponseEntity<String> SendWithOAuthProxyWebClient() {
        Map<String, Object> body = Map.ofEntries(
                entry("prop1", "value1"),
                entry("prop2", List.of(
                        Map.of(
                                "name", "name1",
                                "value", "value2"
                        )
                )),
                entry("prop3", List.of("value3-1", "value3-2"))
        );


        return webClient.post()
                .uri("https://example.com/events")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(body)
                .exchangeToMono(clientResponse -> clientResponse.toEntity(String.class))
                .block();

    }
}
```

