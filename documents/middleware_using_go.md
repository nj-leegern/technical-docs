# Golang常用的中间件使用总结



- Gin(github.com/gin-gonic/gin)：web开发框架，适合api接口、微服务开发，相较于其他框架(iris、beego)更轻量级和更好的性能。其路由功能很强大提供分组功能，非常适合做api开发，具体demo如下：

  ```go
      route := gin.Default()
      // get method
      route.GET("/testGet", func(c *gin.Context) {
          name := c.Query("name")
          c.String(http.StatusOK, "hello " + name)
      })
  
      // post method
      route.POST("/testPost", func(c *gin.Context) {
          param := Param{}
          if err := c.BindJSON(&param); err != nil {
              c.JSON(http.StatusBadRequest, gin.H{
                  "code": 1,
                  "msg" : "error",
              })
              return
          }
          s := fmt.Sprintf("name:%s, age:%s", param.Name, param.Age)
          c.JSON(http.StatusOK, gin.H{
              "code": 0,
              "msg": s,
          })
      })
  
      // route group
      v1 := route.Group("/v1")
      {
          v1.POST("/getInfo", func(c *gin.Context) {
              name := c.PostForm("name")
              c.JSON(http.StatusOK, gin.H{"msg": "hello " + name})
          })
      }
  
      v2 := route.Group("/v2")
      {
          v2.POST("/getInfo", func(c *gin.Context) {
              name := c.PostForm("name")
              age := c.PostForm("age")
              s := fmt.Sprintf("hello %s, i'm %s", name, age)
              c.JSON(http.StatusOK, gin.H{"msg": s})
          })
      }
  
      route.Run(":8080")
  ```

  

- Mysql(github.com/go-sql-driver/mysql)：mysql第三方开源库的实现，提供原生sql功能，喜欢框架的童鞋可以去看看国人写的Gorm，挺不错的一个orm框架（我还是比较习惯写原生sql ^ _ ^ ）:

  ```go
      // init
      dataSourceName := fmt.Sprintf("%s:%s@tcp(%s:3306)/iray_proxy?charset=utf8mb4", "root", "root123", "127.0.0.1")
      conn, err := sql.Open("mysql", dataSourceName)
      if err != nil {
          panic(err)
      }
      db.SetMaxOpenConns(30)
      db.SetMaxIdleConns(10)
      db.SetConnMaxLifetime(10 * time.Minute)
      db.Ping()
  
      // select
      rows, err1 := conn.Query("select name, age from t_user where age > ? and age < ?", 20, 30)
      defer rows.Close()
      if err1 != nil {
          panic(err1)
      }
      result := make([]map[string] interface{}, 0)
      for rows.Next() {
          var name string
          var age  int
  
          err1 = rows.Scan(&name, &age)
          if err1 != nil {
              log.Printf("row scan error: %s", err1.Error())
          } else {
              record := make(map[string] interface{}, 1)
              record["name"] = name
              record["age"]  = age
              result = append(result, record)
          }
      }
  
      // insert
      stmt, err2 := conn.Prepare("insert into t_user (name, age) values (?, ?)")
      if err2 != nil {
          panic(err2)
      }
      rs, err3 := stmt.Exec("jack", 25)
      if err3 != nil {
          log.Printf("insert row error: %s", err3.Error())
      } else {
          rowCount, _ := rs.RowsAffected()
          log.Printf("insert row count: %s", strconv.FormatInt(rowCount, 10))
      }
  
      // update
      stm1, err4 := conn.Prepare("update t_user set age = ? where name = ?")
      if err4 != nil {
          panic(err4)
      }
      rs1, err5 := stm1.Exec(30, "jack")
      if err5 != nil {
          log.Printf("update row error: %s", err5.Error())
      } else {
          rowCount, _ := rs1.RowsAffected()
          log.Printf("update row count: %s", strconv.FormatInt(rowCount, 10))
      }
  
      // delete
      rs2, err6 := conn.Exec("delete from t_user where name = ?", "jack")
      if err6 != nil {
          panic(err6)
      }
      rowCount, _ := rs2.RowsAffected()
      log.Printf("delete row count: %s", strconv.FormatInt(rowCount, 10))
  ```

  

