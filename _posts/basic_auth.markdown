This's quich tip how to use basic auth with spring rest template.

String plainCreds = "willie:p@ssword";
byte[] plainCredsBytes = plainCreds.getBytes();
byte[] base64CredsBytes = Base64.encodeBase64(plainCredsBytes);
String base64Creds = new String(base64CredsBytes);

1

    protected RestTemplate getRestTemplate()
    {
        SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        requestFactory.setBufferRequestBody(false);
        RestTemplate restTemplate = new RestTemplate(requestFactory);
        restTemplate.setInterceptors(Arrays.asList(new AuthInterceptor()));
        return restTemplate;
    }
        
            private class AuthInterceptor implements ClientHttpRequestInterceptor
    {
        @Override
        public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                            ClientHttpRequestExecution execution) throws IOException
        {
            HttpHeaders headers = request.getHeaders();
            headers.add("Authorization", "Bearer " + configService.getDigitaloceanAuthApiKey());
            return execution.execute(request, body);
        }
    }
    
    
2    
    HttpEntity<String> request = new HttpEntity<String>(headers);
ResponseEntity<Account> response = restTemplate.exchange(url, HttpMethod.GET, request, Account.class);
Account account = response.getBody();

