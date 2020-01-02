### mysql + springboot + mybatis 实现读写分离
好久不看后端的内容了，今天突然想整理一下这个读写分离的问题，发现实现其手段有两个方面，第一个就是通过中间件(mycat)，另一个就是通过springboot的配置，今天主要阐述一下如何通过后端配置达到该目标。主要有几个方面的内容：
#### mysql多源数据库配置
为了阐述方便这里用一个master和两个slave进行配置，配置内容摘抄网上内容如下：

    spring:
      datasource:
        master:
          jdbc-url: jdbc:mysql://192.168.102.31:3306/test
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave1:
          jdbc-url: jdbc:mysql://192.168.102.56:3306/test
          username: pig   # 只读账户
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave2:
          jdbc-url: jdbc:mysql://192.168.102.36:3306/test
          username: pig   # 只读账户
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
          
 有关mysql如何进行主从数据同步的问题下一节进行介绍。
 #### springboot 数据源配置
 
    package com.cjs.example.config;
    import com.cjs.example.bean.MyRoutingDataSource;
    import com.cjs.example.enums.DBTypeEnum;
    import org.springframework.beans.factory.annotation.Qualifier;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.boot.jdbc.DataSourceBuilder;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;

    import javax.sql.DataSource;
    import java.util.HashMap;
    import java.util.Map;

    /**
     * 关于数据源配置，参考SpringBoot官方文档第79章《Data Access》
     * 79. Data Access
     * 79.1 Configure a Custom DataSource
     * 79.2 Configure Two DataSources
     */

    @Configuration
    public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave1")
    public DataSource slave1DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave2")
    public DataSource slave2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource myRoutingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                                          @Qualifier("slave1DataSource") DataSource slave1DataSource,
                                          @Qualifier("slave2DataSource") DataSource slave2DataSource) {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DBTypeEnum.MASTER, masterDataSource);
        targetDataSources.put(DBTypeEnum.SLAVE1, slave1DataSource);
        targetDataSources.put(DBTypeEnum.SLAVE2, slave2DataSource);
        MyRoutingDataSource myRoutingDataSource = new MyRoutingDataSource();
        myRoutingDataSource.setDefaultTargetDataSource(masterDataSource);
        myRoutingDataSource.setTargetDataSources(targetDataSources);
        return myRoutingDataSource;
    }

}


#### mybatis 配置
    package com.cjs.example.config;

	import org.apache.ibatis.session.SqlSessionFactory;
	import org.mybatis.spring.SqlSessionFactoryBean;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
    import org.springframework.jdbc.datasource.DataSourceTransactionManager;
    import org.springframework.transaction.PlatformTransactionManager;
    import org.springframework.transaction.annotation.EnableTransactionManagement;

    import javax.annotation.Resource;
    import javax.sql.DataSource;

    @EnableTransactionManagement
    @Configuration
    public class MyBatisConfig {

    @Resource(name = "myRoutingDataSource")
    private DataSource myRoutingDataSource;

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(myRoutingDataSource);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    public PlatformTransactionManager platformTransactionManager() {
        return new DataSourceTransactionManager(myRoutingDataSource);
    }
}
#### 配置theadlocal变量
    package com.cjs.example.enums;

		public enum DBTypeEnum {

    		MASTER, SLAVE1, SLAVE2;

		}
        
   配置threadlocal变量内容
     
    package com.cjs.example.bean;

    import com.cjs.example.enums.DBTypeEnum;

    import java.util.concurrent.atomic.AtomicInteger;

    public class DBContextHolder {

    private static final ThreadLocal<DBTypeEnum> contextHolder = new ThreadLocal<>();

    private static final AtomicInteger counter = new AtomicInteger(-1);

    public static void set(DBTypeEnum dbType) {
        contextHolder.set(dbType);
    }

    public static DBTypeEnum get() {
        return contextHolder.get();
    }

    public static void master() {
        set(DBTypeEnum.MASTER);
        System.out.println("切换到master");
    }

    public static void slave() {
        //  轮询
        int index = counter.getAndIncrement() % 2;
        if (counter.get() > 9999) {
            counter.set(-1);
        }
        if (index == 0) {
            set(DBTypeEnum.SLAVE1);
            System.out.println("切换到slave1");
        }else {
            set(DBTypeEnum.SLAVE2);
            System.out.println("切换到slave2");
        }
    }
	}   
    
#### 设置determineCurrentLookupKey
    package com.cjs.example.bean;

    import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
    import org.springframework.lang.Nullable;

    public class MyRoutingDataSource extends AbstractRoutingDataSource {
        @Nullable
        @Override
        protected Object determineCurrentLookupKey() {
            return DBContextHolder.get();
        }
    }
    