- Redis(github.com/go-redis/redis)：redis客户端的第三方开源库，比较出名两个中的一个，还有个是redigo(github.com/garyburd/redigo/redis)，喜欢谁就用谁，我使用的是go-redis，具体操作如下：

  ```go
      // Init
      redisClient := redis.NewClient(&redis.Options{
          Addr:     "127.0.0.1:6379",
          Password: "123456",
          DB:       1,
          PoolSize: 50,
      })
  
      // Hash
      err := redisClient.HSet("hash_key", "name", "jason statham").Err()
      if err != nil {
          log.Printf("hset error: %s", err)
      }
      resp := redisClient.HGet("hash_key", "name").Val()
      log.Printf("hget result: %s", resp)
  
      // List
      err = redisClient.RPush("list_key", "adam levin").Err()
      if err != nil {
          log.Printf("right push error: %s", err)
      }
      rs, err := redisClient.LPop("list_key").Result()
      log.Printf("left pop : %s", rs)
  
      // Set
      err = redisClient.SAdd("set_key", "harry kane").Err()
      if err != nil {
          log.Printf("add error: %s", err)
      }
      resultSet, err := redisClient.SMembers("set_key").Result()
      if err != nil {
          log.Printf("smembers error : %s", err)
      } else {
          for i, item := range resultSet {
              log.Printf("smembers item[%s] : %s", strconv.Itoa(i), item)
          }
      }
  
      // SortSet
      item := redis.Z{Score: float64(time.Now().Unix()), Member: "jet brains"}
      count, err := redisClient.ZAdd("sortset_key", item).Result()
      if err != nil {
          log.Printf("zadd error : %s", err)
      } else {
          log.Printf("zadd count : %s", strconv.FormatInt(count, 10))
      }
      results, err := redisClient.ZRangeByScore("sortset_key", redis.ZRangeBy{Max:                                                                       strconv.FormatInt(time.Now().Unix(), 10)}).Result()        
      if err != nil {
          log.Printf("zrangeByScore error : %s", err)
      } else {
          for i, item := range results {
              log.Printf("zrangeByScore item[%s] : %s", strconv.Itoa(i), item)
          }
      }
  ```

  

- Zookeeper(github.com/samuel/go-zookeeper/zk)：分布式协作服务客户端第三方开源库的实现，具体代码如下：

  ```go
      // watch
      option := zk.WithEventCallback(func(event zk.Event) {
          eventType := event.Type.String()
          log.Printf("event name : %s", eventType)
      })
      // acl
      acl := zk.WorldACL(zk.PermAll)
      // root path
      path := "/go/zookeeper"
  
      conn,_,err := zk.Connect([]string{"127.0.0.1:2181"}, 5 * time.Second, option)
      if err != nil {
          panic(err)
      }
  
      // create znode
      _, err = conn.Create(path, []byte("duang"), 0, acl)
      if err != nil {
          log.Printf("create znode error : %s", err)
      }
  
      // set
      _, err = conn.Set(path, []byte("duang duang"), 1)
      if err != nil {
          log.Printf("set error : %s", err)
      }
  
      // get node and set watch
      data, stat, _, _ := conn.GetW(path)
      if data != nil {
          log.Println("zk get response:", string(data), ", stat:", stat.Version)
      }
  
      // exist znode
      exist, _, _, existErr := conn.ExistsW(path)
      if existErr != nil {
          log.Printf("exist znode error: %s", existErr)
          return
      }
      log.Printf("exist znode response : %s", strconv.FormatBool(exist))
  
      // watch children
      children, _, _, err := conn.ChildrenW(path)
      if err != nil {
          log.Printf("watch children error: %s", err)
      }
      if len(children) > 0 {
          for item := range children {
              log.Printf("watch children : %s", item)
          }
      }
  ```

  

