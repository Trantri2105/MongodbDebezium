# CDC Mongodb với Debezium

## Giới thiệu

Project sử dụng debezium và kafka để đồng bộ hóa dữ liệu mongodb

## Các thành phần trong docker compose

- Zookeeper: có vai trò quản lý và điều phối các broker trong một kafka cluster
- Kafka: là một message broker, có nhiệm vụ chuyển message từ producer sang consumer
- Mongodb1, mongodb2, mongodb3: replica set chứa data gốc
- Mongo: database cần đồng bộ dữ liệu
- Debezium: debezium là một kafka connect, sử dụng các connector để trao đổi dữ liệu giữa database và kafka. Có hai loại connector:

    - Source connector: theo dõi dữ liệu từ database
    - Sink connector: truyển dữ liệu từ kafka đến database

## Work flow

1. Khi datababe gốc có sự thay đổi về dữ liệu, debezium sẽ sử dụng các source connector để phát hiện những thay đổi đã xảy ra với từng record.
2. Debezium sẽ chuyển những thay đổi này sang message và gửi lên kafka.
3. Khi message đã được gửi lên kafka, ta có thể viết chương trình hoặc dùng các sink connector của debezium để đọc message và thực hiện những thay đổi này lên database.

## Set up

1. Tải plugin [mongodb kafka connector](https://www.confluent.io/hub/mongodb/kafka-connect-mongodb) và [debezium mongodb connector](https://debezium.io/documentation/reference/stable/install.html), giải nén và lưu vào folder ./kafka/connect
2. Chạy file docker compose
3. Debezium theo dõi sự thay đổi bằng cách đọc oplog(operation log) của primary server trong một replica set. Vậy nên để sử dụng debezium với mongodb, ta cần tạo replica set
   ``` 
    docker exec -it mongodb1 mongosh --eval "rs.initiate({
    _id: \"rs\",
    members: [
     {_id: 0, host: \"mongodb1\"},
     {_id: 1, host: \"mongodb2\"},
     {_id: 2, host: \"mongodb3\"}
     ]
    })"
   ```
4. Tạo source connector để debezium theo dõi database gốc
   ``` 
   curl --location 'http://localhost:8083/connectors' \
    --header 'Content-Type: application/json' \
    --data '{
     "name": "mongodb-source-connector",
     "config": {
      "connector.class": "io.debezium.connector.mongodb.MongoDbConnector",
      "mongodb.connection.string": "mongodb://mongodb1:27017,mongodb2:27017,mongodb3:27017/?replicaSet=rs",
      "topic.prefix": "source",
      "database.include.list": "public",
      "collection.include.list": "user,product"
     }
    }'
   ```
   - Với mỗi collection, debezium sẽ tạo một topic để gửi dữ liệu theo format `<topic.prefix>.<database>.<collection>`
   - Đọc thêm các property để config source connector tại [source connector properties](https://debezium.io/documentation/reference/stable/connectors/mongodb.html#mongodb-connector-properties)
5. Tạo sink connector để debezium consume message và đồng bộ database
   ```
   curl --location 'http://localhost:8083/connectors' \
    --header 'Content-Type: application/json' \
    --data '{
     "name": "mongodb-sink-connector",
     "config": {
     "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
     "tasks.max": 2,
     "connection.uri":"mongodb://mongo:27017",
     "database":"public",
     "topics": "source.public.user,source.public.product",
     "topic.override.source.public.user.collection":"user",
     "topic.override.source.public.product.collection":"product",
     "change.data.capture.handler":"com.mongodb.kafka.connect.sink.cdc.debezium.mongodb.MongoDbHandler"
     }
    }'
   ```
   - `"topic.override.source.public.user.collection": "user"`: để map topic source.public.user cho collection user
   - `tasks.max`: số tác vụ chạy song song. Mỗi một partition sẽ được gán cho đúng 1 task. Nếu số task max > số partition thì số task dùng = số partition. Mặc định thì debezium sẽ tạo một partition cho một topic
   - Đọc thêm các property để config sink connector tại [sink connector properties](https://www.mongodb.com/docs/kafka-connector/v1.8/sink-connector/configuration-properties/)

## Trường hợp database cần được đồng bộ bị sập

- Vì message chỉ được lưu trong kafka trong một khoảng thời gian nhất định nên nếu trong trường hợp thời gian database gặp sự cố < thời gian message được lưu, ta chỉ cần restart lại sink connector để tiếp tục process các message chưa consume:
   1. Khi database hoạt động trở lại, kiểm tra tình trạng của các task trong sink connector
   ``` 
   curl --location 'http://localhost:8083/connectors/mongodb-sink-connector/status' --data ''
   ```
   2. Restart lần lượt lại các task failed
   ```
   curl --location --request POST 'http://localhost:8083/connectors/mongodb-sink-connector/tasks/0/restart'
   ```
- Trong trường hợp thời gian database gặp sự cố > thời gian message được lưu, ta cần phải tạo mới source và sink connector, đồng bộ lại từ đầu.Vì debezium sẽ tạo thread cho mỗi connector nên để tối ưu tài nguyên, ta có thể xóa đi các connector cũ
   - Xem các connector hiện có
   ``` 
   curl --location 'http://localhost:8083/connectors'
   ```
   - Xóa connector
   ```
   curl --location --request DELETE 'http://localhost:8083/connectors/mongodb-sink-connector' 
   ```


## Trường hợp database mất dữ liệu và cần được đồng bộ lại từ đầu
- Vì message chỉ được lưu trong kafka trong một khoảng thời gian nhất định nên nếu trong trường hợp thời gian từ khi bắt đầu tạo source connector đến khi database mất dữ liệu < thời gian message được lưu, ta chỉ cần tạo lại sink connector:
   1. Tạo sink connector mới
   2. Vì debezium sẽ tạo thread cho mỗi connector nên để tối ưu tài nguyên, ta có thể xóa đi các connector cũ
      - Xem các connector hiện có
      ``` 
      curl --location 'http://localhost:8083/connectors'
      ```
      - Xóa connector
      ```
      curl --location --request DELETE 'http://localhost:8083/connectors/mongodb-sink-connector' 
      ```
- Trong trường hợp thời gian từ khi bắt đầu tạo source connector đến khi database mất dữ liệu > thời gian message được lưu ta cần phải tạo mới source và sink connector, đồng bộ lại từ đầu