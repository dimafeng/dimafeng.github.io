@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {CAbstractControllersTest.TestConfig.class})
@WebAppConfiguration
public abstract class CAbstractControllersTest
{
    @Autowired
    WebApplicationContext webApplicationContext;

    static IUserProvider userProvider = mock(IUserProvider.class);
    static AuthenticationManager authenticationManager = mock(AuthenticationManager.class);
    static ITokenManager tokenManager = mock(ITokenManager.class);
    static SecurityContextHolderStrategy securityContextHolderStrategy = mock(SecurityContextHolderStrategy.class);

    MockMvc mockMvc;

    @Before
    public void setup()
    {
        reset(userProvider);
        reset(authenticationManager);
        reset(tokenManager);

        mockMvc = MockMvcBuilders
                .webAppContextSetup(webApplicationContext)
                .addFilter(new DelegatingFilterProxy("springSecurityFilterChain", webApplicationContext), "/*")
                .build();
    }
    
    @Configuration
    @ComponentScan(basePackageClasses = CAbstractController.class)
    @EnableWebSecurity
    public static class TestConfig extends WebSecurityConfigurerAdapter
    {
        /**
         * This is a workaround for the issue in spring-test.
         * TODO: This bean declaration should be removed at all when the issue will have been fixed.
         */
        @Bean
        public AnnotationMethodHandlerAdapter annotationMethodHandlerAdapter()
        {
            return new AnnotationMethodHandlerAdapter()
            {
                @Override
                public HttpMessageConverter<?>[] getMessageConverters()
                {
                    return new HttpMessageConverter<?>[]{new MappingJackson2HttpMessageConverter()};
                }
            };
        }

        @Bean
        public CTokenAuthenticationFilter filter()
        {
            return new CTokenAuthenticationFilter();
        }

        @Bean(name = "springSecurityFilterChain")
        public FilterChainProxy configure() throws Exception
        {
            return new FilterChainProxy(new DefaultSecurityFilterChain(request -> {
                System.out.println(request);
                return true;}, filter()
            ));
        }

        @Bean
        public SecurityContextHolderStrategy securityContextHolderStrategy()
        {
            return securityContextHolderStrategy;
        }
    }
}
