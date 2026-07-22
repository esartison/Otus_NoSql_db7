# Домашнее задание Сартисона Евгения N7 #

Необходимо:

сохранить большой жсон (~20МБ) в виде разных структур - строка, hset, zset, list;
протестировать скорость сохранения и чтения;
предоставить отчет.




## развернуть БД Redis в Docker (Некластерный режим) ## 

Все работы выполняю локальной инсталяции Docker-а в Virtual Box VM

создать директорию для данных
> mkdir -p ~/redis_data


Запустить экземпляр Redis 
```
docker run -d \
  --name redis-server \
  -p 6379:6379 \
  -v ~/redis_data:/data \
  --restart always \
  redis:latest \
  redis-server --requirepass "pwd123"
```
<img width="1490" height="528" alt="image" src="https://github.com/user-attachments/assets/a5d4608e-8217-4e8c-a62d-2733c919296b" />


выполнить тестовое подключение
```
docker exec -it redis-server redis-cli -a "pwd123"
redis-cli -h localhost -p 6379 --pass "pwd123"
```
<img width="622" height="491" alt="image" src="https://github.com/user-attachments/assets/30ef217b-77ec-4f41-8866-a6ea3f74e5fc" />



## сохранить большой жсон (~20МБ) в виде разных структур - строка, hset, zset, list; ## 
С помощью  [json-mock-generator](https://dataformatterpro.com/json-mock-generator/) создал дата сет примерно на 20Mb
<img width="539" height="406" alt="image" src="https://github.com/user-attachments/assets/9abd30b2-4575-47c5-a4d7-966d9e3f91bf" />

Сделал загрузку этого Json в разные типы данных.

# LIST #
питон скрипт
```
import json
import redis

# Connect to your local Redis instance
r = redis.Redis(host='localhost', port=6379,  password='pwd123', decode_responses=True)

# Define your target list key and source file
list_key = "my_json_list"
file_path = "Redis_Json.json"

with open(file_path, "r") as f:
    # Load the JSON array from the file
    data_list = json.load(f) 
    
    # Use a pipeline to batch the write operations for performance
    pipe = r.pipeline()
    for item in data_list:
        # Serialize the individual item back to a string and push
        pipe.rpush(list_key, json.dumps(item))
        
    pipe.execute()

print(f"Successfully loaded {len(data_list)} elements into '{list_key}'.")
```
загрузка данных
```
student:~$ time python3 python_import_list.py 
Successfully loaded 120000 elements into 'my_json_list'.

real	0m1,913s
user	0m1,673s
sys	0m0,199s


localhost:6379> type my_json_list
list
localhost:6379> llen my_json_list
(integer) 120000
localhost:6379> flushall
OK
```


# STRING #
загрузка данных
```
student:~$ time jq -c . Redis_Json.json | redis-cli -h localhost -p 6379 --pass "pwd123" -x SET my_json_string
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
OK

real	0m1,028s
user	0m0,751s
sys	0m0,345s

localhost:6379> type my_json_string
string
localhost:6379> MEMORY USAGE my_json_string SAMPLES 0
(integer) 20971560
localhost:6379> flushall
OK
```


# HSET #
питон скрипт
```
import json
import redis

r = redis.Redis(host='localhost', port=6379,  password='pwd123', decode_responses=True)

with open('Redis_Json.json') as f:
    products = json.load(f)

pipeline = r.pipeline()
for product in products:
    key = f"product:{product['id']}"
    # Flatten nested objects to strings for hash storage
    flat = {k: str(v) for k, v in product.items()}
    pipeline.hset(key, mapping=flat)
    if 'ttl' in product:
        pipeline.expire(key, product['ttl'])

pipeline.execute()
print(f"Imported {len(products)} products")
```
загрузка данных
```
student:~$ time python3 python_import_hset.py
Imported 120000 products

real	0m3,315s
user	0m2,938s
sys   	0m0,227s
```

# ZSET #
питон скрипт
```
import json
import redis

# Connect to your local or remote Redis server
client = redis.Redis(host='localhost', port=6379,  password='pwd123', decode_responses=True)

# Define your target Redis Sorted Set key
zset_key = "my_zset_collection"

with open('Redis_Json.json', 'r') as file:
    items = json.load(file)

pipeline = client.pipeline()

for item in items:
    id = item['id']
    name = item['name'] if isinstance(item['name'], str) else json.dumps(item['name'])
    
    pipeline.zadd(zset_key, {name: id})

# Execute all batched insertions at once
pipeline.execute()
print(f"Successfully imported {len(items)} items into the ZSET '{zset_key}'.")
```
загрузка данных
```
student:~$ time python3 python_import_zset.py
Successfully imported 120000 items into the ZSET 'my_zset_collection'.

real	0m1,676s
user	0m1,485s
sys	    0m0,109s
```



## протестировать скорость сохранения и чтения; ## 
Запись уже тестировали в предыдущем шаге, протестируем чтение в этом шаге

# LIST #
Чтение всей коллекции
```
student:~$ time redis-cli -h localhost -p 6379 --pass "pwd123" LRANGE my_json_list 0 -1
119998) "{\"id\": 9444, \"name\": \"Anna Williams\", \"email\": \"anna.williams@example.com\", \"age\": 28, \"active\": false, \"balance\": 2506.54, \"createdAt\": \"2026-04-01T14:16:57.149Z\"}"
119999) "{\"id\": 4670, \"name\": \"Lisa Johnson\", \"email\": \"lisa.johnson@demo.com\", \"age\": 37, \"active\": true, \"balance\": 1553.98, \"createdAt\": \"2025-11-17T14:16:57.149Z\"}"
120000) "{\"id\": 2254, \"name\": \"John Martinez\", \"email\": \"john.martinez@test.com\", \"age\": 62, \"active\": true, \"balance\": 1036.01, \"createdAt\": \"2025-12-29T14:16:57.149Z\"}"

real	0m9,330s
user	0m1,162s
sys	0m0,708s
```
Статистика
```
student:~$ redis-cli -h localhost -p 6379 --pass "pwd123" INFO commandstats
cmdstat_lrange:calls=3,usec=44180,usec_per_call=14726.67,rejected_calls=1,failed_calls=1,slowlog_count=1,slowlog_time_ms_sum=43.46,slowlog_time_ms_max=43.46
```


# STRING #
Чтение всей коллекции
```
 time redis-cli -h localhost -p 6379 --pass "pwd123" --scan --pattern '*' | while read key; do echo "Key: $key"; redis-cli --pass "pwd123" GET "$key"; done

 \":\"Jane Johnson\",\"email\":\"jane.johnson@test.com\",\"age\":43,\"active\":true,\"balance\":7381.41,\"createdAt\":\"2026-06-24T14:16:57.149Z\"},{\"id\":1366,\"name\":\"Tom Rodriguez\",\"email\":\"tom.rodriguez@example.com\",\"age\":33,\"active\":true,\"balance\":5828.1,\"createdAt\":\"2026-02-07T14:16:57.149Z\"},{\"id\":9444,\"name\":\"Anna Williams\",\"email\":\"anna.williams@example.com\",\"age\":28,\"active\":false,\"balance\":2506.54,\"createdAt\":\"2026-04-01T14:16:57.149Z\"},{\"id\":4670,\"name\":\"Lisa Johnson\",\"email\":\"lisa.johnson@demo.com\",\"age\":37,\"active\":true,\"balance\":1553.98,\"createdAt\":\"2025-11-17T14:16:57.149Z\"},{\"id\":2254,\"name\":\"John Martinez\",\"email\":\"john.martinez@test.com\",\"age\":62,\"active\":true,\"balance\":1036.01,\"createdAt\":\"2025-12-29T14:16:57.149Z\"}]\n"

real	0m6,158s
user	0m1,357s
sys	0m0,197s
```

# HSET #
Чтение всей коллекции и одной записи
```
-- все записи
time redis-cli -h localhost -p 6379 --pass "pwd123" --scan --pattern "product:*" | while read -r key; do 
    echo "Key: $key"
    redis-cli --pass "pwd123" HGETALL "$key"
done
real	1m11,889s
user	0m27,422s
sys	0m28,587s

-- одна запись
student:~$ time redis-cli -h localhost -p 6379 --pass "pwd123" HGETALL product:2788
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
 1) "id"
 2) "2788"
 3) "name"
 4) "Chris Williams"
 5) "email"
 6) "chris.williams@mail.com"
 7) "age"
 8) "38"
 9) "active"
10) "True"
11) "balance"
12) "6901.79"
13) "createdAt"
14) "2026-03-20T14:16:57.148Z"

real	0m0,019s
user	0m0,005s
sys	0m0,010s
```


# ZSET #
```
time redis-cli -h localhost -p 6379 --pass "pwd123" ZRANGE my_zset_collection 0 -10 WITHSCORES
156) "8343"
157) "Anna Rodriguez"
158) "8549"
159) "Emma Jones"
160) "8573"
161) "Anna Jones"
162) "8615"
163) "Mike Williams"
164) "8621"
165) "Sarah Smith"
166) "8627"
167) "Sarah Rodriguez"
168) "8715"
169) "David Jones"
170) "8740"
171) "Mike Garcia"
172) "8788"
173) "Emma Brown"
174) "8952"
175) "Lisa Brown"
176) "8961"
177) "Chris Brown"
178) "9051"
179) "Mike Rodriguez"
180) "9163"
181) "John Smith"
182) "9214"

real	0m0,022s
user	0m0,005s
sys	0m0,013s
```

## предоставить отчет. ## 

| Алгоритм | Запись | Чтение |
|----------|--------|--------|
| List     |  1.9s  |  9.3s  |
| String   |  1.02s |  6.2s  |
| HSET     |  3.3s  |  1m35s |
| ZSET     |  1.6s  |  0.01s |

ZSET показал лучший результат в моей загрузке.

Сравнение может не совсем точное, но сравнивать по одной записи смысла особого не вижу.
