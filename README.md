# nginx_preloaders
nginx lua module for automatically determining resources which should be added to "Link" http header in order for web browsers to preload them or push them in the initial response ( http/2 server push )

# Don't use this module yet
This is work in progress and does **NOT** work as advertised
* It does not work correctly if it's used on more than 1 vhost (no isolation, so everything is preloaded regardless of the visited URL :eek:)
* I haven't benchmarked it yet, so no idea if it will blow up on production servers
* It's debugging by default, meaning it constantly spits stuff up in vhost's error.log
* I don't know lua and don't know programming, so this is a perfect example of code you don't want anywhere near you

## Usage

1. load the module in nginx's **http** section (nginx.conf)
```
lua_package_path    "/etc/nginx/lua-modules/?.lua;;";
lua_shared_dict     log_dict    1M;
```
2. add the logging snippet in the location which you want to track. I.e. you'll probably only want to track your static resource usage, so best add it to your static resource section if you have one. Otherwise, add it to your location /
```
log_by_lua '
    local preloadheaders = require("preloadheaders")
    local hit_uri = string.gsub(ngx.var.request_uri, "?.*", "")
    preloadheaders.add_hit(ngx.shared.log_dict, hit_uri, hit_uri)
';
```
3. add the rewrite snippet in the location which should have Link header added. I.e. your index file or your location /
```
rewrite_by_lua '
   local content_type = ngx.header.content_type
   ngx.header.content_type = content_type
   ngx.header.Link = ngx.shared.log_dict:get("linkheader")
';
```
And that's it !

## Usage Help Examples
If you don't have a separate static resource section, you'll want your / location to look something like this:
```
location / {
    index index.html /index.html;
    root   /var/www/example.com;

    log_by_lua '
         local preloadheaders = require("preloadheaders")
         local hit_uri = string.gsub(ngx.var.request_uri, "?.*", "")
         preloadheaders.add_hit(ngx.shared.log_dict, hit_uri, hit_uri)
    ';
    rewrite_by_lua '
         local content_type = ngx.header.content_type
         ngx.header.content_type = content_type
         ngx.header.Link = ngx.shared.log_dict:get("linkheader")
    ';
}	
```

In case you do have a static resource section, you can split it up, so that only static resource usage is tracked and Link header added to everything other than static resources !

```
location ~* ^.+.(jpg|jpeg|gif|png|ico|js|css|exe|bin|gz|zip|rar|7z|pdf)$ {
    root   /var/www/example.com;
    log_by_lua '
         local preloadheaders = require("preloadheaders")
         local hit_uri = string.gsub(ngx.var.request_uri, "?.*", "")
         preloadheaders.add_hit(ngx.shared.log_dict, hit_uri, hit_uri)
    ';    
}
location / {
    index index.html /index.html;
    root   /var/www/example.com;
    rewrite_by_lua '
         local content_type = ngx.header.content_type
         ngx.header.content_type = content_type
         ngx.header.Link = ngx.shared.log_dict:get("linkheader")
    ';
}	
```
