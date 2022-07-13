```
# SEO
location ~* "^/api/v1/test/(.*)-(.*)\.html" {
    set $forward_header "api.test.com";
    if ( $forward_to_gray != '' ) {
        set $forward_header $host;
        proxy_pass http://$forward_to_gray$request_uri;
        break;
    }

    set $scene $1;
    set $para_uri $2;
    set $args "scene=$scene&uri=$para_uri";
    rewrite "^.*$" "/api/v2/genter-html" break;
    proxy_set_header Host $forward_header;
    proxy_pass http://backend_ngx;
}
```