- Kafka(github.com/Shopify/sarama)：kafka的第三方客户端实现，生产者和消费者实现如下：

  ```go
      // producer
      config := sarama.NewConfig()
      config.Producer.RequiredAcks = sarama.WaitForAll
      config.Producer.Partitioner = sarama.NewRandomPartitioner
      config.Producer.Return.Successes = true
  
      producer, err := sarama.NewSyncProducer([]string{"127.0.0.1:9092"}, config)
      if err != nil {
          panic(err)
      }
      defer producer.Close()
  
      // send msg
      param := make(map[string]string)
      param["name"] = "jack"
      param["sex"] = "male"
  
      js, err := json.Marshal(param)
      if err != nil {
          log.Printf("json err: %s", err.Error())
      } else {
          msg := &sarama.ProducerMessage{}
          msg.Topic = "kafka_test"
          msg.Value = sarama.ByteEncoder(js)
  
          partition, offset, err := producer.SendMessage(msg)
          if err != nil {
              log.Printf("failed to produce message :%s", err.Error())
          }
          log.Printf("partition:%d, offset: %d", partition, offset)
      }
  
      // consumer
      var wg sync.WaitGroup
      consumer, err := sarama.NewConsumer([]string{"127.0.0.1:9092"}, nil)
      if err != nil{
          panic(err)
      }
      defer consumer.Close()
  
      partitionList, err := consumer.Partitions("kafka_test")  // get topic all partitions
      if err != nil{
          log.Println("Failed to get the list of partition: ",err)
          return
      }
      // recieve msg
      for partition := range partitionList{
          pc, err := consumer.ConsumePartition("kafka_test", int32(partition), sarama.OffsetNewest)
          if err != nil{
              log.Printf("Failed to start consumer for partition %d: %s", partition, err)
              return
          }
          wg.Add(1)
          go func(sarama.PartitionConsumer) {
              for msg := range pc.Messages(){
                  log.Printf("Partition:%d, Offset:%d, key:%s, value:%s", msg.Partition, msg.Offset,                                                                              string(msg.Key),string(msg.Value))
              }
              defer pc.AsyncClose()
              wg.Done()
          }(pc)
      }
      wg.Wait()
  ```

  

- ElasticSearch(github.com/olivere/elastic)：ES的第三方客户端开源库，主要提供增、删、改、查和搜索的实现：

  ```go
      // init
      esClient, err := elastic.NewClient(
          elastic.SetURL("http://127.0.0.1:9200"),
          elastic.SetScheme("http"),
          // health check
          elastic.SetHealthcheck(true),
          elastic.SetHealthcheckTimeoutStartup(5*time.Second),
          elastic.SetHealthcheckTimeout(2*time.Second),
          elastic.SetHealthcheckInterval(60*time.Second),
          // sniffer
          elastic.SetSniff(true),
          elastic.SetSnifferInterval(15*time.Minute),
          elastic.SetSnifferTimeoutStartup(5*time.Second),
          elastic.SetSnifferTimeout(2*time.Second),
          elastic.SetSendGetBodyAs("GET"),
      )
      if err != nil {
          panic(err)
      }
  
      // create
      param := `{"first_name":"jason","last_name":"kid","age":40}`
      resp, err := esClient.Index().
          Index("es_test").
          Type("employee").
          Id("1").
          BodyJson(param).
          Do(context.Background())
      if err != nil {
          log.Printf("create error: %s", err)
      } else {
          log.Printf("Indexed tweet %s to index s%s, type %s", resp.Id, resp.Index, resp.Type)
      }
  
      // update
      res, err := esClient.Update().
          Index("es_test").
          Type("employee").
          Id("1").
          Doc(map[string]interface{}{"age": 88}).
          Do(context.Background())
      if err != nil {
          log.Printf("update error: %s", err)
      } else {
          log.Printf("update age %s\n", res.Result)
      }
  
      // delete
      resDel, err := esClient.Delete().
          Index("es_test").
          Type("employee").
          Id("1").
          Do(context.Background())
      if err != nil {
          log.Printf("delete error: %s", err)
      } else {
          log.Printf("delete result %s", resDel.Result)
      }
  
      // get
      resGet, err := esClient.Get().
          Index("es_test").
          Type("employee").
          Id("1").
          Do(context.Background())
      if err != nil {
          log.Printf("get error: %s", err)
      } else if resGet.Found {
          log.Printf("Get document %s in version %d from index %s, type %s", resGet.Id, resGet.Version,                                                                                resGet.Index, resGet.Type)
      }
  
      // search
      resQuery, err := esClient.Search("es_test").
          Type("employee").
          Query(elastic.NewQueryStringQuery("first_name:jason")).
          Do(context.Background())
      if err != nil {
          log.Printf("query error: %s", err)
      } else if resQuery.Hits.TotalHits > 0 {
          for _, hit := range resQuery.Hits.Hits {
              b,_ := hit.Source.MarshalJSON()
              log.Printf("employee item : %s ", string(b))
          }
      }
  ```

  

- Cron(github.com/robfig/cron)：定时任务第三方库，支持克隆表达式，很强大：

  ```go
      c := cron.New()
      c.AddFunc("*/30 * * * * ?", func() {
          // to do something
          log.Println("run task ...")
      })
      c.Start()
  ```

  
