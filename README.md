# ELK_Stack_101

Step 

1. Start elasticsearch ขึ้นมา
 docker compose up -d elasticsearch

2.  เข้าตู้ elasticsearch
 docker exec -it elasticsearch /bin/bash

3.  สร้างพิมพ์เขียว เพื่อปั้ม cert สมาชิก ที่ตู้ elasticsearch

cat <<EOF > instances.yml
instances:
  - name: "elasticsearch"
    dns: [ "elasticsearch", "localhost" ]
  - name: "logstash"
    dns: [ "logstash", "localhost" ]
  - name: "kibana"
    dns: [ "kibana", "localhost" ]
  - name: "filebeat"
    dns: [ "filebeat", "localhost" ]
EOF

4. ใบเซอร์แม่พิมพ์ (CA)
bin/elasticsearch-certutil ca --pem --out elastic-ca.zip --pass "" --silent

5. ระเบิดซิปตัวแม่พิมพ์ออกมาใช้งานภายในตู้ก่อน
unzip elastic-ca.zip -d ./ca_meta

6. ปั๊มใบเซอร์ลูกทั้ง 4 ตู้ โดยใช้ตัวแม่ (CA)
bin/elasticsearch-certutil cert --silent --ca-cert ./ca_meta/ca/ca.crt --ca-key ./ca_meta/ca/ca.key --in instances.yml --out all-certs.zip --pem

7. มัดรวมตัวแม่ (ca.crt) เข้าไปอยู่ใน Zip เดียวกันเพื่อความง่ายตอนดึงออก
cd ca_meta && zip -r ../all-certs.zip ca && cd ..

8. ออกจากตู้กลับสู่โลกของ Mac
exit

9. รันคำสั่นใน Terminal (Mac) ทีละบรรทัด
    # 1. ควักไฟล์ซิปข้ามมิติออกมาจากตู้มาวางบน Mac
    docker cp elasticsearch:/usr/share/elasticsearch/all-certs.zip ./certs.zip

    # 2. สั่งทุบตู้ชั่วคราวทิ้งไปได้เลย
    docker compose down --volumes

    # 3. ระเบิดไฟล์ซิปออกมาลงโฟลเดอร์ชั่วคราว
    unzip -o certs.zip -d ./extracted_certs

    # 4. ย้ายของเข้าประจำการตู้ Elasticsearch (รอบนี้มี ca.crt แท้ๆ แล้ว!)
    cp ./extracted_certs/ca/ca.crt ./elasticsearch-config/certs/
    cp ./extracted_certs/elasticsearch/elasticsearch.crt ./elasticsearch-config/certs/
    cp ./extracted_certs/elasticsearch/elasticsearch.key ./elasticsearch-config/certs/

    # 5. ย้ายของเข้าประจำการตู้ Logstash
    cp ./extracted_certs/ca/ca.crt ./logstash-config/certs/
    cp ./extracted_certs/logstash/logstash.crt ./logstash-config/certs/
    cp ./extracted_certs/logstash/logstash.key ./logstash-config/certs/

    # 6. ย้ายของเข้าประจำการตู้ Kibana
    cp ./extracted_certs/ca/ca.crt ./kibana-config/certs/
    cp ./extracted_certs/kibana/kibana.crt ./kibana-config/certs/
    cp ./extracted_certs/kibana/kibana.key ./kibana-config/certs/

    # 7. ย้ายของเข้าประจำการตู้ Filebeat
    cp ./extracted_certs/ca/ca.crt ./filebeat-config/certs/
    cp ./extracted_certs/filebeat/filebeat.crt ./filebeat-config/certs/
    cp ./extracted_certs/filebeat/filebeat.key ./filebeat-config/certs/

    cd logstash-config/certs
    openssl pkcs8 -in logstash.key -topk8 -nocrypt -out logstash.key
    cd ..


gen เสร็จแล้วทำอะไรต่อ ?

10. เอาคอมเมนต์ # ออกใน docker-compose.yml (เปิดด่านตรวจ)

11. รันคำสั่นใน Terminal (Mac)
docker compose down                                             
docker compose up -d


