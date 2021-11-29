---
title: springboot多数据源
categories: springboot
tags: springboot多数据源
---



### springboot + mybatis

#### maven依赖
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>2.2.0</version>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
</dependency>
```

#### yml配置文件
```yml
# 数据源
spring:
  datasource:
    dudao: # duao数据源
      type: com.zaxxer.hikari.HikariDataSource
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/dudao?useUnicode=true&characterEncoding=utf8&nullCatalogMeansCurrent=true
      username: root
      password: root
    bpm: # bpm数据源
      type: com.zaxxer.hikari.HikariDataSource
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:3306/bpm?useUnicode=true&characterEncoding=utf8&nullCatalogMeansCurrent=true
      username: root
      password: root

# mybatis配置      
mybatis:
  mapper-locations:
  - classpath:mybatis/**/*.xml
  type-aliases-package: com.lo.multipools.domain
  configuration:
    map-underscore-to-camel-case: true
```

#### springboot配置类
```
// 主数据源
@Configuration
@MapperScan(basePackages = "com.lo.multipools.mapper.bpm", sqlSessionTemplateRef = "bpmSqlSessionTemplate")
public class DataSourceBpm {
	@Primary
	@Bean("bpmDataSourceProperties")
	@ConfigurationProperties(prefix = "spring.datasource.bpm")
	public DataSourceProperties bpmDataSourceProperties() {
		return new DataSourceProperties();
	}

	@Primary
	@Bean(name = "bpmDataSource")
	public DataSource bpmDataSource(@Qualifier("bpmDataSourceProperties") DataSourceProperties dataSourceProperties) {
		return dataSourceProperties.initializeDataSourceBuilder().build();
	}

	@Primary
	@Bean("bpmSqlSessionFactory")
	public SqlSessionFactory bpmSqlSessionFactory(@Qualifier("bpmDataSource") DataSource dataSource) throws Exception {
		SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
		sqlSessionFactory.setDataSource(dataSource);
		sqlSessionFactory.setMapperLocations(
		        // 配置mapper.xml路劲
				new PathMatchingResourcePatternResolver().getResources("classpath*:/mybatis/bpm/**/*.xml"));
		return sqlSessionFactory.getObject();
	}

	@Primary
	@Bean(name = "bpmTransactionManager")
	public DataSourceTransactionManager bpmTransactionManager(@Qualifier("bpmDataSource") DataSource dataSource) {
		return new DataSourceTransactionManager(dataSource);
	}

	@Primary
	@Bean(name = "bpmSqlSessionTemplate")
	public SqlSessionTemplate bpmSqlSessionTemplate(
			@Qualifier("bpmSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}
}

// dudao数据源
@Configuration
@MapperScan(basePackages = "com.lo.multipools.mapper.dudao", sqlSessionTemplateRef = "dudaoSqlSessionTemplate")
public class DataSourceDudao {
	@Bean(name = "dudaoDataSourceProperties")
	@ConfigurationProperties(prefix = "spring.datasource.dudao")
	public DataSourceProperties dudaoDataSourceProperties() {
		return new DataSourceProperties();
	}

	@Bean(name = "dudaoDataSource")
	public DataSource dudaoDataSource(
			@Qualifier("dudaoDataSourceProperties") DataSourceProperties dataSourceProperties) {
		return dataSourceProperties.initializeDataSourceBuilder().build();
	}

	@Bean("dudaoSqlSessionFactory")
	public SqlSessionFactory bpmSqlSessionFactory(@Qualifier("dudaoDataSource") DataSource dataSource)
			throws Exception {
		SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
		sqlSessionFactory.setDataSource(dataSource);
		sqlSessionFactory.setMapperLocations(
		        // 配置mapper.xml路劲
				new PathMatchingResourcePatternResolver().getResources("classpath*:/mybatis/dudao/**/*.xml"));
		return sqlSessionFactory.getObject();
	}

	@Bean(name = "dudaoTransactionManager")
	public DataSourceTransactionManager bpmTransactionManager(@Qualifier("dudaoDataSource") DataSource dataSource) {
		return new DataSourceTransactionManager(dataSource);
	}

	@Bean(name = "dudaoSqlSessionTemplate")
	public SqlSessionTemplate bpmSqlSessionTemplate(
			@Qualifier("dudaoSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}
}
```

### mybatis-plus 多数据源
@DS