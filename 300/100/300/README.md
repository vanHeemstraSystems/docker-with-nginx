# 300 - How to remove the path with an nginx proxy_pass

Based on "How to remove the path with an nginx proxy_pass" at https://serverfault.com/questions/562756/how-to-remove-the-path-with-an-nginx-proxy-pass

Based on "Removing start of path from nginx proxy_pass" at https://stackoverflow.com/questions/28130841/removing-start-of-path-from-nginx-proxy-pass

You need to add a trailing slash (/) for **external** containers at the end of the path (/d2iq **/** ) as well as at the end of the ```proxy_pass``` (http://d2iq-management-d2iq-dev:3000 **/** $empty;).

```
server {
  ...
  location /d2iq/ {
    resolver 127.0.0.11 ipv6=off valid=30s;
    set $empty "";
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded_Proto $scheme;
    proxy_pass http://d2iq-management-d2iq-dev:3000/$empty; # End with slash
  }
  ...
}
```

nginx/nginx.conf.development
