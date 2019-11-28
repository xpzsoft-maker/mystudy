#SpringBootLog4j2

## 1. 依赖库

```xml
  <!-- 去掉spring自己提供的logback依赖 -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
      <exclusion>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
      </exclusion>
    </exclusions>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
    <version>2.9.9</version>
  </dependency>
```

## 2. 配置日志
命名log4j2-dev.yml，内容如下
```yml
Configuration:
  status: warn
  monitorInterval: 30

  Properties: # 定义全局变量
    Property: # 缺省配置（用于开发环境）。其他环境需要在VM参数中指定，如下：
      - name: log.level.console
        value: info
      - name: log.path
        value: logs
      - name: project.name
        value: gateway
      - name: log.pattern
        value: "${project.name}%d{HH:mm:ss.SSS} %-5level %class %L %M - %msg%n"

  Appenders:
    Console:  #输出到控制台
      name: CONSOLE
      target: SYSTEM_OUT
      PatternLayout:
        pattern: ${log.pattern}
#   全日志
    RollingFile:
      - name: ALL_FILE
        ignoreExceptions: false
        fileName: ${log.path}/all/${project.name}_all.log
        filePattern: "${log.path}/all/$${date:yyyy-MM}/all-%d{yyyy-MM-dd}-%i.log.gz"
        PatternLayout:
          pattern: ${log.pattern}
        Policies:
          TimeBasedTriggeringPolicy:  # 按天分类
            modulate: true
            interval: 1
        DefaultRolloverStrategy:     # 文件最多100个
          max: 100
#   平台日志
      - name: PLATFORM_FILE
        ignoreExceptions: false
        fileName: ${log.path}/${project.name}/${project.name}_only.log
        filePattern: "${log.path}/${project.name}/$${date:yyyy-MM}/only-%d{yyyy-MM-dd}-%i.log.gz"
        PatternLayout:
          pattern: ${log.pattern}
        Policies:
          TimeBasedTriggeringPolicy:  # 按天分类
            modulate: true
            interval: 1
        DefaultRolloverStrategy:     # 文件最多100个
          max: 100
#   异常日志
      - name: ERROR_FILE
        ignoreExceptions: false
        fileName: ${log.path}/error/${project.name}_error.log
        filePattern: "${log.path}/${project.name}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz"
        PatternLayout:
          pattern: ${log.pattern}
        Filters:
          ThresholdFilter:
            - leve: error
              onMatch: ACCEPT
              onMismatch: DENY
        Policies:
          TimeBasedTriggeringPolicy:  # 按天分类
            modulate: true
            interval: 1
        DefaultRolloverStrategy:     # 文件最多100个
          max: 100
#   DB 日志
      - name: DB_FILE
        ignoreExceptions: false
        fileName: ${log.path}/db/${project.name}_db.log
        filePattern: "${log.path}/db/$${date:yyyy-MM}/db-%d{yyyy-MM-dd}-%i.log.gz"
        PatternLayout:
          pattern: ${log.pattern}
        Policies:
          TimeBasedTriggeringPolicy:  # 按天分类
            modulate: true
            interval: 1
        DefaultRolloverStrategy:     # 文件最多100个
          max: 100


  Loggers:

    Logger:
      - name: org.hibernate.SQL
        additivity: false
        level: debug
        AppenderRef:
          - ref: CONSOLE
          - ref: DB_FILE
          - ref: ALL_FILE
      - name: com.avic.mti
        additivity: false
        level: debug
        AppenderRef:
          - ref: CONSOLE
          - ref: ALL_FILE
          - ref: ERROR_FILE
          - ref: PLATFORM_FILE

    Root:
      level: info
      AppenderRef:
        - ref: CONSOLE
        - ref: ALL_FILE
        - ref: ERROR_FILE
```
## 3. SpringBoot集成
application.yml
```yml
logging:
  config: classpath:log4j2-dev.yml
```