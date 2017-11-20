---
layout: post
title:  "Grails 3.3.0去掉Hibernate外键生成"
date:   2017-11-16 12:26:28 +0800
categories: grails
tag: hibernate
---



系统环境：

grails: 3.3.0

gorm: 6.1.6.RELEASE

hibernate:

<pre style="font-family: sans-serif;font-size: 14px;color: black;">
buildscript {
    dependencies {
        classpath "org.grails.plugins:hibernate5:${gormVersion-".RELEASE"}"
    }
}

dependencies{
    compile "org.grails.plugins:hibernate5"
    compile "org.hibernate:hibernate-core:5.1.5.Final"
    compile "org.hibernate:hibernate-ehcache:5.1.0.Final"
}
</pre>

想在Hibernate自动从domain生成数据库表的时候，不生成所有数据库外键。
Hibernate5以前的实现方法： [https://stackoverflow.com/questions/4768231/gorm-prevent-creation-of-a-foreign-key-constraint-for-a-domain](https://stackoverflow.com/questions/4768231/gorm-prevent-creation-of-a-foreign-key-constraint-for-a-domain)

Hibernate5的实现方法：
新建两个类：MyGrailsDomainBinder.java和MyHibernateMappingContextConfiguration.java（java文件或者groovy都可以），两个类的路径都是org.grails.orm.hibernate.cfg。
具体代码：

<pre style="font-family: sans-serif;font-size: 14px;color: black;">

package org.grails.orm.hibernate.cfg;

import org.grails.datastore.gorm.GormEntity;
import org.grails.datastore.mapping.model.PersistentEntity;
import org.grails.orm.hibernate.EventListenerIntegrator;
import org.grails.orm.hibernate.HibernateEventListeners;
import org.grails.orm.hibernate.access.TraitPropertyAccessStrategy;
import org.hibernate.HibernateException;
import org.hibernate.SessionFactory;
import org.hibernate.SessionFactoryObserver;
import org.hibernate.boot.registry.BootstrapServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.boot.registry.classloading.internal.ClassLoaderServiceImpl;
import org.hibernate.boot.registry.classloading.spi.ClassLoaderService;
import org.hibernate.boot.registry.selector.spi.StrategySelector;
import org.hibernate.boot.spi.MetadataContributor;
import org.hibernate.cfg.AvailableSettings;
import org.hibernate.internal.util.config.ConfigurationHelper;
import org.hibernate.property.access.spi.PropertyAccessStrategy;
import org.hibernate.service.ServiceRegistry;
import org.hibernate.service.spi.ServiceRegistryImplementor;
import javax.persistence.Entity;
import java.util.*;

/**
 * 继承HibernateMappingContextConfiguration实现自定义Hibernate设置
 * @author 王宇
 */
public class MyHibernateMappingContextConfiguration extends HibernateMappingContextConfiguration {

    private HibernateEventListeners hibernateEventListeners;
    private Map<String, Object> eventListeners;
    private ServiceRegistry serviceRegistry;
    private MetadataContributor metadataContributor;
    private Set<Class> additionalClasses = new HashSet<>();

    @Override
    public SessionFactory buildSessionFactory() throws HibernateException {

        // set the class loader to load Groovy classes

        // work around for HHH-2624
        SessionFactory sessionFactory;

        Object classLoaderObject = getProperties().get(AvailableSettings.CLASSLOADERS);
        ClassLoader appClassLoader;

        if(classLoaderObject instanceof ClassLoader) {
            appClassLoader = (ClassLoader) classLoaderObject;
        }
        else {
            appClassLoader = getClass().getClassLoader();
        }

        ConfigurationHelper.resolvePlaceHolders(getProperties());

        final MyGrailsDomainBinder domainBinder = new MyGrailsDomainBinder(dataSourceName, sessionFactoryBeanName, hibernateMappingContext);

        List<Class> annotatedClasses = new ArrayList<>();
        for (PersistentEntity persistentEntity : hibernateMappingContext.getPersistentEntities()) {
            Class javaClass = persistentEntity.getJavaClass();
            if(javaClass.isAnnotationPresent(Entity.class)) {
                annotatedClasses.add(javaClass);
            }
        }

        if(!additionalClasses.isEmpty()) {
            for (Class additionalClass : additionalClasses) {
                if(GormEntity.class.isAssignableFrom(additionalClass)) {
                    hibernateMappingContext.addPersistentEntity(additionalClass);
                }
            }
        }

        addAnnotatedClasses( annotatedClasses.toArray(new Class[annotatedClasses.size()]));

        ClassLoaderService classLoaderService = new ClassLoaderServiceImpl(appClassLoader) {
            @Override
            public </S> Collection</S> loadJavaServices(Class</S> serviceContract) {
                if(MetadataContributor.class.isAssignableFrom(serviceContract)) {
                    if(metadataContributor != null) {
                        return (Collection</S>) Arrays.asList(domainBinder, metadataContributor);
                    }
                    else {
                        return Collections.singletonList((S) domainBinder);
                    }
                }
                else {
                    return super.loadJavaServices(serviceContract);
                }
            }
        };
        EventListenerIntegrator eventListenerIntegrator = new EventListenerIntegrator(hibernateEventListeners, eventListeners);
        BootstrapServiceRegistry bootstrapServiceRegistry = createBootstrapServiceRegistryBuilder()
                .applyIntegrator(eventListenerIntegrator)
                .applyClassLoaderService(classLoaderService)
                .build();
        StrategySelector strategySelector = bootstrapServiceRegistry.getService(StrategySelector.class);

        strategySelector.registerStrategyImplementor(
                PropertyAccessStrategy.class, "traitProperty", TraitPropertyAccessStrategy.class
        );

        setSessionFactoryObserver(new SessionFactoryObserver() {
            private static final long serialVersionUID = 1;
            public void sessionFactoryCreated(SessionFactory factory) {}
            public void sessionFactoryClosed(SessionFactory factory) {
                ((ServiceRegistryImplementor)serviceRegistry).destroy();
            }
        });

        StandardServiceRegistryBuilder standardServiceRegistryBuilder = createStandardServiceRegistryBuilder(bootstrapServiceRegistry)
                .applySettings(getProperties());

        StandardServiceRegistry serviceRegistry = standardServiceRegistryBuilder.build();
        sessionFactory = super.buildSessionFactory(serviceRegistry);
        this.serviceRegistry = serviceRegistry;

        return sessionFactory;
    }
}

</pre>

<pre style="font-family: sans-serif;font-size: 14px;color: black;">

package org.grails.orm.hibernate.cfg;

import org.grails.datastore.mapping.model.*;
import org.grails.datastore.mapping.model.config.GormProperties;
import org.grails.datastore.mapping.model.types.*;
import org.hibernate.MappingException;
import org.hibernate.boot.spi.*;
import org.hibernate.mapping.*;
import org.hibernate.mapping.ManyToOne;
import org.hibernate.mapping.OneToOne;
import org.hibernate.mapping.Table;
import java.util.*;
import java.util.List;

/**
 * 继承GrailsDomainBinder实现数据库不创建外键关联
 *
 * @author 王宇
 * @since 0.1
 */
public class MyGrailsDomainBinder extends GrailsDomainBinder {

    MyGrailsDomainBinder(String dataSourceName, String sessionFactoryName, HibernateMappingContext hibernateMappingContext) {
        super(dataSourceName,sessionFactoryName,hibernateMappingContext);
    }

    /**
     * Creates and binds the properties for the specified Grails domain class and PersistentClass
     * and binds them to the Hibernate runtime meta model
     *
     * @param domainClass     The Grails domain class
     * @param persistentClass The Hibernate PersistentClass instance
     * @param mappings        The Hibernate Mappings instance
     * @param sessionFactoryBeanName  the session factory bean name
     */
    protected void createClassProperties(HibernatePersistentEntity domainClass, PersistentClass persistentClass,
                                         InFlightMetadataCollector mappings, String sessionFactoryBeanName) {

        final List<PersistentProperty> persistentProperties = domainClass.getPersistentProperties();
        Table table = persistentClass.getTable();

        Mapping gormMapping = domainClass.getMapping().getMappedForm();

        if (gormMapping != null) {
            table.setComment(gormMapping.getComment());
        }

        List<Embedded> embedded = new ArrayList<>();

        for (PersistentProperty currentGrailsProp : persistentProperties) {

            // if its inherited skip
            if (currentGrailsProp.isInherited()) {
                continue;
            }
            if(currentGrailsProp.getName().equals(GormProperties.VERSION) ) continue;
            if (isCompositeIdProperty(gormMapping, currentGrailsProp)) continue;
            if (isIdentityProperty(gormMapping, currentGrailsProp)) continue;

            if (LOG.isDebugEnabled()) {
                LOG.debug("[GrailsDomainBinder] Binding persistent property [" + currentGrailsProp.getName() + "]");
            }

            Value value = null;

            // see if it's a collection type
            CollectionType collectionType = super.CT.collectionTypeForClass(currentGrailsProp.getType());

            Class<?> userType = getUserType(currentGrailsProp);

            if (userType != null) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("[GrailsDomainBinder] Binding property [" + currentGrailsProp.getName() + "] as SimpleValue");
                }
                value = new SimpleValue(mappings, table);
                bindSimpleValue(currentGrailsProp, null, (SimpleValue) value, EMPTY_PATH, mappings, sessionFactoryBeanName);
            }
            else if (collectionType != null) {
                String typeName = getTypeName(currentGrailsProp, getPropertyConfig(currentGrailsProp),gormMapping);
                if ("serializable".equals(typeName)) {
                    value = new SimpleValue(mappings, table);
                    bindSimpleValue(typeName, (SimpleValue) value, currentGrailsProp.isNullable(),
                            getColumnNameForPropertyAndPath(currentGrailsProp, EMPTY_PATH, null, sessionFactoryBeanName), mappings);
                }
                else {
                    // create collection
//                    Collection collection = collectionType.create((ToMany) currentGrailsProp, persistentClass,
//                            EMPTY_PATH, mappings, sessionFactoryBeanName);
//                    mappings.addCollectionBinding(collection);
//                    value = collection;
                    continue;
                }
            }
            else if (currentGrailsProp.getType().isEnum()) {
                value = new SimpleValue(mappings, table);
                bindEnumType(currentGrailsProp, (SimpleValue) value, EMPTY_PATH, sessionFactoryBeanName);
            }
            else if(currentGrailsProp instanceof Association) {
                Association association = (Association) currentGrailsProp;
                if (currentGrailsProp instanceof org.grails.datastore.mapping.model.types.ManyToOne) {
                    if (LOG.isDebugEnabled())
                        LOG.debug("[GrailsDomainBinder] Binding property [" + currentGrailsProp.getName() + "] as ManyToOne");

                    value = new ManyToOne(mappings, table);
                    bindManyToOne((Association) currentGrailsProp, (ManyToOne) value, EMPTY_PATH, mappings, sessionFactoryBeanName);
                }
                else if (currentGrailsProp instanceof org.grails.datastore.mapping.model.types.OneToOne && userType == null) {
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("[GrailsDomainBinder] Binding property [" + currentGrailsProp.getName() + "] as OneToOne");
                    }

                    final boolean isHasOne = isHasOne(association);
                    if (isHasOne && !association.isBidirectional()) {
                        throw new MappingException("hasOne property [" + currentGrailsProp.getOwner().getName() +
                                "." + currentGrailsProp.getName() + "] is not bidirectional. Specify the other side of the relationship!");
                    }
                    else if (canBindOneToOneWithSingleColumnAndForeignKey((Association) currentGrailsProp)) {
                        value = new OneToOne(mappings, table, persistentClass);
                        bindOneToOne((org.grails.datastore.mapping.model.types.OneToOne) currentGrailsProp, (OneToOne) value, EMPTY_PATH, sessionFactoryBeanName);
                    }
                    else {
                        if (isHasOne && association.isBidirectional()) {
                            value = new OneToOne(mappings, table, persistentClass);
                            bindOneToOne((org.grails.datastore.mapping.model.types.OneToOne) currentGrailsProp, (OneToOne) value, EMPTY_PATH, sessionFactoryBeanName);
                        }
                        else {
                            value = new ManyToOne(mappings, table);
                            bindManyToOne((Association) currentGrailsProp, (ManyToOne) value, EMPTY_PATH, mappings, sessionFactoryBeanName);
                        }
                    }
                }
                else if (currentGrailsProp instanceof Embedded) {
                    embedded.add((Embedded)currentGrailsProp);
                    continue;
                }
            }
            // work out what type of relationship it is and bind value
            else {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("[GrailsDomainBinder] Binding property [" + currentGrailsProp.getName() + "] as SimpleValue");
                }
                value = new SimpleValue(mappings, table);
                bindSimpleValue(currentGrailsProp, null, (SimpleValue) value, EMPTY_PATH, mappings, sessionFactoryBeanName);
            }

            if (value != null) {
                Property property = createProperty(value, persistentClass, currentGrailsProp, mappings);
                persistentClass.addProperty(property);
            }
        }

        for (Embedded association : embedded) {
            Value value = new Component(mappings, persistentClass);

            bindComponent((Component) value, association, true, mappings, sessionFactoryBeanName);
            Property property = createProperty(value, persistentClass, association, mappings);
            persistentClass.addProperty(property);
        }
        bindNaturalIdentifier(table, gormMapping, persistentClass);
    }

    private boolean isHasOne(Association association) {
        return association instanceof org.grails.datastore.mapping.model.types.OneToOne && ((org.grails.datastore.mapping.model.types.OneToOne)association).isForeignKeyInChild();
    }

    /*
     * Creates a persistent class property based on the GrailDomainClassProperty instance.
     */
    protected Property createProperty(Value value, PersistentClass persistentClass, PersistentProperty grailsProperty, InFlightMetadataCollector mappings) {
        // set type
        value.setTypeUsingReflection(persistentClass.getClassName(), grailsProperty.getName());

//        if (value.getTable() != null) {
//            value.createForeignKey();
//        }

        Property prop = new Property();
        prop.setValue(value);
        bindProperty(grailsProperty, prop, mappings);
        return prop;
    }
}
</pre>

application.yml：
<pre style="font-family: sans-serif;font-size: 14px;color: black;">
hibernate:
    configClass: org.grails.orm.hibernate.cfg.MyHibernateMappingContextConfiguration
</pre>

##### [jekyll]:      [http://jekyllrb.com](http://jekyllrb.com)
