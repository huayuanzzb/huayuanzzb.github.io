---
layout: post
title:  JDBC Statement
date:   2019-01-24 17:25:26 +0800
nav_order: 2
parent: java
categories: java
---

Java 提供了 3 个不同的接口与数据库交互:
* Statement: 执行 sql query, 仅支持静态 sql
* PreparedStatement: 执行 sql query, 支持参数化的查询
* CallableStatement: 执行存储过程或者函数

其中，Statement 和 PreparedStatement 较常用。

### Statement
1. 仅支持静态sql，如果有参数，必须拼接在sql中，易造成sql注入
2. 每次查询都会重新解析编译sql语句

### PreparedStatement
1. 支持参数化的动态sql，自动转义敏感字符，有效地防止sql注入
2. 首次查询会执行4个阶段，后续就只执行第4个阶段，提高性能

### 分析
jdbc 查询分为4个阶段：
1. 解析sql
2. 编译sql
3. 获取执行计划
4. 执行查询并返回数据

基于 Postgresql JDBC 驱动源码分析。
追溯```conn.prepareStatement("select * from _order where id = ?")```, 可以发现在创建PreparedStatement时，会执行
```java
// 解析sql，并转义其中的敏感字符
String parsed_sql = replaceProcessing(sql);
    if (isCallable)
        parsed_sql = modifyJdbcCall(parsed_sql);
// 创建参数化的Query
this.preparedQuery = connection.getQueryExecutor().createParameterizedQuery(parsed_sql);
this.preparedParameters = preparedQuery.createParameterList();

int inParamCount =  preparedParameters.getInParameterCount() + 1;
this.testReturn = new int[inParamCount];
this.functionReturnType = new int[inParamCount];
```
### 测试

#### 代码

```java
import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.pool.DruidPooledConnection;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;

public class TestDruid {

    private static String url = "jdbc:postgresql://localhost:5432/test";
    private static String user = "test";
    private static String pass = "test";

    public static void main(String[] args) {
        for (int i = 0; i < 20; i++) {
            System.out.println("test " + i);
            testPreparedStatement();
            testStatement();
            System.out.println("-------------");
        }
    }

    static void testPreparedStatement() {
        List<Integer> ids = initIds();
        Connection conn = null;
        PreparedStatement statement = null;
        ResultSet rs = null;
        int count = 0;
        try{
            conn = getConnection();
            statement = conn.prepareStatement("select * from test where id = ?");
            long start = System.currentTimeMillis();
            for(Integer id : ids){
                statement.setInt(1, id);
                rs = statement.executeQuery();
                if(rs.next()){
                    rs.getString(2);
                    count++;
                }
            }
            System.out.println("prepared statement fetch count " + count + " cost: " + (System.currentTimeMillis() - start) + " ms");
        } catch (Exception e){
            //
        }finally {
            close(conn, statement, rs);
        }
    }

    static void testStatement(){
        List<Integer> ids = initIds();
        Connection conn = null;
        Statement statement = null;
        ResultSet rs = null;
        int count = 0;
        try{
            conn = getConnection();
            statement = conn.createStatement();
            long start = System.currentTimeMillis();
            for(Integer id : ids){
                rs = statement.executeQuery("select * from test where id = " + id);
                if(rs.next()){
                    rs.getString(2);
                    count++;
                }
            }
            System.out.println("statement fetch count " + count + " cost: " + (System.currentTimeMillis() - start) + " ms");
        } catch (Exception e){
            //
        }finally {
            close(conn, statement, rs);
        }
    }

    private static void close(Connection conn, Statement statement, ResultSet rs){
        if(conn != null){
            try {
                rs.close();
                statement.close();
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    private static Connection getConnection() throws Exception {
        Class.forName("org.postgresql.Driver");
        return DriverManager.getConnection(url, user, pass);
    }

    private static List<Integer> initIds() {
        List<Integer> list = new ArrayList<>();
        while (list.size() < 1000) {
            Integer ii = Math.abs(new Random().nextInt(100000000));
            if(!list.contains(ii)){
                list.add(ii);
            }
        }
        return list;
    }
}
```
#### 结果

| case | PreparedStatement | Statement |
| --- | --- | --- |
| test 0 | fetch count 129 cost: 6586 ms | fetch count 131 cost: 11426 ms |
| test 1 | fetch count 107 cost: 7017 ms | fetch count 116 cost: 7791 ms |
| test 2 | fetch count 129 cost: 7017 ms | fetch count 148 cost: 8191 ms |
| test 3 | fetch count 125 cost: 7223 ms | fetch count 130 cost: 9176 ms |
| test 4 | fetch count 124 cost: 6866 ms | fetch count 133 cost: 8228 ms |
| test 5 | fetch count 116 cost: 6677 ms | fetch count 141 cost: 8097 ms |
| test 6 | fetch count 117 cost: 6074 ms | fetch count 131 cost: 7915 ms |
| test 7 | fetch count 135 cost: 6858 ms | fetch count 136 cost: 7875 ms |
| test 8 | fetch count 141 cost: 6847 ms | fetch count 113 cost: 7323 ms |
| test 9 | fetch count 126 cost: 6529 ms | fetch count 142 cost: 8472 ms |
| test 10 | fetch count 135 cost: 6650 ms | fetch count 143 cost: 11611 ms |
| test 11 | fetch count 127 cost: 13707 ms | fetch count 133 cost: 8699 ms |
| test 12 | fetch count 122 cost: 7302 ms | fetch count 114 cost: 7424 ms |
| test 13 | fetch count 128 cost: 6232 ms | fetch count 128 cost: 7099 ms |
| test 14 | fetch count 124 cost: 6296 ms | fetch count 139 cost: 7499 ms |
| test 15 | fetch count 124 cost: 6548 ms | fetch count 125 cost: 7512 ms |
| test 16 | fetch count 123 cost: 6148 ms | fetch count 134 cost: 7686 ms |
| test 17 | fetch count 126 cost: 6434 ms | fetch count 132 cost: 7257 ms |
| test 18 | fetch count 123 cost: 5508 ms | fetch count 113 cost: 8778 ms |
| test 19 | fetch count 135 cost: 6103 ms | fetch count 133 cost: 9025 ms |
