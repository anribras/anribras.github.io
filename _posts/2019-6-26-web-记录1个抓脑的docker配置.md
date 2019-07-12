---
layout: post
title:
modified:
categories: Tech
tags: [web]
comments: true
---


dnmp环境，nginx php72仅在容器主机内.

结果laravel guzzlehttp 访问为localhost时，打死不成功.

```php
function api($method = 'POST', $url = '', $params = [])
{
    $client = new Client([
        //docker addr
//        'base_uri' =>  config('app.url'),
        'base_uri' =>  config('app.url'),
        'defaults' => [
            'exceptions' => false
        ]
    ]);

    $headers = ['Accept' => 'application/json', 'Authorization' => 'Bearer '.Session::get('token')];
    if ($method == 'GET') {
        $response = $client->request($method, $url, [
            'query' => $params,
            'headers' => $headers,
            'http_errors' => false,
        ]);
    } else {
        $response = $client->request($method, $url, [
            'form_params' => $params,
            'headers' => $headers,
            'http_errors' => false,
        ]);
    }

    return json_decode($response->getBody(), true);
}
```

排查了好久，发现直接设置上面的`base_url`为nginx容器内ip(172.21.0.x)时，可以的!

经过确认是docker-compose启动nginx时，应该选择network_mode:host
```yml
network_mode: host
#networks:
    #- default
```

为啥这么做呢? 看完下面2个帖子就大概知道了.

<https://laracasts.com/discuss/channels/laravel/guzzlehttp-exception-connectexception-curl-error-7-failed-to-connect-to-localhost-port-8087-connection-refused>
<https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-mach>