#### 详解源码，为什么设置determineCurrentLookupKey就可以实现动态指定
首先看下AbstractRoutingDataSource类结构，继承了AbstractDataSource
     
     public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean
     
    public Connection getConnection() throws SQLException {  
        return determineTargetDataSource().getConnection();  
    }  
  
    public Connection getConnection(String username, String password) throws SQLException {  
        return determineTargetDataSource().getConnection(username, password);  
    }
   
   关键就在determineTargetDataSource()里：
   
   	/** 
     * Retrieve the current target DataSource. Determines the 
     * {@link #determineCurrentLookupKey() current lookup key}, performs 
     * a lookup in the {@link #setTargetDataSources targetDataSources} map, 
     * falls back to the specified 
     * {@link #setDefaultTargetDataSource default target DataSource} if necessary. 
     * @see #determineCurrentLookupKey() 
     */  
    protected DataSource determineTargetDataSource() {  
        Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");  
        Object lookupKey = determineCurrentLookupKey();  
        DataSource dataSource = this.resolvedDataSources.get(lookupKey);  
        if (dataSource == null && (this.lenientFallback || lookupKey == null)) {  
            dataSource = this.resolvedDefaultDataSource;  
        }  
        if (dataSource == null) {  
            throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");  
        }  
        return dataSource;  
    }
    
这里用到了我们需要进行实现的抽象方法determineCurrentLookupKey()，该方法返回需要使用的DataSource的key值，然后根据这个key从resolvedDataSources这个map里取出对应的DataSource，如果找不到，则用默认的resolvedDefaultDataSource。
	
    public void afterPropertiesSet() {  
                    if (this.targetDataSources == null) {  
                        throw new IllegalArgumentException("Property 'targetDataSources' is required");  
                    }  
                    this.resolvedDataSources = new HashMap<Object, DataSource>(this.targetDataSources.size());  
                    for (Map.Entry entry : this.targetDataSources.entrySet()) {  
                        Object lookupKey = resolveSpecifiedLookupKey(entry.getKey());  
                        DataSource dataSource = resolveSpecifiedDataSource(entry.getValue());  
                        this.resolvedDataSources.put(lookupKey, dataSource);  
                    }  
                    if (this.defaultTargetDataSource != null) {  
                        this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);  
                    }  
                }


#### spring注解配置
    package com.cjs.example.aop;

    import com.cjs.example.bean.DBContextHolder;
    import org.apache.commons.lang3.StringUtils;
    import org.aspectj.lang.JoinPoint;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Before;
    import org.aspectj.lang.annotation.Pointcut;
    import org.springframework.stereotype.Component;

    @Aspect
    @Component
    public class DataSourceAop {

    @Pointcut("!@annotation(com.cjs.example.annotation.Master) " +
            "&& (execution(* com.cjs.example.service..*.select*(..)) " +
            "|| execution(* com.cjs.example.service..*.get*(..)))")
    public void readPointcut() {

    }

    @Pointcut("@annotation(com.cjs.example.annotation.Master) " +
            "|| execution(* com.cjs.example.service..*.insert*(..)) " +
            "|| execution(* com.cjs.example.service..*.add*(..)) " +
            "|| execution(* com.cjs.example.service..*.update*(..)) " +
            "|| execution(* com.cjs.example.service..*.edit*(..)) " +
            "|| execution(* com.cjs.example.service..*.delete*(..)) " +
            "|| execution(* com.cjs.example.service..*.remove*(..))")
    public void writePointcut() {

    }

    @Before("readPointcut()")
    public void read() {
        DBContextHolder.slave();
    }

    @Before("writePointcut()")
    public void write() {
        DBContextHolder.master();
    }


        /**
         * 另一种写法：if...else...  判断哪些需要读从数据库，其余的走主数据库
         */
    //    @Before("execution(* com.cjs.example.service.impl.*.*(..))")
    //    public void before(JoinPoint jp) {
    //        String methodName = jp.getSignature().getName();
    //
    //        if (StringUtils.startsWithAny(methodName, "get", "select", "find")) {
    //            DBContextHolder.slave();
    //        }else {
    //            DBContextHolder.master();
    //        }
    //    }
    }
    
	package com.cjs.example.annotation;

		public @interface Master {
		}
       
#### 样例与测试

    package com.cjs.example.service.impl;

    import com.cjs.example.annotation.Master;
    import com.cjs.example.entity.Member;
    import com.cjs.example.entity.MemberExample;
    import com.cjs.example.mapper.MemberMapper;
    import com.cjs.example.service.MemberService;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;

    import java.util.List;

    @Service
    public class MemberServiceImpl implements MemberService {

        @Autowired
        private MemberMapper memberMapper;

        @Transactional
        @Override
        public int insert(Member member) {
            return memberMapper.insert(member);
        }

        @Master
        @Override
        public int save(Member member) {
            return memberMapper.insert(member);
        }

        @Override
        public List<Member> selectAll() {
            return memberMapper.selectByExample(new MemberExample());
        }

        @Master
        @Override
        public String getToken(String appId) {
            //  有些读操作必须读主数据库
            //  比如，获取微信access_token，因为高峰时期主从同步可能延迟
            //  这种情况下就必须强制从主数据读
            return null;
        }
	}
    package com.cjs.example;

    import com.cjs.example.entity.Member;
    import com.cjs.example.service.MemberService;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringRunner;

    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class CjsDatasourceDemoApplicationTests {

        @Autowired
        private MemberService memberService;

        @Test
        public void testWrite() {
            Member member = new Member();
            member.setName("zhangsan");
            memberService.insert(member);
        }

        @Test
        public void testRead() {
            for (int i = 0; i < 4; i++) {
                memberService.selectAll();
            }
        }

        @Test
        public void testSave() {
            Member member = new Member();
            member.setName("wangwu");
            memberService.save(member);
        }

        @Test
        public void testReadFromMaster() {
            memberService.getToken("1234");
        }

    }
