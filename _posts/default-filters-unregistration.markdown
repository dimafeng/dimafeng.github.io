package com.fundcount.fccloud.app.config;

import java.util.*;

import javax.servlet.*;

import com.google.common.collect.*;
import lombok.extern.slf4j.*;
import org.springframework.beans.*;
import org.springframework.beans.factory.config.*;
import org.springframework.beans.factory.support.*;
import org.springframework.boot.context.embedded.*;
import org.springframework.stereotype.*;

/**
 * BeanFactoryPostProcessor which allows to disable all default filters mapped to /*
 * <p>
 * see https://github.com/spring-projects/spring-boot/issues/2173
 */
@Slf4j
@Component
public class DefaultFiltersBeanFactoryPostProcessor implements BeanFactoryPostProcessor
{
    public static final Set<String> NON_EXCLUDABLE_FILTERS = Sets.newHashSet("metricFilter", "webRequestLoggingFilter");

    public static class FilterRegistrationBeanFactory
    {
        public static FilterRegistrationBean create(Filter filter)
        {
            FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(filter);
            filterRegistrationBean.setEnabled(false);
            return filterRegistrationBean;
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) throws BeansException
    {
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) bf;

        Arrays.stream(beanFactory.getBeanNamesForType(javax.servlet.Filter.class))
                .filter(name -> !NON_EXCLUDABLE_FILTERS.contains(name))
                .forEach(name -> {
                    BeanDefinition definition = BeanDefinitionBuilder.rootBeanDefinition(FilterRegistrationBeanFactory.class, "create")
                            .setScope(BeanDefinition.SCOPE_SINGLETON)
                            .addConstructorArgReference(name)
                            .getBeanDefinition();

                    beanFactory.registerBeanDefinition(name + "FilterRegistrationBean", definition);

                    log.info("Filter that is represented by bean [{}] will be excluded from the default ones in scope of [{}]",
                            name, this.getClass().getName());
                });
    }
}
