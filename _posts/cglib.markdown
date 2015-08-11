import org.springframework.context.support.*;

/**
 * Special spring Application Context which solves <a href="https://jira.spring.io/browse/SPR-12833">SPR-12833</a>
 * using <code>ClassLoader</code> with no signature check.
 *
 * Should be used in cases when CGLIB proxies are generated in a signed environment.
 */
public class CXmlApplicationContextWithNoSignCheck extends ClassPathXmlApplicationContext
{
    public CXmlApplicationContextWithNoSignCheck(String... configLocations)
    {
        super(configLocations, false);
        setClassLoader(new UnsecureClassLoader());
        refresh();
    }
}



import java.lang.reflect.*;
import java.util.*;

/**
 * ClassLoader with disabled signature checking
 */
public class UnsecureClassLoader extends ClassLoader
{
    public UnsecureClassLoader()
    {
        this(UnsecureClassLoader.class.getClassLoader());
    }

    public UnsecureClassLoader(ClassLoader classLoader)
    {
        super(classLoader);

        try
        {
            /**
             * Changing map with certificates from real one to map without elements.
             * All system's attempts to add certificates to the map do nothing.
             */
            Field package2certs = ClassLoader.class.getDeclaredField("package2certs");
            package2certs.setAccessible(true);
            package2certs.set(this, new AbstractMap()
            {

                @Override
                public Set<Entry> entrySet()
                {
                    return Collections.emptySet();
                }

                @Override
                public Object put(Object key, Object value)
                {
                    return null;
                }
            });
        }
        catch (Exception e)
        {
            throw new RuntimeException(e);
        }
    }
}
