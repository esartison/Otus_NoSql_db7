# Домашнее задание Сартисона Евгения N6 #

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

*LIST*

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



*STRING*

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


student:~$ time python3 python_import_hset.py
Imported 120000 products

real	0m3,315s
user	0m2,938s
sys   	0m0,227s









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





student:~$ time python3 python_import_zset.py
Successfully imported 120000 items into the ZSET 'my_zset_collection'.

real	0m1,676s
user	0m1,485s
sys	    0m0,109s




## протестировать скорость сохранения и чтения; ## 


## предоставить отчет. ## 
