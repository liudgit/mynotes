
########### 全局块#################
user www-data;                       #配置用户或者组，默认为nobody nobody。
worker_processes 5;                  #允许生成的进程数，默认为1
pid /run/nginx.pid;                  #指定nginx进程运行文件存放地址
#error_log log/error.log debug;       #制定日志路径，级别。这个设置可以放入全局块，http块，server块，级别以此为：debug|info|notice|warn|error|crit|alert|emerg
########### events块#################
events {
	worker_connections 1024;         #最大连接数，默认为512
    multi_accept on;				 #设置一个进程是否同时接受多个网络连接，默认为off
}
########### http块#################
http {
	include /etc/nginx/mime.types;   #文件扩展名与文件类型映射表
    default_type application/octet-stream;      #默认文件类型，默认为text/plain
    #access_log off; #取消服务日志    
    #log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
    #access_log log/access.log myFormat;  #combined为日志格式的默认值
	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	
	#开启文件传输，一般应用都应设置为on；若是有下载的应用，则可以设置成off来平衡网络I/O和磁盘的I/O来降低系统负载
	sendfile on;
	#告诉nginx在一个数据包里发送所有头文件，而不一个接一个的发送。
	tcp_nopush on;
	#告诉nginx不要缓存数据，而是一段一段的发送--当需要及时发送数据时，就应该给应用设置这个属性，
    #这样发送一小块数据信息时就不能立即得到返回值。
	tcp_nodelay on;
	keepalive_timeout 65;             #长连接超时时间，单位是秒
	types_hash_max_size 2048;
	

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	

	##
	# Gzip Settings
	# gzip模块设置，使用 gzip 压缩可以降低网站带宽消耗，同时提升访问速度。
	##

	gzip on;                           #开启gzip
	gzip_min_length  1k;               #最小压缩大小
	gzip_buffers     4 16k;            #压缩缓冲区
	gzip_http_version 1.0;             #压缩版本
	gzip_comp_level 2;                 #压缩等级
									   #压缩类型
	gzip_types   text/plain text/css text/xml text/javascript application/json application/x-javascript application/xml application/xml+rss;
	gzip_disable "msie6";

	# gzip_vary on;                     匹配压缩类型
	# gzip_proxied any; # gzip_comp_level 6; # gzip_buffers 16 8k; # gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
	#gzip_proxied off;

     #nginx做为反向代理时启用,off(关闭所有代理结果的数据的压缩),expired(启用压缩,如果header头中包括"Expires"头信息),no-cache(启用压缩,header头中包含"Cache-Control:no-cache"),no-store(启用压缩,header头中包含"Cache-Control:no-store"),private(启用压缩,header头中包含"Cache-Control:private"),no_last_modefied(启用压缩,header头中不包含"Last-Modified"),no_etag(启用压缩,如果header头中不包含"Etag"头信息),auth(启用压缩,如果header头中包含"Authorization"头信息)
	#gzip_disable msie6;

	#(IE5.5和IE6 SP1使用msie6参数来禁止gzip压缩 )指定哪些不需要gzip压缩的浏览器(将和User-Agents进行匹配),依赖于PCRE库
	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
########### server块#################
    server {
        listen      7088;
        server_name localhost;
        charset utf-8,gbk;
########### location块#################
        location / {
            #proxy_pass http://localhost:8865;
            proxy_pass http://localhost:8282;

            proxy_redirect          off;
            proxy_set_header        Host $host:$server_port;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size    10240m;     #允许客户端请求的最大单文件字节数
            #client_body_buffer_size 128k;      #缓冲区代理缓冲用户端请求的最大字节数
            proxy_connect_timeout   60;			#nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_send_timeout      60;			#后端服务器数据回传时间(代理发送超时)
            proxy_read_timeout      60;			#连接成功后，后端服务器响应时间(代理接收超时)
            proxy_buffer_size       4k;			#设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffers           4 32k;		#proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
            proxy_busy_buffers_size 64k;		#高负荷下缓冲大小（proxy_buffers*2）
            proxy_temp_file_write_size 64k;		#设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy_ignore_client_abort on;
        }

    }
	# upstream作负载均衡
    upstream myapp { 
      server localhost:8864;
    }  
    server {

        listen      6088 ssl;

        server_name  xxx;
        
        charset utf-8,gbk;
        ssl on;
        #root         /usr/share/nginx/html;
 
        ssl_certificate "/opt/software/https/ssl.crt"; 
        ssl_certificate_key "/opt/software/https/ssl.key";   
		#ssl_client_certificate /opt/software/https/ca.crt;#双向认证
        #ssl_verify_client on; #双向认证
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  5m;
        ssl_protocols SSLv2 SSLv3 TLSv1;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_prefer_server_ciphers on;

        location ~ /(HDP|HDP-UTILS|tools){
        	root /opt/docker/nginx/html;
         	autoindex_exact_size off;
        	autoindex_localtime off;
        	autoindex on;
        }

        location ~ .*\.(htm|gif|jpg|jpg|jpeg|png|ico|rar|css|js|zip|txt|flv|swf|doc|ppt|xls|pdf|url)$ {

            root /opt/images;
            access_log off;
            expires 24h;
        }
         location /ngstatus{
            stub_status on;
            access_log off;
        }
	#反向代理配置
        location / {
            proxy_pass  http://localhost:8864;

            proxy_redirect          off;
            proxy_set_header        Host $host:$server_port;
            proxy_set_header        X-Real-IP $remote_addr;
           # proxy_set_header        X-Forwarded-For $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size    512m;
            #client_body_buffer_size 128k;
            proxy_connect_timeout   200;
            proxy_send_timeout      200;
            proxy_read_timeout      100;
            proxy_buffer_size       4k;
            proxy_buffers           4 32k;
            proxy_busy_buffers_size 64k;
            proxy_temp_file_write_size 64k;
            proxy_ignore_client_abort on;
        }

    }


}


#mail {
#	# See sample authentication script at:
#	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
# 
#	# auth_http localhost/auth.php;
#	# pop3_capabilities "TOP" "USER";
#	# imap_capabilities "IMAP4rev1" "UIDPLUS";
# 
#	server {
#		listen     localhost:110;
#		protocol   pop3;
#		proxy      on;
#	}
# 
#	server {
#		listen     localhost:143;
#		protocol   imap;
#		proxy      on;
#	}
#}