12. กำหนดรหัสผ่าให้กับ kibana_system และ logstash_system
docker compose exec -it elasticsearch curl -X POST "https://localhost:9200/_security/user/kibana_system/_password" \  
  --cacert /usr/share/elasticsearch/config/certs/ca.crt \
  --cert /usr/share/elasticsearch/config/certs/elasticsearch.crt \
  --key /usr/share/elasticsearch/config/certs/elasticsearch.key \
  -u elastic \
  -H "Content-Type: application/json" \
  -d '{"password":"P@ssw0rd"}'

docker compose exec -it elasticsearch curl -X POST "https://localhost:9200/_security/user/logstash_system/_password" \
  --cacert /usr/share/elasticsearch/config/certs/ca.crt \
  --cert /usr/share/elasticsearch/config/certs/elasticsearch.crt \
  --key /usr/share/elasticsearch/config/certs/elasticsearch.key \
  -u elastic \
  -H "Content-Type: application/json" \
  -d '{"password":"P@ssw0rd"}'

13. สร้าง User และกำหนดสิทธิ์ (Role) ให้กับ Logstash เพื่อให้มีสิทธิ์เขียนข้อมูลลง Elasticsearch (logstash_writer) ผ่านหน้า Kibana Dev Tools
    ขั้นตอนที่ 1: สร้าง Role สำหรับเขียนข้อมูล (Logstash Writer Role)
        POST /_security/role/logstash_writer_role
        {
            "cluster": ["manage_index_templates", "monitor"],
            "indices": [
                {
                "names": [ "logstash-*" ],
                "privileges": ["write", "create", "create_index", "manage", "read"]
                }
            ]
        }
    ขั้นตอนที่ 2: สร้าง User และผูกรหัสผ่าน (Logstash Writer User)
        POST /_security/user/logstash_writer
        {
            "password" : "WriterSecurePass123",
            "roles" : [ "logstash_writer_role" ],
            "full_name" : "Logstash Data Writer",
            "email" : "logstash@local.internal"
        }
    ขั้นตอนที่ 3: นำบัญชีชุดนี้ไปอัปเดตในไฟล์คอนฟิกของ Logstash ตรงท่อน output

        output {
            elasticsearch {
                hosts => ["https://elasticsearch:9200"]
                
                # 🎯 เปลี่ยนมาใช้บัญชี Writer ที่เราเพิ่งสร้างผ่าน Dev Tools
                user => "logstash_writer"
                password => "WriterSecurePass123"
                
                ssl_enabled => true
                ssl_verification_mode => "certificate"
                ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
            }
        }

14. สเต็ปสุดท้าย: ไปเปิดสวิตช์ไฟส่องดูข้อมูลบนหน้าจอสวยๆ
ในเมื่อหลังบ้านเชื่อมท่อติดแล้ว ตอนนี้ข้อมูลล็อกกำลังไหลเข้าไปกองใน Elasticsearch รอให้แอดมินเปิดดูครับ ให้แอดมินทำตามขั้นตอนนี้เพื่อดึงข้อมูลมาโชว์บนหน้าเว็บ Kibana ได้เลยครับ:
    1. เปิดบราวเซอร์ไปที่หน้าเว็บ Kibana ของคุณ (https://localhost:5601)
    2. ไปที่เมนูสามขีดมุมซ้ายบน -> เลื่อนลงไปล่างสุดเลือก Management -> Stack Management
    3. ดูที่เมนูด้านซ้าย คลิกที่คำว่า Data Views (เวอร์ชันเก่าอาจจะเขียนว่า Index Patterns)
    4. กดปุ่มสีน้ำเงิน Create data view ที่มุมขวาบน
    5. ตั้งค่าตามนี้เป๊ะๆ ครับ:
        * Name: logstash-*
        * Index pattern: logstash-* (พิมพ์แล้วระบบจะขึ้นไฮไลท์สีเขียวบอกว่าเจออินเดกซ์ของแอดมินทันที)
        * Timestamp field: เลือกเป็น @timestamp จากดรอปดาวน์
    6. กดปุ่ม Save data view to Kibana