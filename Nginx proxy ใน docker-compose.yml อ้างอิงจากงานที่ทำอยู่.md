## Nginx proxy ใน docker-compose.yml อ้างอิงจากงานที่ทำอยู่
มันเริ่มจากงาน React ตัวแรกของผม ที่อยากจะลอง แยก API กับ Frontend มันออกมา OK ใช้งานได้แต่ ……..

**ปัญหา:** งานนี้เดิมใช้ node เป็นตัว proxy ไปยัง server API แต่กับติดปัญหา error timeout เมื่อ User ส่งข้อมูลที่ขนาดใหญ่กว่าที่เราคิดเอาไว้ (ใหญ่มากๆ …… 1000 เท่าได้) เมื่อเราลองทดลองส่งข้อมูลขนาดนั้นดูบ้าง ก็ได้รับ error timeout ผ่านใน 5 วินาที แต่ในเครื่อง Dev สามารถส่งได้สำเร็จ ข้อแต่ต่างระหว่างเครื่อง Dev กับ Sever คือ proxy

มันทำให้ผมมีความคิดว่า “ต้อง upgrade proxy ไปยัง อะไรที่ดีกว่า” ไม่ว่าความคิดนี้จะถูกหรือจะผิด แต่ผมก็ได้ลงมือสร้าง nginx server และให้มัน proxy ไปยัง API, API-v2 และ Frontend เพื่อทดสอบความคิดของผม

**สมมติฐาน:** node proxy มีปัญหา และ nginx proxy จะทำได้ดีกว่า จากความนิยม

### การแก้ไข:
1. ผมเพิ่ม services ชื่อเว็บ web ไปอีกตัวลงไปใน docker-compose.yml
```yml
web:
    image: bitnami/nginx
    ports:
        - 80:8080
        - 443:443
    volumes:
        - ./nginx/main_vhost.conf:/bitnami/nginx/conf/vhosts/my_vhost.conf
    depends_on:
        - api
        - api-v2
        - frontend
```
2.  และเขียน nginx/main_vhost.conf ให้ proxy ไปยัง API, API-v2 และ Frontend
```conf
server {
  listen 0.0.0.0:8080;

  access_log /opt/bitnami/nginx/logs/yourapp_access.log;
  error_log /opt/bitnami/nginx/logs/yourapp_error.log;
  
  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header HOST $http_host;
    proxy_set_header X-NginX-Proxy true;

    proxy_pass http://frontend:3000;
    proxy_redirect off;
  }

  location /api/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;
    proxy_set_header X-Ssl on;

    proxy_pass http://api:3100;
    proxy_redirect off;
  }

  location /api-v2/ {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;
    proxy_set_header X-Ssl on;

    proxy_pass http://api-v2:8080;
    proxy_redirect off;
  }
}
```

โดย proxy_pass สามารถเรียกได้ผ่านชื่อ service ที่ตั้งไว้ใน docker-compose.yml ได้เลย http://<ชื่อ service>:<port> และไม่จำเป็นต้องเปิด port ของ service นอกจาก service web ให้เข้าถึงได้ผ่าน localhost เราแค่ เชื่อมต่อกับ service web ได้และ nginx จะพาเราไปยัง service อื่นๆ ที่อยู่ network ภายใน docker-compose นั้นๆ

**ผลลัพธ์:** ผมส่งข้อมูล (1000 เท่าจากที่คิดไว้) สำเร็จ

**สรุป:** การใช้การเปลี่ยนมาใช้ nginx proxy แก้ปัญหา error timeout ตรงจุดนี้ทำให้ผม คิดว่าบ้างที node proxy อาจจะมี BUG อยู่ก็ได้

**อ้างอิง:** https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
